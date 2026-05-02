---
feature: terminus-portal-portal4
doc_type: tech-plan
status: draft
goal: "Introduce a Go backend sidecar into the portal pod, restructure the React SPA into tabbed navigation (Overview, Release Pipeline, Pods), wire Fourdogs per-microservice health signals, proxy k8s and Prometheus APIs through the sidecar, and fix all visual regressions from portal3."
key_decisions:
  - Go sidecar (single static binary, distroless image) deployed in portal pod alongside nginx
  - Nginx proxies /api/* to sidecar on localhost:8080; SPA only ever calls its own origin
  - PAT from Vault ESO (ExternalSecret targeting terminus/github::Portal_PAT)
  - k8s read access via projected ServiceAccount token (in-cluster client); RBAC read-only on pods/namespaces
  - Prometheus queries via sidecar HTTP proxy to https://prometheus.trantor.internal
  - React tab navigation via useState — no React Router; URL hash optional enhancement
  - Release Pipeline config: per-repo branches array with dev/prod labels; git tag resolved via Refs API
  - emailfetcher health via fourdogs-central health stub endpoint (not direct DB from portal sidecar)
  - Version watermark: git tag injected at build time via Vite VITE_APP_VERSION env var
open_questions: []
depends_on: []
blocks: []
updated_at: "2026-05-02T00:00:00Z"
---

# Tech Plan: terminus-portal-portal4

## Technical Summary

Portal4 introduces a small Go binary deployed as a second container in the existing `terminus-portal` pod. It runs on `localhost:8080` and is never exposed outside the pod; nginx forwards all `/api/*` path traffic to it. The sidecar handles three concerns that cannot be satisfied from a static browser SPA: injecting a GitHub PAT into API calls to private repos, querying the k8s API with a ServiceAccount token, and proxying PromQL requests to the internal Prometheus instance. The React SPA is restructured from a single-page grid layout into a three-tab design: Overview (service health cards + quick metrics + service links), Release Pipeline (per-repo branch state), and Pods (live k8s pod health by namespace). Existing visual regressions (ArgoCD/Semaphore icon slugs, Grafana card overflow) are fixed in the same pass. A Helm chart update wires the sidecar container, RBAC objects, and ESO ExternalSecret.

## Architecture Overview

```
Browser
  └─ HTTPS → terminus-portal nginx (port 443, cert-manager TLS)
                ├─ static assets (/, /assets/*)
                └─ proxy_pass localhost:8080 (/api/*)
                     └─ portal-sidecar (Go, port 8080)
                           ├─ /api/github/* → api.github.com (PAT injected from env)
                           ├─ /api/k8s/*    → k8s API server (in-cluster SA token)
                           └─ /api/metrics/* → https://prometheus.trantor.internal (PromQL)
```

**Components affected:**
- `terminus-portal` Helm chart — new container, ServiceAccount, RBAC, ExternalSecret
- `terminus-portal` React SPA — tab nav, three tab components, updated config files
- `services.js` — add Temporal, fix icon slugs, add devUrl for Fourdogs
- `githubActions.js` → replaced by `releasePipeline.js` config
- `metrics.js` — PromQL query strings wired, `wired: true`
- New Go module: `portal-sidecar` (lives at `TargetProjects/terminus/portal/terminus-portal-sidecar/` or `cmd/sidecar/` in the portal repo)

**Integration points:**
- Vault + ESO: existing pattern used by fourdogs services; ClusterSecretStore already deployed
- GitHub API v3 REST: `/repos/{owner}/{repo}/actions/runs`, `/repos/{owner}/{repo}/git/refs`
- k8s API: `GET /api/v1/pods?fieldSelector=namespace=x` or namespace-scoped `GET /api/v1/namespaces/{ns}/pods`
- Prometheus HTTP API: `GET /api/v1/query?query=<PromQL>` at `https://prometheus.trantor.internal`
- fourdogs-central internal service endpoint for deep health signals

## Design Decisions (ADRs)

### ADR-1: Go sidecar over nginx lua / BFF service

**Decision:** Embed a small Go binary as a second container in the portal pod rather than a standalone BFF deployment or nginx Lua scripting.

**Rationale:** A standalone BFF service doubles infrastructure overhead (separate deployment, service, ingress, TLS) for what is single-tenant tooling. nginx Lua cannot easily make authenticated HTTP calls to multiple external services without becoming unmaintainable. The Go sidecar is a single static binary, adds ~5 MB to the pod, and collocates with nginx so no network hop is needed.

**Alternatives rejected:**
- *Standalone BFF deployment* — overhead unjustified for single operator tool; adds ArgoCD app, service, TLS
- *nginx Lua module* — too brittle; not idiomatic; nginx image must be rebuilt with Lua module support
- *nginx proxy_pass with HTTP header injection* — cannot inject a secret from k8s env var dynamically in nginx without Lua

---

### ADR-2: PAT via Vault ESO ExternalSecret

**Decision:** GitHub PAT sourced from Vault secret `terminus/github`, key `Portal_PAT`, synced to k8s Secret `portal-github-pat` via ExternalSecret, mounted as env var in the sidecar container.

**Rationale:** Consistent with fourdogs services pattern. ESO keeps the secret out of git and Helm values; Vault rotation propagates automatically on pod restart.

**Security constraint:** The sidecar must never log the PAT value, request `Authorization` headers, or environment variable contents in any code path. Structured logging must explicitly omit auth headers. Panic handlers must not dump env vars.

**Alternatives rejected:**
- *Helm values / sealed secret* — doesn't rotate without a re-deploy; Vault is already used for this cluster
- *Direct Vault agent sidecar* — adds Vault agent container on top of sidecar; ESO is already in the cluster and simpler

---

### ADR-3: k8s read access via projected ServiceAccount

**Decision:** Create a ServiceAccount `portal-sidecar-sa` bound to a ClusterRole (or Role per namespace) with `get`/`list`/`watch` on `pods` resource only. Sidecar uses in-cluster token projection.

**Rationale:** Minimal RBAC footprint. Projected token auto-rotates; no manual secret management. List pods only — no write access, no secrets access.

**Alternatives rejected:**
- *kubectl binary in sidecar* — heavyweight; `kubectl exec` surface; not needed
- *Aggregate pod counts from Prometheus kube-state-metrics* — possible but k8s API is more authoritative and simpler for pod-by-namespace listing; both methods used (Prometheus for metrics, k8s API for pod list)

---

### ADR-4: Prometheus proxy over direct browser calls

**Decision:** Sidecar proxies PromQL queries to `https://prometheus.trantor.internal`. Browser never calls Prometheus directly.

**Rationale:** `prometheus.trantor.internal` is an internal DNS name not reachable from the browser. Sidecar has cluster-internal DNS resolution and can reach it.

**Alternatives rejected:**
- *Grafana embedded panels* — requires auth setup; different UX pattern; not needed for 3-4 metric cards
- *Prometheus push gateway* — wrong direction; Prometheus is pull-based and we are querying it, not pushing

---

### ADR-5: emailfetcher health via fourdogs-central API stub

**Decision:** fourdogs-central API will expose a `/v1/health/emailfetcher` endpoint (or fold into existing `/v1/health`) that reads from `import_audit` and returns `{ last_success: ISO, errors_24h: int, status: "ok"|"stale"|"error" }`. The portal sidecar calls this endpoint via the internal fourdogs-central service.

**Rationale:** Direct DB queries from the portal sidecar would require portal to hold DB credentials — wrong trust boundary. fourdogs-central already owns the DB connection pool and the `import_audit` table. A health endpoint is a thin wrapper on an existing query.

**Alternatives rejected:**
- *Portal sidecar queries DB directly* — requires DB credentials in portal namespace; violates service ownership
- *Prometheus metric scraped from emailfetcher* — emailfetcher is a worker binary; instrumenting it adds scope; health endpoint in central API is simpler

---

### ADR-6: Tab navigation via React useState (no React Router)

**Decision:** Tab switching is managed by a single `activeTab` state variable in App.jsx. No URL routing.

**Rationale:** The portal is a single-page operational tool, not a multi-view app that needs deep linking. Adding React Router for 3 tabs introduces unnecessary dependency and build complexity.

**Alternatives rejected:**
- *React Router v6* — correct for multi-view apps; overkill here; no deep-link requirement stated
- *URL hash (#overview, #pipeline)* — enhancement only; deferred to later iteration

---

### ADR-7: Release Pipeline config replaces githubActions.js

**Decision:** `src/config/githubActions.js` is replaced by `src/config/releasePipeline.js`. The new config defines repos with a `branches` array, each with `{ name, label, type: "dev"|"prod" }`. The Release Pipeline tab component fetches latest run + git ref per repo/branch from the sidecar.

**Rationale:** The old config file structure could not express per-branch tracking. A full replacement is cleaner than a backward-compatible extension for a config that had no production traffic.

**Breaking change flag:** No external consumers. Internal rename only.

---

## API Contracts

### Sidecar internal API (nginx-proxied, not externally routable)

All endpoints are prefixed `/api/` and proxied by nginx. They are called by the SPA's own origin — no CORS required (same origin).

#### `/api/github/repos/:owner/:repo/branches/:branch`
```
GET /api/github/repos/electricm0nk/terminus-portal/branches/develop
Response 200:
{
  "name": "develop",
  "sha": "abc1234",
  "tag": "v1.4.2",          // latest tag reachable from this SHA, or null
  "message": "...",          // commit message first line
  "committed_at": "ISO",
  "workflow_runs": [         // last 3 runs on this branch
    {
      "id": 12345678,
      "name": "CI",
      "status": "completed",
      "conclusion": "success",
      "run_at": "ISO"
    }
  ]
}
```
Breaking change: false (new endpoint)

#### `/api/github/repos/:owner/:repo/actions/runs`
```
GET /api/github/repos/electricm0nk/terminus-portal/actions/runs?branch=develop&per_page=5
Response 200: pass-through of GitHub API response (PAT injected server-side)
```
Breaking change: false

#### `/api/k8s/pods`
```
GET /api/k8s/pods?namespace=fourdogs
Response 200:
{
  "namespace": "fourdogs",
  "pods": [
    {
      "name": "fourdogs-central-abc123",
      "ready": true,
      "restarts": 0,
      "age_seconds": 86400,
      "phase": "Running"
    }
  ]
}
```
Breaking change: false (new endpoint)

#### `/api/k8s/pods/all`
```
GET /api/k8s/pods/all
Response 200:
{
  "namespaces": [
    { "namespace": "fourdogs", "pods": [...] },
    { "namespace": "terminus-portal", "pods": [...] },
    ...
  ]
}
```
Breaking change: false

#### `/api/metrics/query`
```
GET /api/metrics/query?query=<PromQL>&step=60s
Response 200: pass-through of Prometheus HTTP API response
```
Breaking change: false

### Changes to existing SPA API calls

| Old call | Status | Replacement |
|----------|--------|-------------|
| `https://api.github.com/repos/...` (no auth) | Removed | `/api/github/repos/...` (PAT injected) |
| Individual service health checks (direct fetch) | Retained | Unchanged — service health checks remain client-side direct fetches |

---

## Data Model Changes

None. Portal4 adds no persistent storage.

The only data-adjacent change: `fourdogs-central` adds a `/v1/health/emailfetcher` sub-endpoint reading `import_audit`. This is a fourdogs-central change tracked in a separate story, not a data model change for portal4.

---

## Dependencies

| Dependency | Version / Constraint | Notes |
|------------|---------------------|-------|
| Go | 1.22+ | Sidecar build; distroless image |
| ESO (External Secrets Operator) | cluster-installed | ExternalSecret for PAT |
| Vault | `terminus/github` path must exist with `Portal_PAT` key | |
| cert-manager | cluster-installed | Existing TLS cert unchanged |
| Prometheus | `https://prometheus.trantor.internal` reachable from pod | |
| fourdogs-central internal service | `fourdogs-central.fourdogs.svc.cluster.local` or equivalent | For emailfetcher health sub-endpoint |
| github.com/electricm0nk PAT | read:repo, workflow scopes | Stored in Vault; rotated manually |

---

## Rollout Strategy

**Approach:** Direct deploy to dev (`terminus-portal-portal4` branch → develop → ArgoCD sync).

**Phased order:**
1. Visual fixes only (icon slugs, Grafana overflow, add Temporal card) — commit and verify in dev
2. Go sidecar scaffold (empty HTTP handler, health endpoint at `/api/healthz`) — verify sidecar starts in pod
3. GitHub API proxy — wire Release Pipeline tab; verify data visible for all repos
4. k8s API — wire Pods tab
5. Prometheus proxy — wire metrics panel
6. Fourdogs health depth — wire per-microservice signals
7. Final: version watermark, tab navigation polish

**Rollback plan:** Each phase is a separate PR/commit. ArgoCD can sync to previous commit SHA. Go sidecar is additive — if the sidecar container crashes, nginx continues serving static assets; only `/api/*` routes fail. `failurePolicy: Fail` is NOT set on the sidecar — nginx serves static content regardless.

**Feature flags:** None required. All changes are additive to the existing portal.

---

## Testing Strategy

| Layer | Coverage |
|-------|---------|
| Go sidecar unit | HTTP handler tests per route using `net/http/httptest`; mock GitHub/k8s/Prometheus responses |
| Go sidecar integration | Sidecar container in a test pod with real in-cluster credentials; validate each proxy route end-to-end |
| React component tests | Jest + React Testing Library for Tab component, ReleasePipelineTab, PodsTab — mock `/api/*` responses |
| Manual smoke | After each deploy phase: verify data populates correctly in dev environment |
| E2E | Playwright: tab switching, pipeline table populated, pod count > 0 |

---

## Observability

**Sidecar logs:**
- Structured JSON logs to stdout (k8s log aggregation via Loki)
- Fields: `ts`, `level`, `route`, `upstream`, `status`, `latency_ms`
- Error-level log on upstream timeout or non-2xx from GitHub/k8s/Prometheus

**Sidecar metrics (optional v2):**
- Expose Prometheus metrics at `/metrics` on a separate port (9090) for scraping
- Deferred — not required for portal4

**Portal SPA observability:**
- Version watermark `VITE_APP_VERSION` visible in UI; confirms correct image is deployed
- No JS error tracking added in portal4 scope

**Alerts added:** None in portal4. Observability is passive (logs + visual dashboard).

---

## Open Questions

_None. All architecture decisions were resolved during requirements elicitation before planning began._
