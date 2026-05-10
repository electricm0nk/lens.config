---
feature: fourdogs-central-orders-manual-refresh
doc_type: business-plan
status: ready
goal: "Add a Sync Now button to the Dashboard data freshness panel so operators can trigger an on-demand etailpet data fetch when one or more sources are stale."
key_decisions:
  - "Button appears only when at least one freshness source has stale: true"
  - "Calls POST /v1/etailpet/trigger?source=all via session cookie auth (not bearer token)"
  - "Backend registers the existing EtailpetTrigger handler at /v1/etailpet/trigger with empty token — session middleware covers auth"
  - "On success or partial success (207), DataFreshnessPanel refetches freshness data"
  - "Error shown inline in the panel — no toast dependency required"
depends_on: [fourdogs-etailpet-api-trigger-sync-fix]
blocks: []
updated_at: "2026-05-10"
---

# Business Plan — fourdogs-central-orders-manual-refresh

## Purpose

Operators currently have no way to force etailpet data to refresh on demand from the UI.
When inventory, sales, or transactions data is stale (age > threshold), the `DataFreshnessPanel`
on the Dashboard shows a warning icon but offers only a `↻ Refresh` button that re-fetches
the freshness metadata — it does **not** trigger a new etailpet data pull.

This feature adds a **"Sync Now"** button to the `DataFreshnessPanel` that triggers an
on-demand etailpet ingest via the trigger endpoint added in `fourdogs-etailpet-api-trigger-sync-fix`,
then refreshes the freshness display after the call completes.

## In Scope

- Dashboard → `DataFreshnessPanel`: Show "Sync Now" button when any source has `stale: true`
- Button visible only when stale; hidden (or replaced by "✓ Up to date") when all sources are fresh
- Loading state: button shows spinner/disabled while POST is in flight
- Success path: refetch freshness data; button returns to normal state
- Partial success (HTTP 207): treat as success — refetch freshness
- Error path: display inline error message within the panel; do not crash or nav away
- Backend: register `POST /v1/etailpet/trigger` in the session-protected route group in `cmd/api/main.go` (uses the `EtailpetTrigger` handler from feature 1 with `token=""`)
- Frontend test: vitest test for the updated `DataFreshnessPanel` (stale state shows button, non-stale hides it, loading state, error state)

## Out of Scope

- Per-source trigger buttons (only `source=all` is wired in this feature)
- Toast notifications (inline error text is sufficient)
- Trigger history or audit log in the UI
- The etailpet client implementation (covered by fourdogs-etailpet-api-trigger-sync-fix stories tsf-1-01, tsf-1-02)
- Any changes to the `/internal/etailpet/trigger` bearer token route

## Success Criteria

- Operator sees the panel on Dashboard when any source is stale
- Clicking "Sync Now" sends `POST /v1/etailpet/trigger?source=all`
- After the call resolves, the freshness panel refreshes to show updated `age_minutes`
- If the backend returns an error, the panel shows an inline message; the button re-enables
- No regressions to the existing `↻ Refresh` (refetch freshness metadata) button
- All existing and new vitest tests pass

## Dependency

This feature depends on `fourdogs-etailpet-api-trigger-sync-fix` delivering:
- `EtailpetTrigger` handler in `internal/handler/etailpet_trigger.go`
- `EtailpetTriggerClient` interface with `TriggerInventory`, `TriggerSales`, `TriggerTransactions`

Feature 2 **only adds the route registration and the UI**. The implementation of the
etailpet trigger logic lives in the parent feature.
