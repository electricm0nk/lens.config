# Story portal4-4-02: Build Pods tab component (PodsTab.jsx)

**Epic:** 4 — Deep Health Signals and Prometheus Metrics
**Status:** backlog
**Initiative:** portal4
**Date:** 2026-05-02
**Estimate:** M

---

## User Story

As an **operator**,
I want a Pods tab that shows all cluster pods grouped by namespace with status, restart count, and age,
So that I can quickly assess cluster health from the portal without logging into the cluster directly.

---

## Context

**Repo:** `terminus-portal` (React/Vite SPA)
**Branch:** `portal4`
**Depends on:**
- portal4-3-01 merged (Pods tab must render in the tab navigation)
- portal4-4-01 merged (sidecar `/api/k8s/pods/all` route must exist)

---

## Acceptance Criteria

**Given** the Pods tab is selected
**When** the component mounts
**Then** it calls `GET /api/k8s/pods/all` and begins loading

**Given** data has loaded
**When** the component renders
**Then** pods are grouped by namespace; each namespace is a collapsible section (or a visible header group)

**Given** a pod entry renders
**Then** it shows: pod name, ready status (✓ or ✗), phase, restart count, age

**Given** a pod has restarts > 0
**When** the restart count renders
**Then** it is highlighted (e.g. yellow for 1–5, red for >5) to draw operator attention

**Given** data is loading
**When** the component is mounted
**Then** a loading state (skeleton or spinner) is shown

**Given** the fetch fails (network error or sidecar error)
**When** the error is received
**Then** an error message is displayed (not a crash)

**Given** `npm test` and `npm run build` are run
**Then** both pass

---

## Dev Notes

- Create `src/components/PodsTab.jsx`
- Group pods by `namespace` — use `Array.reduce` to build a `{ [namespace]: pod[] }` map
- Sort namespaces alphabetically; sort pods within a namespace by name
- Phase pill colours: `Running` → green, `Pending` → yellow, `Failed` → red, other → grey
- Restart count highlight: 0 → no highlight, 1–5 → yellow, >5 → red
- Auto-refresh: optionally add a 30s refresh timer (`setInterval` in `useEffect`) — nice-to-have but not required for DoD
- Reuse the same style tokens/variables as the existing portal theme (check `src/App.css` or equivalent)

---

## Definition of Done

- [ ] Code merged to feature branch (`portal4`)
- [ ] Pods grouped by namespace with phase, ready status, restart count, age
- [ ] Restart count > 5 highlighted in red
- [ ] Loading and error states handled
- [ ] `npm test` passes; `npm run build` passes with no TypeScript/ESLint errors
- [ ] No secrets committed to source code
