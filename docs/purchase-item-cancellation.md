# Purchase List item Cancel 与 Complete

本契约替换未收齐仍可完成的旧规则。与 Approved 整单 cancellation 分开；原整单接口和 metadata 恢复规则保持不变，但所有回 Draft 路径都增加 [提交人冲突检查](purchase-actor-names.md)。服务路径以 OpenAPI servers 为准（API 当前前缀 `/v1`）。

## 状态与数量

- 新增 item 响应字段 `isCancelled`、`cancelledAt`、`cancelledBy`、`cancelledQuantity`、`pendingQuantity`，List summary/detail 增加 `canComplete`。
- 依赖优先发布期间这些响应属性在 schema 中是可选的；新 API 实现必须始终返回。旧响应/历史幂等 payload 缺失时按未取消处理（false/null/0），pendingQuantity 从当前数量推导；canComplete 从 List 状态和全部 items 推导，摘要没有 items 时等待服务端新字段，不凭分页数据判断。
- 不扩展 `receiptStatus` 的四个枚举值，`receiptCounts` 继续按实际收货数量计数；取消项不等同于已收齐。UI 优先显示 Cancelled，同时保留 Received 6/10。
- 未取消项：`cancelledQuantity=0`，`pendingQuantity=max(plannedQuantity-receivedQuantity,0)`。
- 已取消项：`cancelledQuantity=plannedQuantity-receivedQuantity`，`pendingQuantity=0`。原订购和实收数量冻结，不允许新的 Receive 或 Correction（包括 no-op）。不提供恢复。
- List 为 Ordered 且所有 item 已收齐/超收或取消时，`canComplete=true`，其他状态 false。取消不会自动完成 List。
- Draft incomingSummary 的 planned/received/listCount 保留当前活跃单据历史口径；remainingQuantity 汇总 pendingQuantity。不可为排除取消部分而删除该行的已收货历史。Dashboard 原 receipt counts 保留实际收货口径；金额分析不在本次变更中改写。

## Cancel

`POST /purchase-list-items/{itemId}/cancellation`，必须携带认证和 8–128 字符的 `Idempotency-Key`。所有现有 active Member 可操作，tenant 来自服务端账户上下文。示例：

```json
{"expectedListVersion":2,"expectedItemVersion":3,"returnToDraft":true}
```

`returnToDraft` 可省略，默认 false；拒绝 null、未知字段、非法 UUID、非正数或超出 int64 的版本。取消数量始终由服务端加锁计算，客户端不传数量。新 key 对已取消项报 `PURCHASE_ITEM_CANCELLED`；已收齐/超收报 `PURCHASE_ITEM_NOT_CANCELLABLE`；非 Ordered 报 `PURCHASE_LIST_STATE_CONFLICT`。与 Receive/Correction 使用相同的版本和锁顺序，只增加 item version，不增加 List lifecycle version；Complete 必须锁定并重新核对所有 item。

返回完整 `purchaseList`、`purchaseListItem` 和 `restoredDraftItem`（不回 Draft 时为 null，回 Draft 时含 disposition=created/merged 及完整 draftItem）。响应可直接刷新 List 的 Complete 条件，不需要再次提交。

示例数量投影（完整 response 还包含既有详情字段）：

```json
{
  "plannedQuantity":"10.00",
  "receivedQuantity":"6.00",
  "remainingQuantity":"4.00",
  "varianceQuantity":"-4.00",
  "receiptStatus":"partially_received",
  "isCancelled":true,
  "cancelledQuantity":"4.00",
  "pendingQuantity":"0.00"
}
```

- 不回 Draft：没有任何 Draft 写入。
- 回 Draft：只增加本次剩余数量。已有同商品 Draft 必须先比较提交人，同名（按姓名规范化比较）或双方为空才合并数量，保留原提交人展示值及 vendor、vendorItemNumber、unitPrice、notes、priority（包括 null）；姓名不同或仅一方为空返回 PURCHASE_DRAFT_SUBMITTER_CONFLICT，取消和 Draft 均不提交。新建携带商品、数量及原 item 的 submittedByName 快照，以上采购 metadata 仍全部为空，待用户重新采购时填写。
- 冲突提示包含双方姓名和 Draft 检查指引，复用已加载 item/Draft，无额外查询。用户可主动改为 returnToDraft=false 后用新 key 仅取消，不自动降级、不自动改数量/姓名；已收货和库存不受影响。Approved 整单取消仍强制回 Draft，不能用单项的“不回 Draft”方式绕过。
- 当前商品 Draft 唯一性不变。数量合计超过 999999.99 时整次操作失败，取消标记和 Draft 均不提交；其他异常同样全部回滚。
- 同 operation/key/payload 成功重放返回原响应，包含后续 List 已完成的情形，不重复增加数量；改变 payload 复用 key 返回 IDEMPOTENCY_CONFLICT。Receive/Correction 的历史成功请求也可重放，但取消后新 key 不允许变更。
- 单项取消本身使用 nullable `cancelled_at`、`cancelled_by_member_id`，同组织成员外键与成对非空约束；后续业务姓名字段见上述姓名契约。不存 cancelled_quantity，不新建来源关联；取消本身不改 Inventory，不创建 receipt event/Inventory Activity。
- 成功日志在提交后写入，记录组织/操作者、List/item、取消数量、是否回 Draft、Draft disposition/id、requestId 和结果。失败日志不能表述为成功；不输出凭证、内部 SQL 或敏感 payload。

## Complete 与客户端兼容

Complete 请求使用 `expectedListVersion` 和可选 `expectedItemVersions`（完整 item UUID/version map）；不再使用旧文档中的 confirmVariance。它在当前 API 也已不存在，不能作为短收绕过入口。

若有未取消且 received < planned 的项，返回 409 PURCHASE_LIST_ITEMS_PENDING，details 为 pendingItemCount 与 purchaseListItemIds。新客户端禁用 Complete 并提示处理待收货项；旧客户端仍能正常读取原枚举，并按通用错误处理显示后端英文 message。新 Cancel UI 必须等新 API 部署确认后才启用。

Complete 只改 List 状态并写原有 audit，不更改库存。成功结果包含完整详情、alreadyCompleted 和 nullable message；已 Completed 的重复调用维持原有 alreadyCompleted=true 行为。历史 Completed 短收单不自动重开。新增错误不要求旧客户端解析 details，也不能导致崩溃。

## 验证与发布顺序

1. Docs lint/bundle 与独立 High/Full 契约审查，独立 PR 集成并记录校验证据。
2. API 实现、迁移及隔离 PostgreSQL HTTP/并发/回滚/tenant 测试，用户确认 CI/部署版本。
3. Web 使用新字段并交付用户页面 QA；Mobile UI 变更不在本任务范围。

必测边界：10 收 6 取消 4、零收货取消、全收齐/超收不能取消、默认不回 Draft、已有 Draft 合并保留 vendor、合并超限全回滚、取消后 Receive/Correction/no-op 拒绝、取消/收货/Complete 竞争、成功重放、不同 payload 复用 key、跨组织 ID、旧 Completed 与历史幂等 payload。文档任务无 DB 或部署操作，不将下游测试当作本任务已完成证据。
