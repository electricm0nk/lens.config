---
feature: fourdogs-etailpet-api-trigger-sync-fix
doc_type: tech-plan
status: draft
goal: >
  Add POST /internal/etailpet/trigger to the fourdogs-central API server using
  the existing etailpet.NewClient and risk_refresh handler as the implementation model.
key_decisions:
  - New handler file internal/handler/etailpet_trigger.go mirrors risk_refresh.go structure
  - Uses etailpet.NewClient(cfg, q).Trigger() — same call the trigger binary makes
  - source param selects which ingestion cycle to run: inventory | sales | transactions | all
  - No new DB schema required — trigger run timestamp is already recorded by the etailpet client
  - Route registered as POST /internal/etailpet/trigger with bearer token auth (same as VelocityRiskRefresh — token passed via Authorization: Bearer header)
  - Handler returns JSON: {status, source, triggered_at, duration_ms, error} — rows_affected omitted (etailpet Trigger() is fire-and-forget; no row count is returned by the client)
open_questions: []
depends_on: []
blocks:
  - fourdogs-central-orders-manual-refresh
updated_at: "2026-05-10"
---

# Tech Plan — fourdogs-etailpet-api-trigger-sync-fix

## Current State

`fourdogs-central` is a Go monorepo that builds multiple binaries from `cmd/`:
- `cmd/api` — the HTTP API server (target for this change)
- `cmd/etailpet-trigger` — scheduled inventory ingestion (runs `etailpet.NewClient().Trigger()`)
- `cmd/etailpet-sales-fetcher` — scheduled sales ingestion
- `cmd/etailpet-catalog-trigger` — scheduled catalog ingestion

All trigger binaries already call into `internal/etailpet` package functions. The API server
does not currently expose any endpoint to invoke these cycles on demand.

`internal/handler/risk_refresh.go` is the exact implementation model: a `VelocityRiskRefresh`
handler that recomputes risk scores on demand via `POST /internal/velocity-risk/refresh`.

`internal/handler/data_freshness.go` already queries `GetLatestTriggerRun(ctx, source)` for
`"sales"` and `"transactions"`, and `GetLatestInventoryUpdate` for inventory — so the freshness
read path is already in place.

---

## Implementation Plan

### New file: `internal/handler/etailpet_trigger.go`

```go
// EtailpetTrigger handles POST /internal/etailpet/trigger?source={inventory|sales|transactions|all}
// It invokes the etailpet ingest cycle on demand for the requested source(s).
// Bearer token auth required — Authorization: Bearer {token}, same as /internal/velocity-risk/refresh.
```

**Interface:**
```go
type EtailpetTriggerClient interface {
    TriggerInventory(ctx context.Context) (int, error)
    TriggerSales(ctx context.Context) (int, error)
    TriggerTransactions(ctx context.Context) (int, error)
}
```

**Response shape:**
```json
{
  "status": "ok",
  "source": "inventory",
  "triggered_at": "2026-05-10T14:23:01Z",
  "duration_ms": 1840
}
```

**Error response:**
```json
{
  "status": "error",
  "source": "sales",
  "triggered_at": "2026-05-10T14:23:01Z",
  "error": "etailpet auth failure: 401"
}
```

**source=all failure behavior:** Run sources sequentially (inventory → sales → transactions). On first failure, collect the error and continue remaining sources. Return HTTP 207 with per-source results array:
```json
[
  {"source": "inventory", "status": "ok", "triggered_at": "...", "duration_ms": 940},
  {"source": "sales", "status": "error", "error": "...", "triggered_at": "..."}
]
```

### Route registration

In the API server's router setup (same location as `/internal/velocity-risk/refresh`):

```
POST /internal/etailpet/trigger  →  handler.EtailpetTrigger(...)
```

### etailpet.Client extension

`etailpet.NewClient` already has `Trigger()`. If the existing `Trigger()` method covers
inventory only, add `TriggerSales()` and `TriggerTransactions()` methods using the same
auth/call pattern. Inspect `cmd/etailpet-sales-fetcher/main.go` to confirm the call
shape before adding.

---

## Data Flow

```
POST /internal/etailpet/trigger?source=inventory
  └─► handler.EtailpetTrigger
        ├─► etailpet.NewClient(cfg, q).TriggerInventory(ctx)
        │     └─► etailpet API fetch → upsert to DB
        └─► return {status, source, rows_affected, duration_ms}
```

The timestamp recorded by the existing trigger logic is what `GetLatestInventoryUpdate`
/ `GetLatestTriggerRun` already reads, so `data/freshness` will immediately reflect the
updated state after a successful trigger call.

---

## Test Plan

- Unit: `TestEtailpetTrigger_Inventory`, `_Sales`, `_Transactions` — mock client, assert response shape and HTTP 200
- Unit: `TestEtailpetTrigger_All_AllSucceed` — assert 200, 3-item array
- Unit: `TestEtailpetTrigger_All_PartialFailure` — first source fails, assert 207, per-source errors
- Unit: `TestEtailpetTrigger_InvalidSource` — assert 400
- Unit: `TestEtailpetTrigger_ClientError` — mock client returns error, assert 500 + structured JSON
- Unit: `TestEtailpetTrigger_MissingToken` — no Authorization header, assert 401
- Integration: not required for this change (existing trigger integration tests cover the client)

---

## Deployment

No Helm or ArgoCD changes required. The `cmd/api` binary is the only deployment artifact
affected. It is deployed via the existing `fourdogs-central` Helm chart and ArgoCD Application.
Merge to `develop` → ArgoCD syncs dev. Merge to `main` → ArgoCD syncs prod.
