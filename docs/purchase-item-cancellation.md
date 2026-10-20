# Purchase List item Cancel 与 Complete

本契约定义 Approved 商品取消记录保留与 Ordered 剩余收货取消，并替换未收齐仍可完成的旧规则。与 Approved 整单 cancellation 分开；原整单接口和 metadata 恢复规则保持不变，所有回 Draft 路径都进行 [提交人冲突检查](purchase-actor-names.md)，只有单项 Cancel 支持用户确认覆盖。服务路径以 OpenAPI servers 为准（API 当前前缀 `/v1`）。

## Ordered 状态与数量

- 新增 item 响应字段 `isCancelled`、`cancelledAt`、`cancelledBy`、`cancelledQuantity`、`pendingQuantity`、`cancellationReason`，List summary/detail 增加 `canComplete` 和 `cancelledItemCount`。
- 依赖优先发布期间这些响应属性在 schema 中是可选的；新 API 实现必须始终返回。旧响应/历史幂等 payload 缺失时按未取消处理（false/null/0），pendingQuantity 从当前数量推导；canComplete 从 List 状态和全部 items 推导，摘要没有 items 时等待服务端新字段，不凭分页数据判断。
- 不扩展 `receiptStatus` 的四个枚举值，`receiptCounts` 继续按实际收货数量计数；取消项不等同于已收齐。UI 优先显示 Cancelled，同时保留 Received 6/10。
- 未取消项：`cancelledQuantity=0`，`pendingQuantity=max(orderedQuantity-receivedQuantity,0)`。
- 已取消项：`cancelledQuantity=orderedQuantity-receivedQuantity`，`pendingQuantity=0`。原订购和实收数量冻结，不允许新的 Receive 或 Correction（包括 no-op）。不提供恢复。
- List 为 Ordered 且所有 item 已收齐/超收或取消时，`canComplete=true`，其他状态 false。取消不会自动完成 List。
- Draft incomingSummary 的 planned/received/listCount/remaining 仅汇总未取消的活跃项；planned使用订货量，remaining汇总pendingQuantity。取消行的已收货历史在原item/收货记录保留，不重新计入活跃在途摘要。Dashboard 原 receipt counts 保留实际收货口径；金额分析按后续确认金额契约处理，单项取消不推断退款。

## 收货进度排除已取消项

以下收货进度规则只适用于 Ordered/Completed，Approved 取消商品只保留在数据库，不进入当前读取投影，也不显示收货进度。用户确认的新展示口径：取消项不计入收货进度的总 items，但保留订单原本的总项数、商品明细及真实收货记录。`receiptStatus` 与 `receiptCounts` 原有含义不变，不能把 Cancelled 伪装成 received，也不能删掉取消行来缩小 `itemCount`。

- `itemCount`：订单可见商品行数，包括下单后取消行，不包括下单前取消行，商品列表总数继续使用。
- `cancelledItemCount`：当前可见集合中 `cancelled_at` 非空的 item 行数，范围 0 至 itemCount；按 item 标记计算，与整单 Cancelled 状态、取消的 quantity 数量分开。无单项取消的历史行自然计为 0。
- 收货进度有效总项数 = `itemCount - cancelledItemCount`。
- 收齐项数 = `receiptCounts.received + receiptCounts.overReceived`；部分收货仍不计作收齐。取消项不能已收齐/超收，因此原分子无需改写。
- 有效总项数大于 0：进度为收齐项数 / 有效总项数，百分比按最近整数四舍五入，与原页一致；取消项数大于 0 时单独显示，没有取消时不显示取消提示。
- 有效总项数为 0（全部取消）：显示 `No items to receive · 3 items cancelled`（按实际数，单数用 `1 item cancelled`），隐藏百分比和进度条，不显示 0/0，也不把取消描述为 100% received。Complete 仍按原规则允许手动完成。

以 3 项中 2 项收齐、1 项取消为例：

| 位置 | 展示 |
| --- | --- |
| 列表卡片 | `2 of 2 items received`，100% 进度条；下方 `1 item cancelled` |
| 详情收货进度 | `Receiving progress · 100%`，进度条下方 `2 of 2 items received · 1 item cancelled` |
| 订单商品列表 | 仍为 `3 items`；取消行显示 Cancelled 和既有 Reason 入口 |

实时列表、详情、Dashboard 的所有 PurchaseListSummary 投影和 mutation 内嵌 List 统一显式返回真实 `cancelledItemCount`，包括 Approved、Ordered、Completed。聚合沿用组织范围并复用已有查询，无 DB 新字段、迁移、回填或外部服务。取消成功后详情立即更新进度，刷新及返回列表后口径一致。

兼容：OpenAPI 属性为可选且允许 null，便于旧响应及不可变历史幂等快照继续读取；新实时响应必须返回整数，不使用 null/默认 0 掩盖漏映射。未知字段时，完整详情可从全部 items 精确推导；摘要没有完整 items 则刷新最新数据，仍未知时不显示误导进度/取消项数，不能从商品预览或分页推断。历史 Completed 不重开，库存、金额、Draft incomingSummary、receiptCounts 和 canComplete 规则均不改变。API 优先部署确认后再发布使用新字段的 Web。

验收使用既有诊所账户和可创建的 Ordered fixture，覆盖无取消、部分收货后取消、收齐与取消混合、全部取消（含单项）、超收、刷新及 mutation 结果；列表/详情/Dashboard 的计数一致。数据计数和旧快照兼容用 HTTP/隔离 PostgreSQL 自动化，卡片和详情展示由用户页面 QA；Mobile UI、金额报表、取消原因编辑均不扩展。本 Docs 任务无运行时或 DB 验证声明。

## Cancel — 两状态共用入口

`POST /purchase-list-items/{itemId}/cancellation`，必须携带认证和 8–128 字符的 `Idempotency-Key`。Approved 新操作还必须发送 `X-Purchase-Cancellation-Version: 1`（见下方扩展）；Ordered 无此 header 的原请求保持原响应形状。所有现有 active Member 可操作，tenant 来自服务端账户上下文。示例：

```json
{"expectedListVersion":2,"expectedItemVersion":3,"reason":"Vendor cannot supply the remaining quantity.","returnToDraft":true}
```

`reason` 必填且必须为字符串；去除首尾空白后按 Unicode 字符数校验 1–500 字符，两种状态均保存规范化原因及取消人、时间；不新建审计表。缺失、null、空串、纯空白或超过上限返回现有 422 字段校验错误（field 为 reason），没有取消或 Draft 写入。原始输入首尾空白不计入长度。Ordered 取消后不可编辑原因；两分支原因均不复制至 Draft。

`returnToDraft` 可省略，默认 false；拒绝 null、未知字段、非法 UUID、非正数或超出 int64 的版本。取消数量始终由服务端加锁计算，客户端不传数量。Approved 可选恢复全部已保存 orderedQuantity，并保留取消行，List version/updatedAt 及 item version 更新，purchaseListItem 返回完整 item。Ordered 新 key 对已取消项报 `PURCHASE_ITEM_CANCELLED`；已收齐/超收报 `PURCHASE_ITEM_NOT_CANCELLABLE`；非 Approved/Ordered 报 `PURCHASE_LIST_STATE_CONFLICT`。与 Receive/Correction 使用相同的版本和锁顺序，Ordered 只增加 item version，不增加 List lifecycle version；Complete 必须锁定并重新核对所有 item。

返回完整 `purchaseList`、`purchaseListItem`（两种状态均为完整 item；旧成功幂等快照可能为 null）和 `restoredDraftItem`（不回 Draft 时为 null，回 Draft 时含 disposition=created/merged 及完整 draftItem）。响应可直接刷新 List 的 Complete 条件，不需要再次提交。

Ordered 示例数量投影（完整 response 还包含既有详情字段）：

```json
{
  "plannedQuantity":"10.00",
  "receivedQuantity":"6.00",
  "remainingQuantity":"4.00",
  "varianceQuantity":"-4.00",
  "receiptStatus":"partially_received",
  "isCancelled":true,
  "cancellationReason":"Vendor cannot supply the remaining quantity.",
  "cancelledQuantity":"4.00",
  "pendingQuantity":"0.00"
}
```

- 不回 Draft：没有任何 Draft 写入。
- 回 Draft：Approved 增加全部已保存订货量，Ordered 只增加本次剩余数量。已有同商品 Draft 必须先比较提交人，无确认时同名（按姓名规范化比较）或双方为空才合并数量，保留原提交人展示值及 vendor、vendorItemNumber、unitPrice、notes、priority（包括 null）；姓名不同或仅一方为空先返回 PURCHASE_DRAFT_SUBMITTER_CONFLICT，取消和 Draft 均不提交；明确确认后按下方 Draft 版本绑定协议覆盖姓名并合并。新建携带商品、数量及原 item 的 submittedByName 快照，以上采购 metadata 仍全部为空，待用户重新采购时填写。
- 冲突提示包含双方姓名、当前/退回/合并数量与 Draft ID/version，复用已加载 item/Draft，无额外姓名查询。用户可明确继续覆盖，或主动改为 returnToDraft=false 后用新 key 仅取消；不自动降级、不自动确认/改数量/姓名；已收货和库存不受影响。Approved 整单取消仍强制回 Draft，不能用单项的“不回 Draft”方式绕过。
- 当前商品 Draft 唯一性不变。数量合计超过 999999.99 时整次操作失败，取消标记和 Draft 均不提交；其他异常同样全部回滚。
- 同 operation/key/payload 成功重放返回原响应，包含后续 List 已完成的情形，不重复增加数量；改变 payload 复用 key 返回 IDEMPOTENCY_CONFLICT。Receive/Correction 的历史成功请求也可重放，但取消后新 key 不允许变更。
- 两种状态的单项取消均使用 nullable `cancelled_at`、`cancelled_by_member_id` 及 `cancellation_reason`（Text），原因与取消及 Draft 写入同事务提交；历史行不回填。DB 约束非 null 原因只能用于已取消项，并限制为去除首尾空白的 1–500 字符；历史已取消行继续允许 null。迁移 upgrade 保留旧数据，应用回滚保留扩展列及原因，不静默丢弃已保存原因。后续 schema downgrade 如存在非 null 原因必须拒绝，先明确数据保留方案。同组织成员外键与成对非空约束；后续业务姓名字段见上述姓名契约。不存 cancelled_quantity，不新建来源关联；取消本身不改 Inventory，不创建 receipt event/Inventory Activity。
- 成功操作日志在提交后写入，记录组织/操作者、List/item ID、数量、是否回 Draft、Draft disposition/id、requestId 和结果。Approved 和 Ordered 都在现有 item 行保留取消原因/人/时间；操作日志仅作请求关联且不存自由文本原因/商品快照；幂等仅保存规范化请求的 fingerprint 与成功结果。失败日志不能表述为成功；不输出凭证、内部 SQL 或敏感 payload。

## Ordered 取消原因与发布兼容

- item 的实时读响应及 mutation 内嵌 item 统一返回 `cancellationReason: string | null`。未取消或本次新增字段前的已取消行返回 null；旧响应及历史幂等 payload 缺字段时客户端按 null 处理，不虚构历史原因，也不修改旧快照。
- reason 的规范化值纳入幂等 payload。相同 key、相同规范化 reason 复用原结果；只改变首尾空白不改变语义；同 key 改变原因返回 IDEMPOTENCY_CONFLICT。未知结果重试必须冻结完整原请求、原因及 key。
- 新接口校验必填 reason，旧客户端无 reason 的新请求返回 422。升级前已成功的无 reason 请求仅允许以原 key、原 payload 重放原结果；新 key 不能绕过必填。实现需在受认证、租户及 operation 约束的历史成功重放路径识别这些旧请求；没有历史成功记录仍返回 422，不进行新写入。
- Web 原因输入为必填自由文本，在 Cancelled 状态旁提供可聚焦/可查看的提示或详情，刷新后仍可查看；历史 null 不显示虚构原因。操作为 Cancel，状态为 Cancelled，不新增原因编辑或分类接口。
- Ordered 弹窗使用纯说明："After cancellation, this item and its receiving history will be kept. It will no longer block completion, and you won’t be able to receive or correct its quantity afterward." 不插入当前实收数量，不使用 Stop waiting for。Receive/Cancel 水平排列，Receive 保留边框；Cancel hover 仅下划线。
- 新增必填字段属于请求兼容性变更。顺序为 Docs → API → Web，但 API/Web 应协调上线，避免旧 Web 继续提交缺 reason 的新请求；部署窗口保留完整错误提示，不能静默填充占位原因。未提供取消 UI 的 Mobile 读取旧状态枚举不变。若不能协调，则需另行批准 API 暂时兼容缺省 reason 的 expand-migrate-contract 方案，不能在实现中静默放宽必填规则。
- 本次不新增外部服务或日志系统；取消原因由持久化 item 审计可查，沿用现有操作日志的组织/操作者/List/item/requestId 关联和脱敏规则，不无条件输出用户自由文本。

## Complete 与客户端兼容

Complete 请求使用 `expectedListVersion` 和可选 `expectedItemVersions`（当前 GET 详情可见 items 的完整 UUID/version map，不包含下单前取消行）；不再使用旧文档中的 confirmVariance。它在当前 API 也已不存在，不能作为短收绕过入口。

若有未取消且 received < orderedQuantity 的项，返回 409 PURCHASE_LIST_ITEMS_PENDING，details 为 pendingItemCount 与 purchaseListItemIds。新客户端禁用 Complete 并提示处理待收货项；旧客户端仍能正常读取原枚举，并按通用错误处理显示后端英文 message。新 Cancel UI 必须等新 API 部署确认后才启用。

Complete 只改 List 状态并写原有 audit，不更改库存。成功结果包含完整详情、alreadyCompleted 和 nullable message；已 Completed 的重复调用维持原有 alreadyCompleted=true 行为。历史 Completed 短收单不自动重开。新增错误不要求旧客户端解析 details，也不能导致崩溃。

## 验证与发布顺序

1. Docs lint/bundle 与独立 High/Full 契约审查，独立 PR 集成并记录校验证据。
2. API 实现、迁移及隔离 PostgreSQL HTTP/并发/回滚/tenant 测试，用户确认 CI/部署版本。
3. Web 使用新字段并交付用户页面 QA；Mobile UI 变更不在本任务范围。

新增原因必测：缺失/null/纯空白/trim 后 500 与 501 字符、保存规范化值及读回、历史 null/缺字段、相同 key 改原因冲突及旧成功无原因请求重放、取消/回 Draft 失败不保存原因、迁移保护已有原因。

必测边界：10 收 6 取消 4、零收货取消、全收齐/超收不能取消、默认不回 Draft、已有 Draft 合并保留 vendor、合并超限全回滚、取消后 Receive/Correction/no-op 拒绝、取消/收货/Complete 竞争、成功重放、不同 payload 复用 key、跨组织 ID、旧 Completed 与历史幂等 payload。文档任务无 DB 或部署操作，不将下游测试当作本任务已完成证据。

## 2026-10-18 订货确认扩展

[订货确认契约](purchase-order-confirmation.md)新增独立orderedQuantity，旧plannedQuantity保留原需求。实时收货状态、Complete、pending/cancelled quantity及所有回Draft数量以orderedQuantity为准；旧快照缺新字段时回退plannedQuantity。原整单取消的metadata/姓名冲突规则保持，单项Cancel的姓名确认覆盖按下方扩展。新confirmation能力header保护所有改变订货量或人工金额后的相关取消/收货操作，成功旧幂等重放仍返回原快照。

Requested4、Ordered5、Received2，取消并回Draft仅增加3；Ordered 没有自动退款，Actual Total不减少；Approved 取消商品则从有效采购金额排除，原金额快照保留。历史Received与库存不被修改，取消后Receive/Correction仍拒绝。

## Approved 商品取消记录保留、空清单与显式冲突确认（2026-10-18）

### Approved Cancel

沿用上方单项 cancellation 路径、现有 active Member 权限、必填原因（trim后1–500）和可选回 Draft（默认 false）。Approved 必须无 receipt history；异常历史状态返回 `PURCHASE_LIST_RECEIPTS_PREVENT_CANCELLATION`，不删除历史或已取消行。锁 List 再 item，检查 `expectedListVersion`/`expectedItemVersion`，与 confirmation Save、Mark as Ordered、整单取消串行。

成功保留商品行并设置 `cancelled_at`、`cancelled_by_member_id`、`cancellation_reason`，item version 加1。**不物理删除，不新增状态字段、cancelled_from_status 或独立商品取消审计表**。Requested/Ordered/价格/金额快照保持不变；取消后禁止编辑（含 no-op）、再次取消、Receive 和 Correction。Approved List version 加1并更新 updatedAt，其他 item versions 不变。`purchaseListItem` 返回完整取消行；旧成功幂等 payload 可为 null，保持原样不改写。


退回数量为全部已保存 `orderedQuantity`：Requested=4、Ordered=5，退5；不依据原 Requested 或未保存输入。新 Draft 的 submittedByName 复制 item，其他采购 metadata 沿用单项 Cancel 的 null 规则，不复制人工金额/原因；已有 Draft 数量相加且保留 vendor 等 metadata。新建或合并超999999.99整次失败，沿用单项数量上限；整单历史恢复例外不扩展到单项。重新建单走正常审批，不能保留原单的金额覆盖或绕过审批。

后端统一从清单 `items`、`previewItems`、`itemCount`、`cancelledItemCount` 和 `receiptCounts` 排除下单前取消行，Dashboard 同步此口径；下单后取消行仍计入详情可见集合及取消数；但卡片 previewItems 额外排除所有阶段的取消项，不改变 summary 计数，全部取消时预览为空。单项取消响应的独立 `purchaseListItem` 仍返回完整取消记录，即使它不再出现在内嵌清单中。有效项为未取消行，仅有效项参与 Approved 下单、整单回 Draft、CSV/PDF 导出和在途汇总。已取消项不参与 In active list 和 Approved 库存删除阻拦。整单取消不重复退回已取消项。

未新增取消阶段字段：`cancelled_at` 非空且 `ordered_at` 为空或 `cancelled_at < ordered_at` 表示下单前取消。即使父清单后来 Ordered/Completed，仍从当前读取集合隐藏这些行，并排除实际采购支出和参考价格，以及库存删除阻拦；Ordered 阶段取消保留既有金额和库存阻拦规则，不推断退款。原始数量/金额快照不会改写为0。

按钮统一为 **Cancel**，确认按钮/原因输入/可选回 Draft 与 Ordered 一致；Approved 说明明确从当前清单移除展示，但取消记录与原因保留。前端直接使用后端可见集合，不重复计算取消阶段；详情无分页，下单后取消行稳定排列在有效行之后，显示 Cancelled 和 Reason，不显示编辑/取消按钮。计数仅包含可见集合。存在 dirty/saving/failed 输入时，先保存或明确 Discard changes，再按已保存版本取消；Discard changes 只清理前端输入。提交人冲突弹框显示 Current Draft submitter 和 Submitter on this item。并发取消导致商品从当前详情消失后，另行显示未保存编辑的恢复提示和明确 Discard changes 入口；若此前 Save 结果未知，先以冻结的原请求/key Retry Save 确认结果，再允许丢弃，不能隐藏恢复入口或永久锁住其他操作。


### Draft 提交人冲突确认

仅 Approved/Ordered **单项** Cancel 支持 `draftReturnConfirmation`；只在 `returnToDraft=true` 时可提供，否则422。必需字段是 `draftItemId` 与正整数 `expectedVersion`，不允许 null/未知字段。首次无确认请求遇姓名冲突返回409、无任何写入；用户看过双方姓名和数量后明确继续，使用新 key 提交：

```json
{
  "expectedListVersion": 2,
  "expectedItemVersion": 3,
  "reason": "Vendor cannot supply this item.",
  "returnToDraft": true,
  "draftReturnConfirmation": {
    "draftItemId": "44444444-4444-4444-8444-444444444444",
    "expectedVersion": 4
  }
}
```

服务端仍检查 List/item版本，并在同商品Draft锁内比较确认ID/version；不能用一个布尔值跳过检查。目标换行、删除、version改变（即使姓名现在相同）均409 `PURCHASE_DRAFT_VERSION_CONFLICT`，无写入，刷新后重新显示当前合并/覆盖影响并由用户确认。目标已删除时 details.draftItemId 为确认的ID、currentVersion=null；不能自动新建或覆盖另一行。未提供确认且Draft不存在时允许正常新建，并发唯一性竞争后必须重新读取并按相同规则检查。

有效确认把合并后**整行** submittedByName 设置为退回 item 快照（包括 null），不只影响新增数量；其他 metadata 不改。无冲突/双方null且无确认时保留 Draft 姓名展示值，不额外弹窗。Draft新建version沿用现行规则，合并数量/覆盖姓名作为一次修改，version加1并更新updatedBy/updatedAt。超限、权限、状态、版本、任一写入/依赖失败，取消标记/数量/姓名/幂等成功结果一并回滚。

关闭确认不提交；用户可以重新选择不回Draft，仅取消且不改姓名。Approved 整单取消不接受此确认字段，仍遇姓名冲突全回滚。新确认请求与原请求payload不同必须用新 key；未知结果重试冻结原请求/key。服务端可接受首次携带有效确认的请求，显式用户确认由UI保证；不能自动生成确认绕过弹窗。

### 空 Approved 与统一整单 Cancel（2026-10-20 修正）

最后一个有效商品取消后，List 保持 Approved，不自动 Cancelled、不自动删除。数据库历史行仍在；详情 items/previewItems 为空、itemCount/cancelledItemCount/receiptCounts 均为0、`canComplete=false`，有效金额为已知0。不允许创建空审批单或将无有效项的 Approved 下单。历史版本产生的真正空 Approved 仍支持读取。仍列入 Approved 和 Dashboard 清单计数，直到用户手动整单取消。导出只含有效项；无有效项时 CSV 仅BOM/header，PDF保留页头和空表、项数/合计0。

Mark as Ordered 对无有效项清单返回409 `PURCHASE_LIST_EMPTY`，无写入。UI 禁用下单，沿用整单 **Cancel** 按钮，不新增 Delete 或整单 DELETE 接口。

整单统一调用 `POST /v1/purchase-lists/{purchaseListId}/cancellation`，保留 active Member 权限、`Idempotency-Key`、`expectedListVersion`、可选原因和原提交人冲突规则。锁 List、校验 Approved/版本/无 receipt history 后，只将有效项恢复至 Draft，再将 List 转为 Cancelled、version 加1并记录既有生命周期事件。没有有效项时跳过 Draft 恢复，`restoredDraftItems=[]`；数据库保留全部历史行，不物理删除；返回详情按相同规则隐藏下单前取消行。


清单与审批历史保留，父清单外键及物理删除保护保持。不再出现在 Approved/当前 MVP 列表或 Dashboard；已知 ID 的 GET 仍返回 Cancelled 审计详情，沿用既有 Cancelled 可见性，不新增历史列表页。同 operation/key/payload 重放原200，改变 payload 复用 key 返回 `IDEMPOTENCY_CONFLICT`；新 key 使用旧 version 返回版本冲突，使用当前 version 返回状态冲突，不再次写入。跨组织/未知 ID 仍404；失败不产生部分状态或历史记录。不新增恢复接口。

### 幂等、能力与回滚

先认证/确认当前组织和active membership，再按组织/operation/key及原资源ID/fingerprint查历史成功结果，最后才查当前资源/状态/版本/能力。对无历史记录的不存在或跨租户ID统一404，不可凭ID跨组织取幂等结果。Approved取消、即使后续清单已整单取消仍可重放原 Cancel 成功结果；不能依赖当前item获取List/组织后才寻找成功结果。重放只返回原快照，不复活资源，Web成功后刷新当前集合；404仍不得自动重建；旧历史删行结果按兼容协议处理。改变confirmation/版本/退回选择/规范化原因复用key返回IDEMPOTENCY_CONFLICT。

新客户端发送 `X-Purchase-Cancellation-Version: 1`。Approved单项Cancel必须提供；Ordered新确认覆盖也必须提供，旧Ordered无确认仍允许缺省。提供不支持值或必需时缺失均409 `PURCHASE_CANCELLATION_CLIENT_REQUIRED`，details.requiredCancellationVersion=1，无副作用。与订货确认header独立：订货量/金额变化的操作仍需 `X-Purchase-Confirmation-Version: 1`。header不授予权限；CORS沿用既有机制允许两者。OpenAPI header是条件必需，允许字符串以业务409处理不支持值。成功历史重放优先于当前能力检查。

交付顺序为 Docs、API 迁移与应用、Web；本轮用户授权仅本地并行开发，未授权 staging 迁移、推送或 PR。旧成功幂等响应可为 null，Web 应成功后刷新当前详情；旧空清单兼容继续保留。新 API 使用原取消能力 header，旧 Ordered 无确认请求保持兼容。

迁移仅替换数据库保护函数，无新表/列，无历史回填；恢复 Approved 行物理删除保护，允许严格 Approved、零收货且无收货历史的取消标记。不放宽其他状态/快照保护或审批历史外键。存在保留的下单前取消记录时 schema downgrade 必须拒绝，不能丢弃历史；无该数据时可升降级。已部署的物理删除版 API 与新约束在 Approved Cancel 路径不兼容，因此迁移和新 API 必须协调发布；不能在迁移后回滚到物理删除 writer。原成功 payload、Draft 增量和姓名均保留，不重复重放恢复。


### 下游验收例

| 输入/操作 | 预期 |
| --- | --- |
| Approved需求4订5，Draft原2，选择退回 | Draft=7，其余项保留；原行保留 Cancelled、原因及取消人/时间 |
| Ordered需求4订5收2，Draft原2 | Draft=5，原取消行/原因/收货历史保留 |
| Draft Alice2，返回Bob5，无确认 | 409且零写入；提示合并7、整行姓名将变Bob |
| 同一快照确认继续 | Draft=7且姓名Bob；刷新持久化 |
| 退出确认 / 不回Draft | 零写入 / 仅取消，不改Draft |
| 确认后Draft改名、改量、被删重建 | 409重新确认；不覆盖、不新建降级 |
| null返回姓名覆盖Alice，明确确认 | 数量相加，整行提交人清空为null |
| 最后一个有效项取消后下单 | 保留Approved及取消历史；409 PURCHASE_LIST_EMPTY |
| 手动Cancel无有效项Approved，同key重试 | 原200；Draft不变；清单及历史保留，GET仍可读，Approved列表不再出现 |
| 单项取消后再整单取消 | 只恢复剩余商品，无重复数量 |
| Approved取消/整单取消后重放成功Cancel | 原成功结果，无重复Draft、无行复活 |
| 合并超限或写入失败 | List/item/Draft全部回滚 |

以上为契约验收计划，HTTP/PostgreSQL/浏览器与部署证据由下游提供；Docs只验证schema、样例、一致性和独立审查。
