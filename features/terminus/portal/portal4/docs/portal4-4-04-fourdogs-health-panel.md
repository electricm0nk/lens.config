# Story portal4-4-04: Build FourDogsHealthPanel.jsx and wire per-microservice health signals

**Epic:** 4 — Deep Health Signals and Prometheus Metrics
**Status:** backlog
**Initiative:** portal4
**Date:** 2026-05-02
**Estimate:** L

---

## User Story

As an **operator monitoring the Fourdogs stack**,
I want a health panel on the Overview tab showing the live status of all 5 Fourdogs microservices,
So that I can identify degraded services at a glance without opening multiple tabs or running kubectl commands.

---

## Context

**Repo:** `terminus-portal` (React/Vite SPA)
**Branch:** `portal4`
**Depends on:** portal4-4-01 merged (sidecar `/api/k8s/pods/all` must exist for trigger worker signals)

**Conditional dependency:** The emailfetcher health signal (`/v1/health/emailfetcher` via `fourdogs-central`) requires a new endpoint on fourdogs-central that may not exist yet. If this endpoint is absent by the time this story is executed, the emailfetcher cell shows "no signal" gracefully and the story is marked partial-complete until fourdogs-central is updated.

**Health signal sources per microservice:**

| Service | Signal Source | Notes |
|---------|--------------|-------|
| fourdogs-central UI | Sidecar proxies `GET /health` to `fourdogs-central` internal service → check `component: "central-ui"` | Internal k8s service call from sidecar |
| fourdogs-central API | Sidecar proxies `GET /v1/health` to `fourdogs-central` internal service | Internal k8s service call from sidecar |
| emailfetcher | Sidecar proxies `GET /v1/health/emailfetcher` to `fourdogs-central` internal service | **Conditional** — graceful "no signal" if absent |
| etailpet-trigger | Pod phase + restart count from `/api/k8s/pods/all` (namespace: `fourdogs-etailpet-trigger`) | No k8s Service exists; pod status is the health signal |
| etailpet-sales-trigger | Pod phase + restart count from `/api/k8s/pods/all` (namespace: `fourdogs-etailpet-sales-trigger`) | No k8s Service exists; pod status is the health signal |

All fourdogs-central calls route through a new sidecar route `GET /api/fourdogs/health` which aggregates all three fourdogs-central health signals.

---

## Acceptance Criteria

**Given** the Overview tab loads
**When** FourDogsHealthPanel renders
**Then** five health rows are visible: Central UI, Central API, Emailfetcher, ETailPet Trigger, ETailPet Sales Trigger

**Given** fourdogs-central is running and `/v1/health` returns 200
**When** FourDogsHealthPanel fetches data
**Then** Central API row shows green "UP" status

**Given** fourdogs-central `/v1/health/emailfetcher` does not exist (returns 404)
**When** FourDogsHealthPanel fetches emailfetcher signal
**Then** emailfetcher row shows "no signal" (grey) — not an error state, not blank

**Given** the etailpet-trigger pod is in `Running` phase with 0 restarts
**When** FourDogsHealthPanel reads its pod status from `/api/k8s/pods/all`
**Then** the ETailPet Trigger row shows green "Running" with restart count 0

**Given** the etailpet-trigger pod has >5 restarts
**When** FourDogsHealthPanel reads its pod status
**Then** the restart count is highlighted in red with an alert indicator

**Given** `npm test` and `npm run build` are run
**Then** both pass

---

## Dev Notes

- Create `src/components/FourDogsHealthPanel.jsx`
- New sidecar route to add: `GET /api/fourdogs/health` — sidecar calls fourdogs-central internal service URL (`http://fourdogs-central.fourdogs-central.svc.cluster.local` or the appropriate in-cluster DNS) and returns aggregated health JSON
  - This requires adding a new handler `sidecar/handlers/fourdogs.go`
  - fourdogs-central internal service URL: confirm from the existing fourdogs-central Helm chart or service manifest
- Etailpet trigger signal: the React component should filter the `/api/k8s/pods/all` response (already fetched by PodsTab) for pods in `fourdogs-etailpet-trigger` and `fourdogs-etailpet-sales-trigger` namespaces — reuse the pods data if already in component state, or call the route again
- Error handling: each health row is independent — one failed fetch should not blank out other rows
- Partial-complete acceptance: if fourdogs-central `/v1/health/emailfetcher` does not exist, mark story done with a `TODO` comment in the component and the emailfetcher cell showing "no signal"

---

## Definition of Done

- [ ] Code merged to feature branch (`portal4`)
- [ ] Five health rows visible in FourDogsHealthPanel on Overview tab
- [ ] Central API and Central UI rows show live status from fourdogs-central
- [ ] emailfetcher row shows "no signal" gracefully if endpoint absent (not an error)
- [ ] etailpet-trigger and etailpet-sales-trigger rows show pod phase + restart count
- [ ] All health row failures are isolated (one failure doesn't blank others)
- [ ] `npm test` passes; `npm run build` passes with no TypeScript/ESLint errors
- [ ] No secrets committed to source code
