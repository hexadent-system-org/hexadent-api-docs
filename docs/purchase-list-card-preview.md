# Purchase List card preview contract

`GET /v1/purchase-lists` keeps its route, query parameters, pagination, `counts`,
sort order, and existing list fields. Each object in the page's `items` array
adds required `previewItems`. The array contains **every** item snapshot in
that Purchase List, even when the client displays only a few rows on a card.
The page still paginates Purchase Lists, not their preview items.

Each preview item contains `id`, `productName`, nullable `containedQuantity`
as a decimal string, nullable `containedUnitName`, `unitName`, and
`plannedQuantity` as a decimal string. `id` identifies the Purchase List item
row. `plannedQuantity` is the planned quantity saved on that row. The
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
