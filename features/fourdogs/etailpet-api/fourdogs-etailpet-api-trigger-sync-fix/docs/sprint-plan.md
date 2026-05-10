---
feature: fourdogs-etailpet-api-trigger-sync-fix
doc_type: sprint-plan
status: draft
goal: "Deliver POST /internal/etailpet/trigger in a single focused sprint."
key_decisions:
  - Single sprint, one developer, roughly one day of work
  - No pre-spike needed — etailpet.NewClient.Trigger() is already proven in the trigger pods
  - Handler pattern is a direct lift of risk_refresh.go
open_questions: []
depends_on: []
blocks:
  - fourdogs-central-orders-manual-refresh
updated_at: "2026-05-10"
---

# Sprint Plan — fourdogs-etailpet-api-trigger-sync-fix

## Sprint Overview

| Sprint | Goal | Stories | Complexity | Risks |
|---|---|---|---|---|
| 1 — Trigger Endpoint | Add POST /internal/etailpet/trigger to central API | tsf-1-01, tsf-1-02 | S+S | etailpet.Client may need TriggerSales/TriggerTransactions methods added |

**Story size legend:** `S` ≈ half-day (single developer).

---

## Sprint 1 — Trigger Endpoint

**Goal:** Merge a working `POST /internal/etailpet/trigger` endpoint into `develop` and
confirm the central dev deployment responds correctly.

| Story | Title | Estimate |
|---|---|---|
| tsf-1-01 | Extend etailpet.Client with per-source trigger methods + unit tests | S |
| tsf-1-02 | Add EtailpetTrigger handler and route | S |

---

### Story tsf-1-01 — Extend etailpet.Client with per-source trigger methods

**As a** handler author,
**I want** `etailpet.Client` to expose `TriggerInventory`, `TriggerSales`, and `TriggerTransactions` methods,
**So that** the HTTP handler can invoke each source independently.

**Acceptance Criteria:**
- [ ] `TriggerInventory(ctx)` invokes the same logic as `cmd/etailpet-trigger/main.go` uses
- [ ] `TriggerSales(ctx)` invokes the same logic as `cmd/etailpet-sales-fetcher/main.go` uses (read the sales fetcher main.go first)
- [ ] `TriggerTransactions(ctx)` verified or stubbed with error if no transactions trigger path exists (verify by inspecting `internal/etailpet/client.go` before assuming)
- [ ] Each method returns `error` only (fire-and-forget; etailpet Trigger() is async — no row count available)
- [ ] Existing `Trigger()` method is preserved for backward compatibility (no breaking change)
- [ ] Unit tests mock the etailpet HTTP client; test success and error paths for each method

**Implementation Notes:**
- Read `cmd/etailpet-sales-fetcher/main.go` and `internal/etailpet/client.go` before assuming method signatures
- If transactions has no separate trigger path, scope tsf-1-01 to inventory + sales only and track transactions as follow-on

---

### Story tsf-1-02 — Add EtailpetTrigger handler and route

**As an** internal caller (central-ui or on-call engineer),
**I want** to POST to `/internal/etailpet/trigger?source={source}` and have the etailpet fetch cycle run immediately,
**So that** stale data can be refreshed without waiting for the next scheduled pod run.

**Acceptance Criteria:**
- [ ] `POST /internal/etailpet/trigger?source=inventory` returns `200 OK` with `{status:"ok", source:"inventory", triggered_at:"...", duration_ms:N}`
- [ ] `POST /internal/etailpet/trigger?source=sales` works identically
- [ ] `POST /internal/etailpet/trigger?source=all` triggers all available sources; returns `207` with per-source result array on partial failure
- [ ] Unknown source returns `400` with `{error:"unknown source"}`
- [ ] Client error returns `500` with `{status:"error", source:..., error:"..."}`
- [ ] Missing or invalid bearer token returns `401`
- [ ] Route is registered in the API server router alongside `/internal/velocity-risk/refresh`
- [ ] Handler has table-driven unit tests covering all source variants, partial failure, 401, and error paths

**Implementation Notes:**
- File: `internal/handler/etailpet_trigger.go` (mirror of `risk_refresh.go`)
- Test file: `internal/handler/etailpet_trigger_test.go`
- Bearer token same mechanism as `VelocityRiskRefresh(database, token string) http.HandlerFunc`

---

## Definition of Done

- Both stories merged to `develop`
- central-dev ArgoCD synced and healthy
- Manual smoke test: `curl -sk -X POST https://central-dev.fourdogspetsupplies.com/internal/etailpet/trigger?source=inventory` returns `200` with structured JSON
- `GET /data/freshness` reflects an updated `inventory.updated_at` timestamp after the trigger call
