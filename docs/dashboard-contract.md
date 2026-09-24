# Dashboard contract

This is the target contract for the Web Dashboard, approved on 2026-09-23, with the Purchase Spend amendment approved on 2026-09-24.
It does not claim that these endpoints are implemented or deployed. The source
requirements and delivery status are in the [Dashboard Docs task](https://www.notion.so/3df41cca7e9c811996ffe98a8071d388).

## Module boundaries

| GET path, relative to `/v1` | Contents | Query |
| --- | --- | --- |
| `/dashboard/inventory-summary` | Four inventory counts and four attention lists | None |
| `/dashboard/purchase-list-overview` | Draft row count, Approved/Ordered counts, this week's Completed count, three lists | `limit`: integer 1–5, default 3, applied separately to each list |
| `/dashboard/top-used-products` | Five most-used inventory records and comparison with the previous 30 days | None |
| `/dashboard/purchase-spend` | Monthly ordered-supply value | `period`: `last_6_months` (default), `last_12_months`, `last_24_months` |

Clients can request modules in parallel and retry a failed module independently.
Counts and lists in one response must use the same database snapshot and `asOf`.
There is no atomic snapshot across the four requests. Each response supplies the
organization's IANA `timezone` and an offset-bearing `asOf` in that timezone.
All reads, including historical activity and joins, are scoped to the authenticated
organization; a client cannot select another organization through these endpoints.
Existing active-member read permissions apply. Use the existing `ErrorEnvelope`:
401/403 for access failures, 422 for invalid `limit`/`period`, and 503
`SERVICE_UNAVAILABLE` for unavailable reads. A failed query is not an empty result.

The old combined `GET /dashboard` and quantity-based
`GET /dashboard/usage-trend` definitions and their unused aggregate schemas are
replaced, not silently reinterpreted at the same paths. The user confirmed that
App Dashboard development has not started, and the current Web Dashboard has no
calls to these old endpoints. No legacy adapter is required for this planned
contract. The old inventory category-breakdown payload has no corresponding card
in the current Web Dashboard and is not part of these four modules. Analytics
paths and their schemas remain unchanged.

## Inventory and navigation

Count inventory records, not units, and exclude soft-deleted records. Low Stock
means `0 < quantityOnHand <= lowStockThreshold`; Out of Stock means quantity is
zero. Expiring Soon uses the clinic's local **date**, includes today and day 30,
and is independent of quantity. All is a union by inventory ID; overlap counts
once in All and once in each matching individual category. Count the whole set,
then sort and truncate each list independently to five records.

All orders Low Stock before Out of Stock before Expiring Soon-only; within that
priority, order expiry date ascending (null last), name ascending, then UUID
ascending. Each individual list orders expiry date (null last), name, UUID.
Name comparison uses Unicode code-point order, not a locale-dependent database
default. An expired low/out-of-stock item still appears by its stock condition;
the `expired` data value does not introduce a Dashboard Expired filter or badge.

Total Items is informational. The other three cards' View All links select the
matching Inventory filter. View Inventory under All opens unfiltered Inventory;
under another tab it opens that filter. Attention rows are not clickable. URL
routing/deep-link behavior remains with its existing Web task.

## Purchase overview

Return full counts independently of `limit`; the lists contain
`min(count, limit)` entries each. Draft counts rows, not ordered quantities.
Completed means the list is currently completed and `completedAt` falls between
the local Monday at 00:00 and `asOf`, inclusive. Lists in all three groups sort by
creation time descending, then ID descending, including the Completed group.
Do not prioritize ready-to-complete lists or use update time for ordering.

The initial Web selection is Ordered. Clicking Approved/Ordered/Completed selects
the already-returned group; View Draft navigates without changing that selection.
View All opens the corresponding Purchase Lists status; Completed opens all
Completed lists, without requiring a new weekly filter. A list card opens its
detail. Ordered progress uses the existing `receiptCounts` interpretation, not
permission to complete. There is no Ready to complete metric and no change to
the existing forced-completion flow.

## Top Used

The source is recorded Activity with `reasonCode=product_used` and a negative
`quantityDelta`, grouped by `inventoryItemId`; sum `-quantityDelta`. A Set action
with this reason and a negative delta also represents actual recorded use.
Positive or zero deltas do not count. Data corrections, physical counts, damage,
expiry, receiving, and other reasons neither count nor reverse earlier usage.
This measures **recorded use**, so fixing stock with a generic correction does
not correct a previous usage entry.

For local date D, the current window starts at local midnight D−29 and ends at
`asOf`, inclusive. The prior window starts at local midnight D−59 and ends just
before local midnight D−29. This is 30 calendar dates per window, not 720 elapsed
hours; the current day is partial. Use activity occurrence time, not creation
time. Do not filter out removed inventory. Rank positive current totals, comparing
against the full prior total for the same ID, even if it was not in the prior top
five. Use Decimal arithmetic and half-up rounding for the two-decimal percentage;
prior zero returns null, rendered as an em dash.

Name and unit come from the latest qualifying current-window Activity snapshots
(occurrence time then activity ID descending). Category comes from the retained
Product's current category, including retained soft-deleted records: there is no
historical category snapshot in Activity today. This choice does not require a
new history backfill. Units are not normalized across products. All rows are
display-only, including still-available inventory.

## Purchase Spend

The 2026-09-24 approved amendment replaces received-value accounting with ordered
value. Keep the card title **Purchase Spend** and use the subtitle **Estimated
value of orders placed**. The endpoint, period values, and response shape remain
unchanged; this is the target for the not-yet-implemented API and Web module.

Return one total amount per month, not consumption quantities or per-category
series. Include the current incomplete month and exactly 6, 12, or 24 continuous
months in ascending order. Use Purchase List item `plannedQuantity` times its
`unitPrice` snapshot, assigned to the list's `orderedAt` month in the clinic
timezone. Include lists currently Ordered or Completed with `orderedAt` between
local midnight on `fromDate` and `asOf`, inclusive. Draft, Approved, and Cancelled
lists do not contribute. Join all records within the authenticated organization.

The current lifecycle prohibits cancellation and changes to ordered quantities
or price snapshots after ordering. Receiving (including partial or cross-month
receiving), receipt corrections, and completion (including forced completion)
neither change the original ordered amount nor add another amount. Retain ordered
snapshots after inventory removal; do not join/filter on current inventory
availability or use current Inventory prices. This measures order placement,
not eventual fulfilled value, consumption, invoices, refunds, or payments.

Compute precise Decimal products, sum by month, then round half-up to two decimal
places. Missing price is excluded; an explicit zero price is valid. An empty month,
including a month with only missing-price ordered items, is `0.00`. This is not
proof that all orders were free or that no orders were placed.

Follow the existing single-currency price convention: return the current
Organization `currencyCode`. Existing purchase prices have no per-order currency
snapshot; changing regional currency does not convert historical numeric values.
This contract does not introduce FX or historical currency accounting. The Web
always displays this qualification:

> Estimated from ordered quantities and purchase prices. Items without a price are excluded. This does not represent payments.

### Amendment and implementation dependencies

The user approved ordered-month / ordered-quantity accounting in the Codex
Purchase Spend task on 2026-09-24, retained the card title, and confirmed that
ordered lists cannot currently be cancelled or their ordered amounts edited.
They then requested contract update, Notion task update, and API execution in that
order. This supersedes the 2026-09-23 received-value and original-receipt-month
correction policy. Receipt attribution and historical correction migration are
no longer prerequisites for this metric; the receipt workflow itself is unchanged.

Implementation uses existing Purchase List `orderedAt` and item quantity/price
snapshots. Verify the existing schema and tenant-scoped query behavior; do not
introduce receipt links, migrations, or provisioning under this contract change.
Docs, API, and the existing Web integration task are independently delivered,
in that compatibility order. The API remains read-only; the Web changes only its
source data and the approved subtitle/qualification in its own task.

## Representative acceptance cases

These are contract fixtures for subsequent API tests, not claims of runtime QA.
The OpenAPI response examples are schema-validated with the repository checks.

| Case | Expected result |
| --- | --- |
| Local D=2026-09-23; expiry D, D+30, D+31 | First two are expiring soon; day 31 is not. |
| Four active records: low+expired, low+expires today, zero+expires day 30, in-stock+expires day 31 | Summary 4/2/1/2; All count 3, not 5; expired low-stock remains in All. See inventory `boundaries` example. |
| Eight low-stock matches | Full low count 8; each applicable list has at most 5. All must be independently selected from the complete union. |
| Empty inventory | 200, zero counts and empty attention arrays. |
| `limit=1`; Approved count 2, Ordered count 4 | Counts remain 2/4; one entry from each group. `limit=0`, 6, or a fractional value is 422. |
| August-created list completed Monday 2026-09-21 00:00 local | Included this week. Completion just before Monday is excluded. Both eligibility and counts use completion time; ordering uses creation time. |
| September 23 Top Used | Current Aug 25–Sep 23 (to asOf), previous Jul 26–Aug 24. A record at Aug 25 00:00 is current, one just before is previous. |
| Same inventory used 3+2+4; prior 6; generic correction +2 | Current 9, comparison +50.00%; correction does not change 9. |
| Removed inventory used 4; prior 0 | Still ranked if eligible; `inventoryItemAvailable=false`, trend null. |
| Aug order 10×20; Sep order 4×20; Aug order partially received, corrected, then force-completed in Sep | Aug 200.00, Sep 80.00 throughout; receipts and completion add no spend. See spend `orderedSnapshots` example. |
| An ordered item has unitPrice null | Excluded; the fixed estimate warning remains. Explicit unitPrice 0 contributes a known zero. |
| Two ordered items, each 0.25 units at 0.01, same month | Exact monthly sum 0.005 rounds half-up to 0.01; do not round each product before summing. |
| Draft/Approved/Cancelled lists; ordered snapshot whose Inventory item was removed | First three statuses excluded; retained ordered snapshot still included. |
| Order at local first-month midnight or exactly asOf; order just outside either boundary | Boundary timestamps included; outside timestamps excluded. |
| No orders in the default period at 2026-09-23 | Six zero points April–September, not an empty points array. |

Release evidence for this task is contract validation plus independent review and
user-verified CI. Page QA and API/Web deployment verification do not apply to this
Docs-only change. Runtime isolation, timestamp boundaries, query performance,
ordered snapshot aggregation, and browser acceptance remain gates of their implementation
tasks, not evidence supplied by this document.
