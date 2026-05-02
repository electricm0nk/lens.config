# Story portal4-4-01: Implement /api/k8s/pods and /api/k8s/pods/all sidecar routes

**Epic:** 4 — Deep Health Signals and Prometheus Metrics
**Status:** backlog
**Initiative:** portal4
**Date:** 2026-05-02
**Estimate:** M

---

## User Story

As a **developer building the Pods tab and FourDogsHealthPanel**,
I want sidecar routes that return pod status for the current namespace and all namespaces,
So that the React SPA can display live pod health without direct access to the k8s API.

---

## Context

**Repo:** `terminus-portal` (sidecar Go code)
**Branch:** `portal4`
**Depends on:** portal4-2-05 merged (ServiceAccount and RBAC must be wired; sidecar must have `list pods` permission)

**Architecture (ADR-3 in tech-plan):**
- Route 1: `GET /api/k8s/pods` — pods in the sidecar's own namespace (`terminus-portal`)
- Route 2: `GET /api/k8s/pods/all` — pods in all namespaces (uses the same ClusterRole — list pods across cluster)
- Sidecar uses in-cluster config (SA token at standard mount path) to call the k8s API
- Returns pod summary; no raw k8s API response forwarded (sanitise fields)

**Response shape per pod:**
```json
{
  "name": "fourdogs-central-abc123",
  "namespace": "fourdogs-central",
  "phase": "Running",
  "ready": true,
  "restartCount": 0,
  "age": "3 days ago",
  "nodeName": "k3s-node-1"
}
```

---

## Acceptance Criteria

**Given** the sidecar is running with `portal-sidecar-sa`
**When** `GET /api/k8s/pods/all` is called
**Then** the response is HTTP 200 with a JSON array of pod summaries from all namespaces

**Given** a pod is in `Running` phase with 0 restarts
**When** its entry is returned
**Then** `"phase": "Running"`, `"ready": true`, `"restartCount": 0`

**Given** a pod has had restarts
**When** its entry is returned
**Then** `"restartCount"` reflects the actual restart count from the pod status

**Given** `GET /api/k8s/pods` is called (no `/all`)
**Then** only pods in the `terminus-portal` namespace are returned

**Given** `go vet ./...` and `go build ./...` are run
**Then** both pass

---

## Dev Notes

- Add `sidecar/handlers/k8s.go`
- Use `k8s.io/client-go` for in-cluster k8s API access — add as a Go module dependency
- In-cluster config: `rest.InClusterConfig()` auto-reads the SA token and CA cert from standard mount paths
- For `/api/k8s/pods/all`: `clientset.CoreV1().Pods("")` (empty string = all namespaces)
- For `/api/k8s/pods`: `clientset.CoreV1().Pods("terminus-portal")`
- Map pod status to the response shape — extract: `pod.Name`, `pod.Namespace`, `pod.Status.Phase`, ready condition from `pod.Status.Conditions`, sum of container restart counts, `pod.CreationTimestamp` for age, `pod.Spec.NodeName`
- Do not forward raw pod spec — only the summary fields needed for display
- Register routes in `main.go`: `GET /api/k8s/pods` and `GET /api/k8s/pods/all`
- Note: This route is also the health signal source for etailpet trigger workers (no HTTP Services exist for those workers; the Pods tab `/api/k8s/pods/all` response is the health signal)

---

## Definition of Done

- [ ] Code merged to feature branch (`portal4`)
- [ ] `go vet ./...` and `go build ./...` passing
- [ ] `GET /api/k8s/pods/all` returns pod summaries from all namespaces in dev cluster
- [ ] `GET /api/k8s/pods` returns only `terminus-portal` namespace pods
- [ ] Restart counts and phase are accurate
- [ ] No raw k8s API response forwarded (only summary fields)
- [ ] No secrets committed to source code
- [ ] ArgoCD dev app syncs Healthy after deploy
