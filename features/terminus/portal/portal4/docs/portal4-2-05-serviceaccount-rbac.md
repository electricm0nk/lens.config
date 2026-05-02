# Story portal4-2-05: Create ServiceAccount and RBAC for k8s pod listing

**Epic:** 2 — Go Sidecar Foundation and Helm Wiring
**Status:** backlog
**Initiative:** portal4
**Date:** 2026-05-02
**Estimate:** M

---

## User Story

As a **developer implementing the Pods tab**,
I want the sidecar to run with a ServiceAccount bound to a Role allowing pod listing across namespaces,
So that Sprint 4 can call `/api/k8s/pods` and get live pod status without elevated privileges.

---

## Context

**Repo:** `terminus-portal`
**Branch:** `portal4`
**Depends on:** portal4-2-03 merged (Helm chart with sidecar container must exist)

**Architecture (ADR-3 in tech-plan):**
- ServiceAccount `portal-sidecar-sa` in `terminus-portal` namespace
- ClusterRole `portal-sidecar-pods-reader` — verbs: `get`, `list`, `watch`; resources: `pods`; no other resources
- ClusterRoleBinding binding `portal-sidecar-sa` to `portal-sidecar-pods-reader`
- Sidecar Deployment uses `serviceAccountName: portal-sidecar-sa`
- SA token auto-mounted at standard `/var/run/secrets/kubernetes.io/serviceaccount/token` path — used by sidecar's k8s client

**Scope note:** ClusterRole is required (not just Role) because the sidecar needs to list pods in all namespaces for the Pods tab. The role grants read-only access to pods only — not Secrets, ConfigMaps, Deployments, or any other resource.

---

## Acceptance Criteria

**Given** the Helm chart is deployed
**When** `kubectl get clusterrolebinding portal-sidecar-pods-reader-binding` is run
**Then** it exists and is bound to `portal-sidecar-sa` in `terminus-portal` namespace

**Given** the sidecar pod is running with `portal-sidecar-sa`
**When** the following kubectl command is run from inside the pod:
`kubectl auth can-i list pods --all-namespaces --as=system:serviceaccount:terminus-portal:portal-sidecar-sa`
**Then** the output is `yes`

**Given** the RBAC is applied
**When** `kubectl auth can-i list secrets --as=system:serviceaccount:terminus-portal:portal-sidecar-sa` is run
**Then** the output is `no` (confirms minimal permissions)

**Given** `helm template` is run
**Then** ServiceAccount, ClusterRole, and ClusterRoleBinding all appear in rendered output

---

## Dev Notes

- Add to Helm chart (can be in `templates/serviceaccount.yaml` or a combined RBAC template):
  ```yaml
  # ServiceAccount
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: portal-sidecar-sa
    namespace: terminus-portal

  # ClusterRole
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    name: portal-sidecar-pods-reader
  rules:
    - apiGroups: [""]
      resources: ["pods"]
      verbs: ["get", "list", "watch"]

  # ClusterRoleBinding
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: portal-sidecar-pods-reader-binding
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: portal-sidecar-pods-reader
  subjects:
    - kind: ServiceAccount
      name: portal-sidecar-sa
      namespace: terminus-portal
  ```
- Update sidecar Deployment spec: `serviceAccountName: portal-sidecar-sa`
- The sidecar's k8s client (implemented in Sprint 4 stories) will use the in-cluster config auto-loaded from the SA token mount

---

## Definition of Done

- [ ] Code merged to feature branch (`portal4`)
- [ ] `helm template` renders SA, ClusterRole, ClusterRoleBinding without errors
- [ ] `kubectl auth can-i list pods --all-namespaces` returns `yes` for `portal-sidecar-sa`
- [ ] `kubectl auth can-i list secrets` returns `no` for `portal-sidecar-sa`
- [ ] Sidecar pod runs under `portal-sidecar-sa` (confirmed via `kubectl get pod -o yaml | grep serviceAccountName`)
- [ ] ArgoCD dev app syncs Healthy
- [ ] No secrets committed to source code
