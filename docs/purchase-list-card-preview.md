# Purchase List card preview contract

`GET /v1/purchase-lists` keeps its route, query parameters, pagination, `counts`,
sort order, and existing list fields. Each object in the page's `items` array
adds required `previewItems`. The array contains **every** item snapshot in
that Purchase List, even when the client displays only a few rows on a card.
The page still paginates Purchase Lists, not their preview items.

Each preview item contains exactly the card data from the Purchase List item
snapshot: `id`, `productName`, nullable `specification`, and
`plannedQuantity` as a decimal string. `id` identifies the Purchase List item
row. `plannedQuantity` is the planned quantity saved on that row. The preview
does not include current inventory stock, and a later product or inventory
change does not rewrite the snapshot.

`previewItems` belongs to the pagination-specific `PurchaseListPageItem` type.
`PurchaseListSummary` remains the shared summary for detail and Dashboard
responses; those endpoints do not acquire a preview field through this change.
Clients needing full receipt, vendor, or price data use the existing Purchase
List detail endpoint.
