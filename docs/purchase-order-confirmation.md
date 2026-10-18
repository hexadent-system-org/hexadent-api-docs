# Purchase item order confirmation

This is the contract-first extension approved on 2026-10-18. It does not claim
runtime availability. Deliver Docs, then the API expansion and migration, then
Web consumption. It adds no infrastructure, payment, invoice, refund or promotion
engine. Every amount is in the existing organization currency; there is no FX.

## Quantities and amount sources

| Item field | Meaning |
| --- | --- |
| `plannedQuantity` | Existing, immutable Requested Qty copied from Draft at approval. Never repurpose this field as Ordered Qty. |
| `orderedQuantity` | Actual quantity to order, initially equal to `plannedQuantity`; editable only while Approved. New edits are positive, at most 999999.99, at most two decimals; historical outputs retain older valid positive quantities without this new input cap. |
| `unitPrice` | Existing immutable purchase-unit price snapshot; this extension does not introduce price editing. |
| `estimatedTotal` | Nullable display amount: `orderedQuantity * unitPrice`, rounded half up to two decimals. Null when unit price is unknown. |
| `actualTotalOverride` | Nullable persisted manual item total, nonnegative and at most 999999999999.99, at most two decimals. Null means use the estimate; zero is a real override. |
| `actualTotalReason` | Nullable persisted explanation, trimmed Unicode length 1–500 when an override exists. |
| `actualTotal` | Nullable effective display amount: override when non-null, otherwise estimate. This is confirmed item order value, not money paid. |

New live item responses always include the five new fields. Historical cached or
idempotent responses may omit them: fall back to `orderedQuantity = plannedQuantity`
and no manual override, deriving estimates only if the old price is known. Do not
write these read fallbacks back as manual overrides. Do not add a persisted
`manuallyEdited` flag or infer source from two amounts being equal.

Approval copies the original quantity into both quantity fields and initializes
both override fields to null. Historical database rows receive that same default;
no receipt, inventory, cancellation reason or historical status is rewritten.
A historical quantity above the current input cap remains readable/backfilled
without clamping; existing legacy Approved cancellation restoration exceptions
remain as documented. New explicit quantity edits use the current bounded input.
Draft has no actual-total fields. An item's return to Draft never copies its manual
amount or amount reason into another purchase; a later approval starts afresh.

A null override follows later Ordered Qty edits while Approved. A manual override,
even one equal to the estimate, stays unchanged when quantity changes, with no
reconfirmation prompt or warning. Its reason is not a cancellation reason.
Missing price and no override means unknown, not zero; missing amount does not
prevent ordering. An override can provide a known amount without a unit price.

The explicit **Save** action submits only fields the user actually changed. Merely
showing an estimated default, focusing/blurring it or saving a quantity must not
create a manual override. Restore estimate explicitly clears both override and
current reason; historical audit entries remain. No blur autosave is introduced.

## Approved item mutation

`PATCH /v1/purchase-list-items/{itemId}/confirmation`, with required
`Idempotency-Key`, applies one item's explicit Save atomically. Authentication,
organization/membership availability and item lookup follow existing purchasing
rules. All active organization members may perform it, matching Mark as Ordered;
only initial approval remains restricted to Admin/Manager. No reapproval is added.

Required body: `expectedListVersion`, `expectedItemVersion`. At least one of
`orderedQuantity`, `actualTotalOverride`, `actualTotalReason` is required. Unknown
properties are rejected. Quantity cannot be null or zero. Omission leaves a field
unchanged. Manual amount and reason have the following coupled rules:

| Supplied fields | Result |
| --- | --- |
| Quantity only | Preserve override and reason, including nulls. |
| Non-null override plus nonblank reason | Replace both, even if effective amount equals the estimate. |
| Non-null override without a reason in this request | 422; an old reason cannot silently justify a new manual amount. |
| Null override, with reason omitted or null | Clear override and current reason; resume estimate. |
| Null override plus non-null reason | 422. |
| Reason only | Allowed only with an existing non-null override; must remain nonblank. Otherwise 422. |

Example explicit Save:

```json
{
  "expectedListVersion": 2,
  "expectedItemVersion": 1,
  "orderedQuantity": "6.00",
  "actualTotalOverride": "50.00",
  "actualTotalReason": "Buy five, get one free"
}
```

Return `200 PurchaseListDetail` with current list and item versions. Lock the
parent list before its item, compare both versions, and require Approved state;
order/cancel, empty-list deletion and competing saves serialize on that parent. A material change
increments list and item versions and updates list `updatedAt`; immutable product,
price, submitted-by and approver snapshots do not change. No inventory, activity
or receipt is created. A current-version no-op completes idempotency but does not
increment versions or create a change audit event. Stale versions are rejected
even for a numerically identical request.

Existing idempotency rules apply: scope operation plus organization plus key,
fingerprint item ID and normalized body including expected versions and reason;
same key/material payload replays the original successful response, changed
payload conflicts. Authorize and scope before replay; do not rerun a successful
mutation after ordering or cancellation. Replay snapshots are not enriched with
new current values. Normalize equivalent decimal strings before fingerprinting.
Audit material changes with item/list IDs, real actor, before/after quantity,
override and reason, request correlation, versions and outcome; retain history
when clearing an override. Do not create a replacement audit system.

Use the existing error envelope and field errors: 401/403 access, 404 unknown or
cross-organization item, 409 `PURCHASE_LIST_STATE_CONFLICT` for non-Approved,
409 `PURCHASE_LIST_VERSION_CONFLICT` for stale list/item, existing idempotency
conflict for changed-key payload, and 422 `REQUEST_VALIDATION_ERROR` for invalid
quantity, amount, paired fields or an empty edit. No partial writes on failure.

## Ordering, receiving and cancellation

Mark as Ordered uses the latest successfully saved list version; it does not
implicitly save local edits. Web must disable/resolve Mark as Ordered and Approved item Cancel while a
quantity/amount edit is dirty, saving or failed; explicitly save or discard local
input before Cancel, which always uses saved server quantities. Ordering freezes Ordered Qty,
override and reason. After ordering neither quantity nor amount can be edited,
even before first receipt. Completed and Cancelled are also read-only.

All new live receipt statuses, receipt counts, completion eligibility and current
completion item snapshots compare received quantities with `orderedQuantity`:

- `remainingQuantity = orderedQuantity - receivedQuantity` (may be negative).
- `varianceQuantity = receivedQuantity - orderedQuantity`.
- `pendingQuantity = 0` for Cancelled, otherwise `max(remainingQuantity, 0)`.
- `cancelledQuantity = remainingQuantity` when cancelled, otherwise zero.
- Over-receipt warnings retain legacy `plannedQuantity` as Requested Qty, add
  `orderedQuantity`, and compute `overReceivedBy` against Ordered Qty.
- Complete stays manual: every item must be received/over-received or cancelled.
  Historical Completed lists are not reopened; old completion snapshots replay
  as stored. The deprecated historical variance-confirmation schema is unchanged.

For Requested 4 / Ordered 5: receive 2 leaves 3; receive another 3 is received,
not over-received; total 6 is over-received by 1. Correction adjusts received
stock only, never Ordered Qty or item amount. Cancel with return-to-Draft restores
3 in the first example. Approved whole-list cancellation also restores Ordered
Qty, following its existing metadata and submitted-by conflict rules. Ordered
item cancellation retains its existing optional return choice, metadata rules,
reason, no-receive/no-correction rule, idempotency and atomic Draft merge. Approved item Cancel restores all saved Ordered Qty (Requested 4 / Ordered 5
returns 5), removes the row and requires the cancellation capability header.
Single-item submitter conflicts may be explicitly confirmed against the current
Draft ID/version; whole-list conflicts remain blocking. Overflow or an
unconfirmed/stale merge conflict rolls the whole cancellation back.

Neither partial receiving nor Ordered item cancellation reduces the confirmed
item amount or infers a refund. Approved item Cancel removes that row and its
amount from the current list, as defined in the [cancellation contract](purchase-item-cancellation.md). Quantity progress and active incoming exclude cancelled items as
before; money and inventory movement are separate. There is no new refund field.

## Read surfaces, money and export

- Detail and all nested item mutation responses include the new fields. Paginated
  preview items add `orderedQuantity`, retaining original `plannedQuantity`.
- Active Draft incoming summaries keep their existing wire keys: their aggregate
  `plannedQuantity` now sums effective Ordered Qty, with `receivedQuantity` and
  remaining from the same non-cancelled Approved/Ordered set. This aggregate is
  not the immutable Requested Qty field of an individual purchase item.
- Forecast incoming stock uses non-cancelled Ordered items' positive Ordered Qty
  minus Received Qty. Future recommended-price selection remains the existing
  unit-price-snapshot method: do not infer a new unit price from a discounted
  item total or spread an override over forecast months.
- Dashboard and Analytics historical purchase spend, and any existing list-value
  totals, use manual override when non-null, otherwise the precise Decimal
  Ordered Qty × price product. Keep existing list-state/date/category/tenant
  filters and ordered-month attribution. Cancelled item flags or partial receipts
  must not be treated as refunds; Ordered/Completed items' full confirmed amounts
  remain. Whole-list Cancelled remains excluded by the existing status filter.
- Null override and null price is unpriced, not zero; zero override is priced,
  including without a unit price. Preserve coverage counts/qualifications. Keep
  existing aggregation rounding: sum precise products/overrides then half-up at
  the total boundary. Item `estimatedTotal`/`actualTotal` are rounded display
  amounts; do not sum rounded item displays for monthly aggregation. No currency
  conversion or financial-payment claim is introduced.
- CSV/PDF export remains Approved-only with existing item limits, filenames,
  business-name snapshots and failure responses. The export columns now show
  both quantities and both amounts explicitly, as approved on 2026-10-18:
  `Vendor Item Number`, `Vendor`, `Description`, `Unit`, `Requested Qty`,
  `Ordered Qty`, `Unit Price`, `Est. Total`, `Actual Total`, `Actual Total Reason`,
  `Submitted By`; CSV additionally ends with `Approved By`. PDF retains its
  existing Approved By / list name / approved at / notes metadata.
- `Requested Qty` exports immutable `plannedQuantity`; `Ordered Qty` exports
  `orderedQuantity`. `Unit Price` remains the snapshot. `Est. Total` exports
  `estimatedTotal`; `Actual Total` exports effective `actualTotal`, even when it
  differs from the estimate. Both total columns round half up to two decimals,
  independently of aggregate calculations. Unknown totals are empty cells;
  manual zero exports `0.00`, including when price and estimate are unknown.
  `Actual Total Reason` exports the persisted reason, or an empty cell when no
  manual override exists. Restoring the estimate clears that current export
  reason but does not erase audit history. There is no editable whole-order total.
- This explicitly replaces the earlier `Qty` / `Total Price` column layout; it is
  a file-format change, not additional aliases alongside ambiguous old columns.
  CSV keeps UTF-8 BOM, RFC 4180 quoting and one header with no metadata preamble.
  Existing formula-injection protection also covers `Actual Total Reason` as a
  free-text field. PDF retains all fields with readable wrapping, page breaks
  and repeated table headings; long reasons must not truncate or overlap rows.
  All text, including the reason, participates in existing PDF character checks.

### Export acceptance examples

| Case | Requested Qty | Ordered Qty | Unit Price | Est. Total | Actual Total | Actual Total Reason |
| --- | --- | --- | --- | --- | --- | --- |
| Requested 4, order 6, price 10, manual 50 | 4.00 | 6.00 | 10.00 | 60.00 | 50.00 | Buy five, get one free |
| Same item after restoring estimate | 4.00 | 6.00 | 10.00 | 60.00 | 60.00 | Empty |
| Price unknown, manual zero | 4.00 | 4.00 | Empty | Empty | 0.00 | Promotional gift |
| Price and manual total unknown | 4.00 | 4.00 | Empty | Empty | Empty | Empty |

CSV and PDF must both preserve the independent fields and current values. Test
reason text with commas, line breaks and formula prefixes in CSV, and long/multi-
page reasons plus unsupported characters in PDF. The UI continues downloading
server-generated files; it must not reconstruct totals from Requested Qty.


## Legacy consumer protection and rollout

The local Web and React Native services currently read `plannedQuantity` for
requested/receipt calculations; they do not yet consume these new fields. Keeping
that immutable field alone is not sufficient compatibility. New quantity/amount
editing must not be exposed before the new Web consumer is verified.

New clients send `X-Purchase-Confirmation-Version: 1` on purchasing mutations.
The new confirmation route requires it. Existing order, Approved whole-list
cancellation and completion require it if **any** item differs from its Requested
Qty or has a non-null manual override; receipt, correction and item cancellation
apply the same guard to their target item. Unchanged old orders retain existing
headerless behavior. This header declares supported interpretation, not identity,
permission or security authorization. Existing organization/role checks still run.

A new non-replay mutation without that capability on a changed record returns
409 `PURCHASE_CONFIRMATION_CLIENT_REQUIRED`, no side effects, and safe details
`requiredConfirmationVersion: 1`. Unsupported supplied versions fail with the
same code, even on unchanged records. Preserve authorized same-key/material
successful historical replays before current state/capability checks, as for
other lifecycle changes. New clients always send the header, even on old records;
new keys after conflict require rereading current fields and user confirmation.

Old clients may still read Requested Qty, but cannot perform unsafe new mutations
on changed items. A clear 409 upgrade/supported-client message is the fallback;
never silently accept a wrong receipt or reinterpret an old field. Read/export
endpoints remain accessible and server-generated exports use the new values.
CORS must allow the capability header using the existing API CORS mechanism;
this introduces no new environment variable or external service.

Delivery order:
1. Merge and validate this contract, including lint/bundle and independent review.
2. API adds backward-compatible nullable fields and backfills Ordered Qty from
   Requested, constraints, atomic mutation, guards, derived reads and audit.
   Verify upgrade and historical queries/replays in an isolated database. Schema
   downgrade must refuse loss of changed Ordered Qty, manual values or reasons.
3. User confirms API CI, migration and deployed candidate. New Web implements
   quantities/amounts/Save and capability header; verify exact API/Web candidates
   before enabling editing. APP update is outside these tasks; old APP must fail
   safely on changed records until a separate authorized APP task is delivered.
4. Rolling back Web must not enable unsafe old writes: retain the API guard and
   extended database. Do not roll back API to an unguarded binary while changed
   records exist. Disabling editing does not erase saved quantities or totals.

## Contract acceptance examples

| Case | Expected result |
| --- | --- |
| Requested 4, Ordered 5, price 10, no override | Estimate/effective amount 50, received 5 is complete. |
| Requested 5, Ordered 6, price 10, manual 50 + reason | Estimate 60, effective amount 50; buy-five-get-one needs no unit-price change. |
| Change Ordered 6 to 7 after that Save | Estimate 70, manual/effective amount stays 50; no confirmation prompt. |
| Save quantity only with untouched displayed default | Override stays null and continues following estimate. |
| Explicit manual amount equal to estimate | Non-null override and reason persist, even though displayed amounts match. |
| Restore estimate using explicit null override | Reason clears, next quantity change updates effective amount. |
| Price null, override null / override 0 | Unknown / known zero; both may be ordered. |
| Override without reason or negative / overprecision amount | 422, no quantity or amount changes. |
| Requested 4, Ordered 5, received 2, cancel and return | Draft grows by 3 atomically; stored manual amount and historical spend do not reduce. |
| Headerless old receipt against changed Ordered Qty | 409 upgrade requirement, inventory unchanged. |
| Historical Requested/Ordered 1000000.00 | Read/backfill remains 1000000.00 without clamping; a new explicit edit to 1000000.00 is 422. |
| Two saves / Save racing Mark as Ordered | One serializable winner; stale or non-Approved request conflicts, never partial state. |

## Approved item Cancel compatibility

The [cancellation extension](purchase-item-cancellation.md) independently requires
`X-Purchase-Cancellation-Version: 1` for Approved item Cancel, single-item Draft
submitter overwrite confirmation and empty Approved deletion. It does not replace
`X-Purchase-Confirmation-Version: 1` where this contract requires it. Old successful
replays remain first. An empty Approved list cannot be ordered or whole-list
cancelled (`PURCHASE_LIST_EMPTY`); it stays visible until manually deleted.
