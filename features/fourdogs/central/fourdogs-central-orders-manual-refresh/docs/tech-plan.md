---
feature: fourdogs-central-orders-manual-refresh
doc_type: tech-plan
status: ready
goal: "Specify the backend route addition and frontend hook + component changes needed to wire the Sync Now button."
key_decisions:
  - "Route /v1/etailpet/trigger registered in the session-protected chi group — no new handler needed"
  - "Existing EtailpetTrigger handler called with token='' so bearer check is skipped; session middleware covers auth"
  - "Frontend uses React Query useMutation (not useQuery) to call the trigger endpoint"
  - "DataFreshnessPanel receives an optional onTrigger mutation prop or calls useEtailpetTrigger internally"
  - "Button visibility: computed from any(entry.stale) across inventory, sales, transactions"
  - "After mutation settles (success or partial), invalidate data-freshness query key"
updated_at: "2026-05-10"
depends_on: [fourdogs-etailpet-api-trigger-sync-fix]
---

# Tech Plan — fourdogs-central-orders-manual-refresh

## Target Repos

| Repo | Change |
|---|---|
| `fourdogs-central` (Go) | Add `POST /v1/etailpet/trigger` route in session-protected group |
| `fourdogs-central-ui` (React/Vite) | New hook `use_etailpet_trigger.ts` + updated `DataFreshnessPanel.tsx` |

---

## Backend Change — cmd/api/main.go

### Location

`cmd/api/main.go`, inside the session-protected `r.Group`:

```go
// Protected API routes -- session middleware validates DB-backed session
r.Group(func(r chi.Router) {
    r.Use(auth.SessionMiddleware(queries))
    // ... existing routes ...
    // UI-accessible trigger endpoint — auth covered by session middleware
    r.Post("/v1/etailpet/trigger", handler.EtailpetTrigger(etailpetClient, ""))
})
```

### Notes

- `handler.EtailpetTrigger` is the handler added by feature 1 (tsf-1-02)
- `token=""` skips the bearer token check — identical pattern to how `VelocityRiskRefresh` would behave with no token configured
- `etailpetClient` is the same `*etailpet.Client` constructed at server startup
- No new handler, no new interface — just a second route binding

### wiring etailpetClient at startup

If `fourdogs-central` doesn't already instantiate `etailpet.Client` in `cmd/api/main.go`
(it currently only exists in the trigger binaries), add:

```go
import "github.com/electricm0nk/fourdogs-central/internal/etailpet"
// in main():
etailpetClient := etailpet.NewClient(cfg.EtailpetClientID, cfg.EtailpetClientSecret, cfg.EtailpetTenantSchema)
```

Check `cmd/etailpet-trigger/main.go` to confirm the constructor signature.

---

## Frontend Change — Hook

### New file: `src/hooks/use_etailpet_trigger.ts`

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { api } from '@/lib/api'

export interface TriggerResult {
  status: string
  source: string
  triggered_at: string
  duration_ms: number
}

export function useEtailpetTrigger() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: () =>
      api.post<TriggerResult | TriggerResult[]>('/v1/etailpet/trigger?source=all', {}),
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['data-freshness'] })
    },
  })
}
```

**Response handling:**
- HTTP 200: `TriggerResult` (single source) or `TriggerResult[]` (source=all success) — resolves normally
- HTTP 207: `TriggerResult[]` with partial failures — **207 is in the 200–299 range so `res.ok === true`; `apiFetch` resolves (does not throw)**; the hook receives the array normally and `onSettled` fires
- HTTP 4xx/5xx: `ApiError` thrown by `apiFetch`; caught by `isError` from `useMutation`

---

## Frontend Change — DataFreshnessPanel.tsx

### Changes to existing component

1. Import `useEtailpetTrigger` hook
2. Derive `isStale = data && [data.inventory, data.sales, data.transactions].some(e => e?.stale)`
3. Add "Sync Now" button rendered when `isStale && !triggerMutation.isPending`
4. Show disabled "Syncing…" state when `triggerMutation.isPending`
5. Show inline error when `triggerMutation.isError`

```tsx
const triggerMutation = useEtailpetTrigger()
const isStale = data && [data.inventory, data.sales, data.transactions]
  .some((e) => e?.stale === true)
```

**Button placement:** below the existing `↻ Refresh` button in the panel header row, or as a second row within the panel (design discretion — keep it readable in both uiMode=dark and uiMode=light).

**Inline error:**
```tsx
{triggerMutation.isError && (
  <p className="text-sm font-mono text-red-400 mt-1">
    Sync failed — try again or wait for scheduled pull.
  </p>
)}
```

---

## Tests

### Backend

- No new handler test needed — `EtailpetTrigger` is tested in feature 1 (tsf-1-02)
- Integration smoke: `POST /v1/etailpet/trigger?source=all` returns 200 with session cookie

### Frontend — `src/__tests__/data-freshness.test.tsx`

Extend existing test file with:

| Test | Assertion |
|---|---|
| `renders Sync Now button when data is stale` | Button visible when any `stale: true` |
| `does not render Sync Now when all fresh` | Button absent when all `stale: false` |
| `shows Syncing state while mutation pending` | Button text changes, disabled |
| `shows error message on mutation failure` | Error text appears in panel |
| `invalidates data-freshness on success` | `queryClient.invalidateQueries` called |

---

## Response Shape (from feature 1)

```json
// source=all success (HTTP 200)
[
  { "status": "ok", "source": "inventory", "triggered_at": "2026-05-10T21:00:00Z", "duration_ms": 450 },
  { "status": "ok", "source": "sales", "triggered_at": "2026-05-10T21:00:01Z", "duration_ms": 320 },
  { "status": "ok", "source": "transactions", "triggered_at": "2026-05-10T21:00:02Z", "duration_ms": 180 }
]

// source=all partial failure (HTTP 207)
[
  { "status": "ok", "source": "inventory", "triggered_at": "2026-05-10T21:00:00Z", "duration_ms": 450 },
  { "status": "error", "source": "sales", "error": "etailpet api timeout" },
  { "status": "ok", "source": "transactions", "triggered_at": "2026-05-10T21:00:02Z", "duration_ms": 180 }
]
```

The frontend treats 200 and 207 identically — both trigger a freshness refetch.
The `ApiError` thrown by `apiFetch` on 4xx/5xx is caught by `isError`.

---

## Open Questions

1. **etailpetClient startup**: Does `cmd/api/main.go` currently instantiate `etailpet.Client`? Verify in tsf-1-02 — if not, this feature adds the startup wiring. Also verify whether `cfg.EtailpetClientID`, `cfg.EtailpetClientSecret`, and `cfg.EtailpetTenantSchema` exist in `internal/config/config.go`; if absent, they must be added (and the corresponding Kubernetes secret must be verified).
2. **button placement in dark vs light mode**: DataFreshnessPanel uses `uiMode` prop but dark mode styling is dominant. Match the existing `↻ Refresh` button class string exactly.
3. **207 re-trigger cooldown**: After any trigger attempt (200 or 207), consider disabling the Sync Now button for 30 seconds client-side to prevent rapid re-triggering of already-succeeded sources. Implementer discretion — document the choice in the story completion note.
