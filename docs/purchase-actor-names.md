# Purchase Lists 提交人与业务审批人

本文件与 `openapi.yaml` 共同定义待交付契约，不表示运行时已部署。Draft 保持每个 Organization、每个 Inventory Item 最多一行，不新增人员或申请明细表。

## 业务姓名与账号

| 字段 | 含义 |
| --- | --- |
| Draft `submittedByName` | 用户登记的可选业务提交人，可修改或清空 |
| List item `submittedByName` | 审批时复制的提交人快照，可为 null，之后不可修改 |
| List `selectedApprover` | 必填业务审批人；选择有效成员或手填姓名，姓名保存为审批时快照 |
| List `approvedBy` 与事件 actor | 实际执行请求的登录账号，由服务端认证上下文决定 |

业务姓名只能证明当时登记了谁，不证明该人本人进行了认证操作。手填不改变权限、tenant 或成员身份。`PurchaseActor` 的非空 membershipId 和 role 契约不变；selectedApprover 使用独立 `PurchaseSelectedApprover`，手填时 membershipId/role 显式返回 null，displayName 不能空。

姓名输入拒绝 CR/LF，trim、折叠连续空白并保留展示大小写；手填提交人和审批人最多 100 字符，schema 的 maxLength 也限制原始输入，不能靠 trim 绕过长度上限。提交人空白转 null；审批人空白返回 422。姓名冲突比较在同样空白规范化后仅将英文 A–Z 转小写，不做 NFKC、别名或身份推断；Alice 与 alice 相同，Alice 与 Alicia 不同。

现有 Profile 的 firstName/lastName 各允许 80 字符，成员完整姓名可能超过 100 字符。因此选择现有成员必须完整保存其合法姓名，不能截断或因手填上限拒绝该成员；审批人快照存储容量要覆盖现有成员姓名。100 字符上限用于自由文本输入，不缩小现有成员选择能力。

## Draft 添加、编辑与审批

- 添加请求在每个 item 上携带可选 submittedByName；不传/null/空白为 null。Web 批量输入一次时为本次每个 item 复制该值，不需要新增批次级字段。
- 已存在商品返回 existing/current item，数量和姓名都不覆盖；修改使用带 expectedVersion 的 PATCH。
- PATCH 省略姓名不修改，null 或空白清空；规范化后的持久化值改变时更新真实 updatedBy、updatedAt 和 version。既有 no-op、stale-version 和幂等顺序保持不变。
- Draft 普通单元格为空时留空，不显示“未填写”，不自动预填或记住上一次姓名。
- Approve 校验全部 Draft version，原子复制提交人及审批人姓名快照并移除所选 Draft。之后修改账号姓名、Draft 或清单 name/notes，均不改快照。
- 姓名/审批人选择属于 material request；改变姓名或输入方式不能复用成功请求的幂等 key。同 operation/key/material payload 的重试返回原结果。

Approve 保持现有成员请求有效：

```json
{
  "name": "Weekly order",
  "selectedApproverMembershipId": "33333333-3333-4333-8333-333333333333",
  "draftItems": [{"draftItemId": "44444444-4444-4444-8444-444444444444", "expectedVersion": 3}]
}
```

成员必须为当前组织 active Admin/Manager；姓名取其当时完整显示名。实际 actor 仍必须为 Admin/Manager。另一种输入是：

```json
{
  "name": "Weekly order",
  "selectedApproverName": "Alice",
  "draftItems": [{"draftItemId": "44444444-4444-4444-8444-444444444444", "expectedVersion": 3}]
}
```

对应响应投影如下，完整响应仍包含清单/item 的全部必需字段：

```json
{
  "approvedBy": {"membershipId": "33333333-3333-4333-8333-333333333333", "displayName": "Clinic Account", "role": "admin"},
  "selectedApprover": {"membershipId": null, "displayName": "Alice", "role": null}
}
```

以下是无效审批人输入片段；即使其余请求字段合法，也返回 REQUEST_VALIDATION_ERROR 422，且不创建清单或移除 Draft：

```json
[
  {},
  {"selectedApproverName": "   "},
  {"selectedApproverName": null},
  {"selectedApproverMembershipId": null},
  {"selectedApproverMembershipId": "33333333-3333-4333-8333-333333333333", "selectedApproverName": "Alice"},
  {"selectedApproverMembershipId": "33333333-3333-4333-8333-333333333333", "selectedApproverName": null}
]
```

## 退回 Draft 的冲突

规则覆盖 Ordered item 的可选退回，以及 Approved 整单取消时的必然退回。退回姓名取原 item.submittedByName 快照；只使用现有已加载 item 和加锁 Draft 记录，无额外姓名/成员查询。

| 同商品 Draft | 提交人比较 | 结果 |
| --- | --- | --- |
| 不存在 | 任意，包括 null | 新建并复制姓名 |
| 存在 | 规范化后相同 | 合并数量，保留 Draft 的姓名展示值和其他 metadata |
| 存在 | 双方 null | 合并数量，姓名仍 null |
| 存在 | 不同，或仅一方 null | 409 PURCHASE_DRAFT_SUBMITTER_CONFLICT，整次操作无写入 |

单项取消新建 Draft 仍只恢复未收货数量，vendor/vendorItemNumber/unitPrice/notes/priority 为 null，另复制提交人。整单取消仍恢复全部 plannedQuantity 及既有 metadata 快照，priority 为 null，另复制提交人；保持原有历史数量兼容。合并仍遵循原数量上限，不覆盖 vendor 等 metadata。

同组织检查、锁、版本、数量上限和幂等必须与原事务一致。整单任一项冲突时，整个取消和全部 Draft 写入回滚；不能先取消再报错、跳过冲突行或只恢复部分。即使创建 Draft 时发生并发竞争，也不能绕过姓名检查。成功重放不重新合并；失败没有成功结果可重放，用户修改请求（如 returnToDraft）应使用新 key。

冲突响应示例：

```json
{
  "error": {
    "code": "PURCHASE_DRAFT_SUBMITTER_CONFLICT",
    "message": "This product is already in Draft with a different submitter (Draft: Alice; returning item: Bob). Check Draft and confirm with the people involved before adjusting the quantity.",
    "requestId": "01K-PURCHASE-EXAMPLE",
    "details": {
      "purchaseListId": "55555555-5555-4555-8555-555555555555",
      "purchaseListItemId": "77777777-7777-4777-8777-777777777777",
      "inventoryItemId": "11111111-1111-4111-8111-111111111111",
      "draftItemId": "44444444-4444-4444-8444-444444444444",
      "draftSubmittedByName": "Alice",
      "returningSubmittedByName": "Bob"
    }
  }
}
```

上述 details 字段对新错误均必需，两个姓名可为 null；整单冲突返回首个冲突项即可，客户端不依赖冲突项顺序。只包含当前组织记录，不暴露其他 tenant、SQL 或内部 fingerprint。API 英文 message 中缺姓名可写 Not specified；这不改变普通 Draft 空值留空的规则。客户端按 code 处理，不能解析 message 决定流程；显示姓名时当作纯文本。

冲突后提示去 Draft 检查并先与相关人员沟通，不自动覆盖提交人或调整数量。Ordered item 可由用户重新确认 returnToDraft=false，表示仅取消未收货部分，Draft、已收货和库存不变；系统不能自动改选。Approved 整单取消没有“不退回”选项，本次不新增该能力。

## 展示与导出

业务审批人统一显示 selectedApprover.displayName，包括创建确认/成功提示、清单列表/详情及 Dashboard；实际账号仍作为 approvedBy 保留。商品详情使用提交人快照，null 留空。

仅 Approved 可导出及既有数量/价格规则保持。CSV 保持 BOM、单 header、无 metadata preamble；原七列顺序不变，末尾增加 Submitted By、Approved By，分别为 item 提交人快照与每行重复的业务审批人姓名。PDF 商品表末尾增加 Submitted By，页头 Approved By 使用业务审批人快照。两个新自由文本值均纳入既有 CSV formula-injection 防护及 PDF 字符支持检查；中文姓名可保存，但当前 PDF 字体不支持时仍返回 PDF_EXPORT_UNSUPPORTED_CHARACTERS 422 并可改用 CSV，不在本次引入字体或 PDF 引擎。

## 发布、迁移与验证边界

1. 本 Docs PR 先完成 lint/bundle、示例检查和独立审查；不修改运行时。
2. API/DB 任务在既有三表加姓名/快照，selected_approver_member_id 允许 null，真实 approved_by/approved_at 必填且保持 FK。姓名快照加入不可变保护，Draft 商品唯一性不变。
3. 无历史数据回填需求，但不能加必填快照字段后让 stable main/旧 API 的合法成员审批写入失败。执行 API 时明确兼容扩展迁移、姓名派生或等价方案并验证旧 ORM 写入；成员完整姓名不能被 VARCHAR(100) 截断。手填姓名非空由 API 与最终 DB 约束保证。
4. 现有成员审批请求继续有效；原 actor 结构不放宽。新 API 返回 Draft/item submittedByName（null 也显式返回）。旧 API 响应和历史幂等 payload 缺此字段时，兼容读取按 null；成员审批响应保持 UUID/string role 形状，姓名从新快照读取。新成功 payload 记录快照；重放旧 payload 不冒充已有真实姓名快照，不执行数据回填。
5. Web/Mobile 必须在启用手填前证明 selectedApprover 的 null membershipId/role 不会导致解析、筛选或展示崩溃。API 扩展先于 Web，但不能据此假定旧客户端天然兼容；若有严格解析器，先做独立兼容交付或阻止启用手填并记录依赖，不能静默扩展 Mobile UI 范围。API/Web/旧客户端兼容与实际部署候选由下游记录并由用户确认。
6. Web 消费新 API 后交付用户页面 QA。回滚 Web/API 前验证已有手填快照可读、旧成员请求可写，保留所有已取消状态/姓名/Draft 数量；不能靠清字段或自动重复退回修复。

文档校验覆盖 schema、正反示例及与取消/导出的口径一致性，不证明真实 API、PostgreSQL、PDF 渲染或页面通过。下游必测：提交人新增/修改/清空/重复添加及 stale-version；两种审批人与空/歧义输入；姓名快照与账号改名；退回四种分支、整单原子回滚、部分收货仅取消、并发/幂等/tenant；CSV 姓名注入与 PDF 字体限制；隔离库 migration round-trip 和 stable-main 写入兼容。
