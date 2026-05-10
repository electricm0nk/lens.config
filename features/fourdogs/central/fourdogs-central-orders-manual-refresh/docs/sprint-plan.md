---
feature: fourdogs-central-orders-manual-refresh
doc_type: sprint-plan
status: ready
goal: "Two stories: (1) backend route registration, (2) frontend hook + component. Sequenced — backend first so the frontend story can verify against the real endpoint."
updated_at: "2026-05-10"
depends_on: [fourdogs-etailpet-api-trigger-sync-fix]
---

# Sprint Plan — fourdogs-central-orders-manual-refresh

**Track:** quickdev | **Sprint:** 1  
**Total stories:** 2 | **Dependency:** fourdogs-etailpet-api-trigger-sync-fix (tsf-1-01, tsf-1-02 must be done first)

---

## Story omr-1-01 — Register /v1/etailpet/trigger in session-protected route group

**Status:** backlog  
**Points:** 1  
**Repo:** `fourdogs-central` (Go)  
**Depends on:** tsf-1-02 (EtailpetTrigger handler must exist)

### User Story

As a session-authenticated user,  
I want `POST /v1/etailpet/trigger` to be reachable via session cookie auth,  
So that the central UI can call the trigger endpoint without needing a bearer token.

### Acceptance Criteria

1. **Prerequisite gate**: omr-1-01 MUST NOT begin until tsf-1-02 is merged to `develop` in `fourdogs-central` — Go build CI must pass with `handler.EtailpetTrigger` present
2. `cmd/api/main.go` registers `r.Post("/v1/etailpet/trigger", handler.EtailpetTrigger(etailpetClient, ""))` inside the session-protected `r.Group`
3. `etailpet.Client` is instantiated at server startup in `main.go` — if `cfg.EtailpetClientID`, `cfg.EtailpetClientSecret`, `cfg.EtailpetTenantSchema` are absent from `internal/config/config.go`, add them before wiring the client
4. `POST /v1/etailpet/trigger?source=inventory` returns HTTP 200 with a valid session cookie
5. `POST /v1/etailpet/trigger?source=inventory` returns HTTP 401 when called without a session
6. The `/internal/etailpet/trigger` route from feature 1 is unchanged and still requires bearer token
7. `go build ./cmd/api/...` passes

### DoD

- Code reviewed
- `go build ./cmd/api/...` green
- Manual smoke test: `POST /v1/etailpet/trigger?source=inventory` returns 200 with session cookie
- Commit: `feat(api): register /v1/etailpet/trigger for session-authenticated UI access (omr-1-01)`

---

## Story omr-1-02 — Add Sync Now button to DataFreshnessPanel

**Status:** backlog  
**Points:** 3  
**Repo:** `fourdogs-central-ui` (React/Vite)  
**Depends on:** omr-1-01 (route must be live in dev)

### User Story

As a central operator,  
I want to see a "Sync Now" button in the data freshness panel when any source is stale,  
So that I can trigger an on-demand etailpet data pull without restarting anything or waiting for the scheduler.

### Acceptance Criteria

1. New file `src/hooks/use_etailpet_trigger.ts` — `useMutation` wrapping `api.post('/v1/etailpet/trigger?source=all', {})`, `onSettled` invalidates `['data-freshness']`
2. `DataFreshnessPanel.tsx` shows "Sync Now" button when any `entry.stale === true`
3. "Sync Now" button is hidden (or replaced with "✓ Up to date") when all sources are fresh
4. While mutation is pending: button shows "Syncing…" and is `disabled`
5. On success (HTTP 200 or 207): freshness panel refetches automatically (via `invalidateQueries`)
6. On error (HTTP 4xx/5xx): panel shows inline error text "Sync failed — try again or wait for scheduled pull."
7. Existing `↻ Refresh` button (refetch freshness metadata) is unchanged
8. `src/__tests__/data-freshness.test.tsx` extended with: stale→button visible, fresh→button hidden, pending state ("Syncing…"), error state (inline message), and 207-partial-success→freshness-refetch tests
9. `npm test` passes

### DoD

- Code reviewed
- `npm test` green (all tests including new ones)
- Manual smoke: dashboard loads, stale source triggers button visibility, click syncs, freshness panel refreshes
- Commit: `feat(ui): add Sync Now button to DataFreshnessPanel when stale (omr-1-02)`
