# Story portal4-2-03: Update Helm chart: sidecar container + nginx proxy_pass for /api/*

**Epic:** 2 — Go Sidecar Foundation and Helm Wiring
**Status:** backlog
**Initiative:** portal4
**Date:** 2026-05-02
**Estimate:** M

---

## User Story

As a **developer deploying the portal pod**,
I want the Helm chart updated to include the sidecar as a second container and nginx configured to proxy `/api/*` to it,
So that the portal SPA can call sidecar routes via relative `/api/*` URLs without CORS or cross-origin issues.

---

## Context

**Repo:** `terminus-portal`
**Branch:** `portal4`
**Depends on:** portal4-2-02 merged (sidecar image must exist in GHCR before Helm can reference it)

The portal is deployed via a Helm chart co-located in the terminus-portal repo. This story adds a second container (`portal-sidecar`) to the pod spec and updates the nginx configuration to forward `/api/*` requests to `localhost:8080`.

**Architecture (ADR-1):**
- nginx `proxy_pass http://localhost:8080` for `location /api/ {}`
- Sidecar container reads env from the PAT Secret (wired in story 2-04) and SA token (mounted automatically)
- Sidecar does NOT expose a k8s Service — it's only accessible on localhost within the pod

---

## Acceptance Criteria

**Given** the Helm chart is rendered with `helm template`
**When** the output is inspected
**Then** the portal pod spec contains two containers: the existing nginx container and a new `portal-sidecar` container

**Given** the Helm chart is deployed to the dev cluster via ArgoCD
**When** the portal pod is running
**Then** `curl https://portal.trantor.internal/api/healthz` returns `{"status":"ok"}` (proxied through nginx to sidecar)

**Given** `helm template` is run against the chart
**Then** it renders without schema validation errors

**Given** the nginx config is updated
**When** a request for `/api/healthz` is made to nginx
**Then** nginx forwards it to `http://localhost:8080/api/healthz` and returns the sidecar response

**Given** the ArgoCD dev app syncs the updated chart
**Then** ArgoCD reports Healthy and Synced with no errors

---

## Dev Notes

- Check the existing Helm chart structure in the terminus-portal repo (likely `helm/` or `charts/` directory)
- Add to `values.yaml`:
  ```yaml
  sidecar:
    image: ghcr.io/electricm0nk/terminus-portal-sidecar
    tag: "latest"   # will be pinned by CI on each deploy
    port: 8080
  ```
- nginx config update — add to `nginx.conf` configmap:
  ```nginx
  location /api/ {
      proxy_pass http://localhost:8080;
      proxy_set_header Host $host;
      proxy_read_timeout 30s;
  }
  ```
- Sidecar container spec: `image`, `ports: [{containerPort: 8080}]`, no `readinessProbe` yet (added in a later sprint if needed), resource limits matching the other microservices in the cluster
- The PAT env var will be wired in story 2-04 — leave a placeholder `env: []` on the sidecar container for now
- Validate with `helm template . --debug | grep -A5 "portal-sidecar"` to confirm the container appears

---

## Definition of Done

- [ ] Code merged to feature branch (`portal4`)
- [ ] `helm template` renders without errors; sidecar container visible in output
- [ ] `go vet ./...` and `go build ./...` still passing in `sidecar/`
- [ ] Manual smoke test: portal deployed to dev; `curl https://portal.trantor.internal/api/healthz` returns `{"status":"ok"}`
- [ ] ArgoCD dev app syncs Healthy
- [ ] No secrets in Helm values or chart templates
