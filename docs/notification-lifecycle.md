# Notification lifecycle and delivery contract

Approved by the user on 2026-09-14 in the Notification planning conversation. This contract describes target behavior, not deployed implementation. [OpenAPI](openapi.yaml) defines HTTP shapes; [client behavior](public-error-codes-and-client-behavior.md) defines navigation and messages.

## Episodes and generation

An episode is a continuous condition for one inventory item and alert type. Low Stock is positive quantity at or below the minimum; Out of Stock is zero. Maintain these immediately on inventory changes. The two are mutually exclusive. Expiry notifications require positive quantity and an expiry date from clinic-local today through day 30 inclusive, scanned daily. The quantity gate does not change Inventory's independent expiry classification.

Generate at most one notification per episode and one recipient per member. Snapshot the original message/product/quantity/threshold/date. Resolve the episode automatically when its condition ends; deleting inventory closes every open episode, zero stock closes expiry, and Low/Out transitions close one and open the other. Recurrence opens a new episode. No manual Resolve endpoint or detailed inventory-transition labels.

Initial backfill creates current episodes and inbox records without Push. Re-enabling a type fills missing records for existing episodes without resetting creation times, restoring dismissed records or replaying Push. Disabling retains history. Keep episode truth even when a type is disabled. Notification/recipient insertion and any enqueue must be recoverable and idempotent.

## Inbox and preferences

In-App is always enabled; reject inAppEnabled writes. Types default on, Email defaults on but requires a verified configured address, and Push defaults off. Settings are scoped to the authenticated member; the destination is email.verifiedAddress, never an automatic broadcast to all login addresses. Default verified-address initialization requires a server-verifiable Auth verification fact; otherwise return null and do not send. Verification-link TTL/rate limits require implementation-readiness evidence before the email-verification task starts.

Read/unread, dismissed and resolved are independent dimensions. Keep resolved history until it ages out or is dismissed. NotificationPage returns asOf; cursors bind window/asOf and createdAt/id descending. The first request without a cursor/asOf uses server time. To open another window, pass the original asOf query parameter; a continuation recovers it from the cursor. A future asOf or a cursor/asOf/window mismatch returns REQUEST_VALIDATION_ERROR 422. Refresh clears both windows/cursors and obtains a new baseline. The baseline freezes time boundaries, not current read/dismiss/resolution state. recent includes [asOf minus 7 days, asOf]; earlier includes [asOf minus 30 days, asOf minus 7 days). The unread count includes only unread, undismissed, unresolved records in the last 30 days. read-all covers undismissed history including resolved records and unloaded Earlier, between through minus 30 days and through. through cannot be in the future.

Opening an inbox does not mark everything read. A click reads then navigates. Dismiss has no confirmation dialog; failure restores the item. Empty copy is “No new notifications” / “暂无新通知”. Low Stock is visually emphasized without disrupting chronological pagination. Longer-running conditions may disappear from history while continuing to appear in email summaries.

## Email summaries

Send Monday 09:00 and Friday 16:00 in the organization's IANA timezone. Build from current open conditions and enabled types at send time, independent of inbox age, read or dismissed state. Order sections Low Stock, Out of Stock, Expiring Soon, with expiry items ordered by date. Omit empty sections and skip an entirely empty email. Distinguish new episodes since the last successful summary from continuing ones; the first summary presents all as new. Continuing episodes can recur in later scheduled summaries.

There are no per-item immediate emails or daily reminders. One or many missed slots coalesce into one pending recovery summary built from latest data, across local dates if necessary. Remove now-resolved/deleted items; do not merge old bodies or replay individual missed slots. Re-enabling Email waits for the next scheduled summary.

Persist logical delivery identity and covered slots. Successful delivery cannot be repeated. A skipped empty summary advances evaluation without claiming successful delivery. Recheck destination and preferences immediately before dispatch. Confirmed failures can be rebuilt from current data; changed payloads must respect provider idempotency-key rules. Unknown outcomes require reconciliation rather than blindly creating another send. Scheduler/worker overlap must not produce both a recovery and a regular email for the same covered slot.

## Push

All three types qualify once on the initial live occurrence. Use stable notification/member/installation delivery identity; token rotation must not permit resending. Registration of a new installation does not replay old notifications.

Before each send/retry check unread, undismissed, open episode, existing inventory, notification within 30 days, enabled type and Push, valid member/installation, and no previous successful send to that installation. There is no additional 24-hour limit. Reading cancels all unsent devices; already delivered system notifications need not be recalled. Preserve the original snapshot, and do not reset its timestamp after queuing.

Persist attempts, leases and successful outcomes. A provider timeout may mean an unknown outcome, not a confirmed failure. Verify provider reconciliation/idempotency behavior during readiness work. Queue at-least-once processing alone cannot establish end-to-end exactly-once delivery.

## Push navigation

After authentication call GET /notifications/{notificationId}/location. visible returns the notification and recent/earlier window; load/highlight it, expanding Earlier as needed, then call read. GET itself has no read side effect. Dismissed/aged-out authorized records return not_in_list with no hidden body; show “This notification is no longer in your list.” Unknown, cleaned-up and other-account targets return the same NOTIFICATION_UNAVAILABLE 404. Network/5xx means retry, not absence.

Resolved and deleted-inventory history is still locatable. Deleted inventory only disables product navigation; show “This inventory item has been deleted.” if attempted. No hidden-notification detail screen or automatic undo of Dismiss.

## Compatibility and verification

The API currently has a target contract ahead of implementation. Inspect actual consumers before removing obsolete In-App writes. Publish and validate the additive location/resolvedAt/asOf contract, implement the API, then update Web/App consumers. Device revocation uses X-Installation-Id to select the current authenticated installation. Do not change deployed resources as part of document synchronization.

Core integration evidence covers tenant/member isolation; 7/30-day and expiry 0/30/31-day boundaries; concurrent read-all/generation; backfill and preference re-enable; deletion/zero-stock resolution; summary coalescing, empty sections and timezone/DST; and successful Push deduplication across token rotation, read/dismiss and unknown provider outcomes. Actual email and Push tasks require controlled mailbox/device evidence, not only mocks.

### Consumer audit (2026-09-14)

The fetched Web staging commit `2dfac3a8bd11f5d8a35649f5acfb7782038bde50` still seeds `INITIAL_NOTIFICATIONS` in `src/features/inventory/InventoryApp.tsx`; `GeneralSettingsTab.saveNotifications` only changes local state. No `/notifications`, `device-tokens` or `inAppEnabled` HTTP consumer was found in `src`. The fetched API staging commit `555af152e85a6f5df50d850a0efd1dc30572ade1` has no Notification/device-token implementation in `app`. These are repository-source observations, not live deployment verification. App source is not available in this workspace; its owner must audit consumers before App rollout.

Delivery order: merge this target contract first; implement and verify API inbox/settings/location in their own slices; then integrate Web/App against those verified APIs. If another deployed consumer is discovered, update it to omit `inAppEnabled` before enforcing rejection; do not silently reinterpret a false value as an active switch. Location, required `resolvedAt`/`asOf`, the structured navigation object and the installation selector must be available before a client relies on them. A contract-only rollback reverts this document change; any later API/client rollout needs its own compatibility-aware rollback and must preserve read/dismiss and delivery identities.

### Contract acceptance evidence

| Criterion | Evidence in this change | Deferred runtime gate |
|---|---|---|
| History, stable windows and unread count | List/count descriptions and NotificationPage shape | Database pagination and exact boundary tests |
| Snapshot, resolution and safe Push location | Notification/NotificationLocation schemas, examples and error mapping | Member isolation and deleted-history API tests |
| Read, read-all and Dismiss | Idempotent mutation descriptions and cutoff semantics | Concurrent mutations and queued-Push filtering |
| Preferences and verified destination | Settings read/write schemas and partial-update example | Persistence, verification and initialization facts |
| Installation, Push and summary delivery | Device selector and lifecycle sections | Provider reconciliation, deduplication, scheduler/DST and controlled delivery |
| Public compatibility | Consumer audit above; OpenAPI lint/bundle | API deployment before client integration |

For this documentation slice, Core fixtures are the OpenAPI examples and approved PRD/Schema, all available locally. Page QA, database migrations, deployed-service smoke tests, real credentials and provider calls are not applicable to contract publication. Tenant/data safety, concurrency and recovery are reviewed as contract requirements; proof of their implementation belongs to the linked API/Platform tasks. No runtime resource is provisioned or asserted ready here.

Pagination boundary example: load Recent with `asOf=2026-09-14T10:00:00Z`, then expand Earlier an hour later using the same asOf. A notification created `2026-09-07T10:30:00Z` remains only in Recent. A notification exactly at `2026-09-07T10:00:00Z` belongs to Recent; one just before it belongs to Earlier. Both requests share the same 30-day lower boundary. A cursor for Recent cannot be used for Earlier.
