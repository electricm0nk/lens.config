# Story portal4-4-05: Wire metrics.js to live Prometheus endpoints in MetricsPanel

**Epic:** 4 — Deep Health Signals and Prometheus Metrics
**Status:** backlog
**Initiative:** portal4
**Date:** 2026-05-02
**Estimate:** S

---

## User Story

As an **operator viewing the portal Overview tab**,
I want the MetricsPanel to display live CPU load, memory usage, and pods-running counts from Prometheus,
So that I have a real-time cluster health summary without leaving the portal.

---

## Context

**Repo:** `terminus-portal` (React/Vite SPA)
**Branch:** `portal4`
**Depends on:** portal4-4-03 merged (sidecar `/api/metrics/query` route must exist)

The portal currently has a MetricsPanel component (from portal3 or earlier) that may use static data, mocked data, or a previous metrics source. This story replaces its data source with live Prometheus queries via the sidecar proxy.

---

## Acceptance Criteria

**Given** the Overview tab loads and Prometheus is reachable
**When** MetricsPanel fetches data
**Then** CPU load, memory usage, and pods-running count all display real values (not 0 or placeholder)

**Given** the metrics queries are:
- CPU load: `100 - (avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)` (overall cluster CPU %)
- Memory usage: `(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100` (overall memory %)
- Pods running: `count(kube_pod_status_phase{phase="Running"})` (total running pods)
**Then** the panel displays these three metrics in the existing MetricsPanel layout

**Given** the Prometheus query returns no data (e.g. metric doesn't exist)
**When** MetricsPanel renders
**Then** the cell shows "N/A" (not blank, not a crash)

**Given** `npm test` and `npm run build` are run
**Then** both pass

---

## Dev Notes

- Find the existing `src/config/metrics.js` (or equivalent) and update the query definitions to the three PromQL queries above
- The MetricsPanel component likely already has `useEffect` + `fetch` — update the URL to call `GET /api/metrics/query?query=<promql>`
- Encode the PromQL query with `encodeURIComponent` when building the URL
- For a single current value (not a time range), use the Prometheus instant query endpoint: the sidecar route may need to support both `GET /api/metrics/query` (instant: `/api/v1/query`) and `GET /api/metrics/query_range` (range: `/api/v1/query_range`) — check story 4-03 implementation and adjust if needed
- If MetricsPanel doesn't yet exist (portal3 may have introduced it), create a minimal `<MetricsPanel>` with three cells: CPU Load, Memory, Pods Running

---

## Definition of Done

- [ ] Code merged to feature branch (`portal4`)
- [ ] CPU load, memory usage, pods-running show live Prometheus values on Overview tab
- [ ] "N/A" shown gracefully when a metric query returns no data
- [ ] `npm test` passes; `npm run build` passes with no TypeScript/ESLint errors
- [ ] No secrets committed to source code
- [ ] ArgoCD dev app syncs Healthy after final deploy
