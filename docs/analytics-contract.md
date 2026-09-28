# Analytics forecast and purchase spend contract

This document defines the two read-only Analytics APIs. The previous Analytics
overview and consumption-trend drafts have no runtime consumers and are replaced
by these contracts. Dashboard Purchase Spend remains unchanged.

## Shared rules

- Every request uses the authenticated active member's organization. Clients
  cannot provide an organization ID. All dates use the organization's IANA
  timezone. Monetary responses use its current currency code. Changing that
  setting relabels historical numeric amounts; it does not convert them.
- The response separates `baselineThroughDate` (the latest clinic-local day
  included in consumption) from `stateReadAt` (the live inventory and purchase
  state read). A normal daily baseline includes usage through yesterday. The
  request never scans historical activity to repair a missing baseline.
- Only negative Inventory Activity with `reasonCode=product_used` contributes to
  demand. Corrections, damage, expiry, and other decreases do not. A SKU must
  have been observable for at least 60 clinic-local calendar days and have
  positive usage in at least four distinct Monday-start weeks. Otherwise its
  rate and forecasts are unavailable, never fabricated as zero. The observation
  interval begins when the inventory item becomes active; weekend Product Used
  events remain in the numerator and the denominator counts Monday-to-Friday
  business days. A valid zero
  rate is possible only after both eligibility checks pass.
- The daily job stores a per-inventory-item demand rate, algorithm version and
  cutoff date. It is rerunnable by organization and date. A missing baseline,
  failed run, or a cutoff before yesterday is reported as unavailable or stale;
  stale values are visibly marked if used. Live requests batch-read current
  inventory, ordered/received quantities, list states and price snapshots.
- Decimal quantities and money are JSON strings. Money is accumulated using
  Decimal and rounded half up to two places only at the monthly or response
  total boundary. A missing price is unavailable, whereas a recorded zero
  price is valid. No exchange conversion is performed.

## Buying needs and order timing

`GET /analytics/forecasted-buying-needs` returns three summary cards, a
paginated Buying Needs table, and calendar counts from one organization-wide
projection. `coverageDays` accepts 7, 14, or 30 and defaults to 30. The risk
window is always the next 30 clinic-local calendar days, including today;
changing coverage changes recommended quantities and spend, not risk counts.
The table contains only SKUs with an additional recommended purchase greater
than zero and `orderByDate` in the half-open 30-day window
`[today, today + 30 calendar days)`; risk sets may overlap. Summary and calendar cover all matching
SKUs, regardless of pagination or an optional order-date filter. Each row is
keyed by `inventoryItemId` and labels its inventory quantity unit.

For each item, `A = currentStock + incomingStock`, where incoming is the sum of
`max(plannedQuantity - receivedQuantity, 0)` across Ordered purchase-list
items. Draft, Approved, Cancelled and already received quantities do not add
to incoming. Multiple rows for one inventory item are combined. Incoming is
available for planning from the request date; the API does not invent an ETA
for older orders or write stock. This optimistic timing assumption is returned
in the response.

Let `M` be Minimum Quantity, `d` be mean usage per Monday-to-Friday business
day, `L` be lead time in business days, and `U(a,b)` be expected usage on
business days in the half-open interval `[a,b)`. The first projected date at
or below M and at or below zero are distinct. The order-by date is the first
Monday-to-Friday clinic-local order date `t`, no earlier than today, for which projected stock upon arrival,
`max(0, A - U(today, t + L))`, is at or below M. If this already holds, return
today when it is a business day, or the next Monday when today is a weekend.
`t + L` advances by L Monday-to-Friday business days. A trigger reached on a
weekend is scheduled for the next Monday; the projected stockout date may
precede the estimated arrival date and remains visible in the response. The arrival stock is
`S = max(0, A - U(today, orderBy + L))`. Raw additional
quantity is `max(0, M + U(arrival, arrival + R calendar days) - S)`; convert
to the purchasing unit, round up to its pack multiple and minimum order
quantity, then return the exact equivalent in the stated inventory unit. A failed or
unsafe unit conversion makes that item's quantity and estimated amount
unavailable. Already ordered incoming is never billed again.

At least three valid received order samples are needed for historical `L`.
Each clinic-wide sample runs from an Ordered timestamp to its first positive
receipt; a missing or reversed timestamp is invalid. Convert both timestamps
to the clinic timezone, then count Monday-to-Friday dates strictly after the
order date through the first receipt date, inclusive. A receipt on a later
local date counts at least one business day, even when it falls on a weekend;
only a same-local-date receipt may have zero lead days. Do not trim valid long
durations. Use the median business-day duration for valid samples, rounding a
half-day median up to the next whole business day; fewer than three uses five
business days. Return the source
and sample count. This is a planning estimate, not a supplier ETA.

For example, if the clinic-local date is Saturday 2026-10-03 and the threshold
is already met, `orderByDate` is Monday 2026-10-05. With a five-business-day
lead time, estimated arrival is Monday 2026-10-12. Holidays are not subtracted
in this MVP estimate.

The latest valid Ordered/Completed price snapshot for the same inventory item
prices additional quantity. `estimatedTotal` is the rounded product of that
quantity and unit price. The top 30-day spend is the sum of all priceable matching table
rows before pagination; its price coverage counts priceable and unpriced
recommendations. Risk counts, table and calendar use the same SKU projection.
When an entire baseline is unavailable, the API returns an explicit data state
and no invented forecast totals.

## Purchase Spend trend and breakdown

`GET /analytics/purchase-spend` returns a continuous 6, 12 or 24 month actual
series, including the current partial month, followed by exactly the next two
complete clinic-local calendar months as forecast. Historical actuals use the
existing Dashboard rule: Ordered/Completed `plannedQuantity * unitPrice`
snapshots in the local `orderedAt` month. Draft/Approved/Cancelled and missing
prices are excluded; receipt, correction and completion never move an amount
to another month. The current month contains actual orders only. Its point
is marked `actual`, never joined to a future forecast as if it had occurred.

Future points represent new recommended orders, grouped by forecast Order By
month and current category. They run the same demand, incoming, lead-time,
coverage and price rules as Buying Needs, extending virtual replenishment
cycles in memory through the end of the second future month. A virtual cycle
does not create a purchase list or change inventory. Each predicted point is
marked `forecast`, and unavailable SKUs are excluded with explicit coverage.
The 30-day summary amount is a different time window and need not equal a
calendar-month point.

The selected month breakdown uses the same rows and exact total as its trend
point. Both historical Actual and future Forecast amounts use each product's
current category. Changing that category reclassifies prior months without
changing their total amount; the purchase snapshot has no category field.
Missing categories go into an explicit `uncategorized` bucket; an item
is counted once. Amounts are summed as Decimal before rounding, so the
response includes a rounding adjustment if independently rounded category
amounts do not sum to the rounded total. A future month requires a forecast
point; earlier months, including the current month, require an actual point.
Empty months contain `0.00` only when the relevant data is genuinely known;
unavailable forecasts have a separate status.

Both endpoints return the usual 401/403/422 envelopes and a 503 when a
required dependency cannot be read. Empty inventory and valid zero demand
are ordinary 200 results. The organization boundary applies to all source
rows, including the daily baseline and purchase snapshots.
