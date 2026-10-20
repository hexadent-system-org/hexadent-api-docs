# Purchase List card preview contract

`GET /v1/purchase-lists` keeps its route, query parameters, pagination, `counts`,
sort order, and existing list fields. Each object in the page's `items` array
adds required `previewItems`. The array contains **every non-cancelled** item snapshot in
that Purchase List (excluding cancellations at every stage), even when the client displays only a few rows on a card.
The page still paginates Purchase Lists, not their preview items.

Each preview item contains `id`, `productName`, nullable `containedQuantity`
as a decimal string, nullable `containedUnitName`, `unitName`, and
`plannedQuantity` and `orderedQuantity` as decimal strings. `id` identifies the Purchase List item
row. `plannedQuantity` is the immutable original Requested Qty. `orderedQuantity` is
the actual ordered quantity and supplies card quantity/receipt context. New live
responses include it; old cached/replay responses fall back to `plannedQuantity`. The
contained quantity and contained unit describe the amount within one purchase
unit: for example, `100.0000` + `pcs` + `box` can display as `100 pcs / box`.
`specification` is free text and is not used to derive this line.

The contained quantity and unit name are saved from the Product when the list
is approved. They are either both present or both null. For historical rows
without these snapshots, both are null; the client can omit the content line.
The purchase unit `unitName` still appears. The preview does not include
current inventory stock, and later product or inventory changes do not rewrite
the snapshot.

`previewItems` belongs to the pagination-specific `PurchaseListPageItem` type.
`PurchaseListSummary` remains the shared summary for detail and Dashboard
responses; those endpoints do not acquire a preview field through this change.
Clients needing full receipt, vendor, or price data use the existing Purchase
List detail endpoint.

The [order confirmation extension](purchase-order-confirmation.md) defines
Approved edits and freezes at ordering. Content, product and price snapshots
remain immutable; Requested Qty is not relabeled as ordered. No amount data is
added to previews; full quantity/price/manual-amount data comes from detail.

## Approved cancellation history and zero active items

[Approved item Cancel](purchase-item-cancellation.md) retains the database row,
but the backend excludes cancellations before ordering from `items`, `previewItems`,
`itemCount`, `cancelledItemCount` and receipt counts, even after the list is ordered.
Post-order cancellations remain visible in detail and counted in summary; detail places them last with
Cancelled and Reason. Card previews exclude all cancelled items, including post-order
cancellations. An all-cancelled Ordered/Completed card has an empty preview but keeps
its original total/cancelled counts and actions; preview length is not the total item count. Preview rows do not
add a cancellation field; use summary counts, not previews, for orderability.
When `itemCount == cancelledItemCount`, disable Mark as Ordered but retain the
Approved card and whole-list Cancel. Effective order amount is zero; historical
snapshots remain. Whole-list Cancel restores only active rows to Draft and retains the
parent and all historical rows in the database; its detail uses the same visibility rule. Empty arrays remain supported. Ordered/Completed receipt
progress and post-order cancellation accounting remain unchanged.
