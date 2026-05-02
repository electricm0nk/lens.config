---
feature: terminus-portal-portal4
doc_type: sprint-plan
status: draft
goal: "Deliver the portal4 feature in four focused sprints: visual fixes → Go sidecar foundation → Release Pipeline tab → deep health and metrics wiring"
key_decisions:
  - Visual fixes sprint first to de-risk known regressions before adding complexity
  - Go sidecar scaffolded and deployed before any data features depend on it
  - Release Pipeline tab is sprint 3 milestone — highest business value item
  - Fourdogs deep health and Prometheus metrics in sprint 4 — depend on sidecar + fourdogs-central stub
open_questions: []
depends_on: []
blocks: []
updated_at: "2026-05-02T00:00:00Z"
---

# Sprint Plan: terminus-portal-portal4

## Sprint Overview

| Sprint | Goal | Stories | Complexity | Key Risk |
|--------|------|---------|-----------|----------|
| S1 | Visual fixes and service config | 4 | S, S, S, M | Icon slug discovery may require asset investigation |
| S2 | Go sidecar foundation + Helm wiring | 4 | M, M, L, M | Sidecar RBAC misconfiguration |
| S3 | Tab navigation + Release Pipeline tab | 4 | M, L, M, S | GitHub API rate limit; branch ref resolution edge cases |
| S4 | Deep health signals + Prometheus metrics | 5 | M, M, M, L, S | fourdogs-central health stub scope; Prometheus TLS trust |

---

## Sprint 1 — Visual Fixes and Service Config

**Goal:** All known visual regressions fixed; services.js complete and accurate; portal renders correctly with no card overflow or broken icons.

**Stories:**

| Story ID | Title | Estimate |
|----------|-------|---------|
| portal4-1-01 | Fix Grafana card CSS overflow in ServiceGrid | S |
| portal4-1-02 | Fix ArgoCD and Semaphore icon slugs | S |
| portal4-1-03 | Add Temporal service card to services.js | S |
| portal4-1-04 | Add Fourdogs dev URL and split Fourdogs card metadata | M |

**Acceptance Criteria Summary:**
- ServiceGrid panel no longer overflows on standard viewport sizes
- ArgoCD and Semaphore cards display correct icons (not broken image or default placeholder)
- Temporal appears as a service card in the Platform section
- Fourdogs card includes a dev link to `https://central-dev.fourdogspetsupplies.com/login`

**Risks:**
- Icon slug research may reveal that icon assets are missing from the project — in that case a fallback SVG or text-only card is acceptable for this sprint

---

## Sprint 2 — Go Sidecar Foundation and Helm Wiring

**Goal:** A Go sidecar binary is deployed in the portal pod and accessible via nginx `/api/*` proxy. RBAC, ESO ExternalSecret for PAT, and ServiceAccount are all wired. Sidecar responds to `/api/healthz`.

**Stories:**

| Story ID | Title | Estimate |
|----------|-------|---------|
| portal4-2-01 | Scaffold Go sidecar module with health endpoint | M |
| portal4-2-02 | Add sidecar Dockerfile and Semaphore CI build job | M |
| portal4-2-03 | Update Helm chart: sidecar container, nginx proxy_pass for /api/* | M |
| portal4-2-04 | Wire PAT via Vault ESO ExternalSecret in Helm chart (pre-condition: Vault provisioned) | L |
| portal4-2-05 | Create ServiceAccount and RBAC for k8s pod listing | M |

**Acceptance Criteria Summary:**
- `portal-sidecar` Go binary builds and runs; `/api/healthz` returns 200 JSON
- Sidecar Dockerfile exists; Semaphore CI job builds and pushes image on commit to feature branch
- nginx proxy forwards `/api/*` to sidecar; SPA can call `/api/healthz` successfully
- ESO ExternalSecret syncs `portal-github-pat` Secret from Vault `terminus/github::Portal_PAT` (operator must provision PAT in Vault before this story)
- ServiceAccount `portal-sidecar-sa` bound to Role allowing `get`/`list`/`watch` on `pods`; sidecar can list pods in its own namespace

**Risks:**
- Vault path `terminus/github` or key `Portal_PAT` may not exist yet — operator must provision before S2 deploy; blocker if absent
- nginx config for proxy_pass localhost must be validated with `nginx -t` in container

---

## Sprint 3 — Tab Navigation and Release Pipeline Tab

**Goal:** React SPA ships with three-tab navigation (Overview, Release Pipeline, Pods placeholder). Release Pipeline tab is fully functional: all tracked repos show per-branch tag, SHA, commit message, age, and last workflow run status.

**Stories:**

| Story ID | Title | Estimate |
|----------|-------|---------|
| portal4-3-01 | Implement tab navigation bar in App.jsx (useState) | M |
| portal4-3-02 | Implement `/api/github/repos/:owner/:repo/branches/:branch` sidecar route | L |
| portal4-3-03 | Build Release Pipeline tab component (ReleasePipelineTab.jsx) | M |
| portal4-3-04 | Replace githubActions.js with releasePipeline.js config | S |

**Acceptance Criteria Summary:**
- Three tabs render: Overview (existing layout), Release Pipeline, Pods (placeholder "coming soon")
- `releasePipeline.js` defines all 9 repos with branch topology (see repo table in business plan)
- Each repo row in Release Pipeline shows: dev branch (tag + SHA + age + last CI status), main branch (tag + SHA + age + last CI status)
- Data fetches use `/api/github/*` endpoints (PAT-proxied); all private repos resolve with data
- Tag resolution uses GitHub Releases API as primary source; falls back to `git/matching-refs` semver-sorted; repos with no releases show "no tag" gracefully
- Rate limit response (403/429 from GitHub) surfaces as a "rate limited — try again" state, not a crash
- Version watermark (`VITE_APP_VERSION`) displayed in portal header/footer

**Risks:**
- GitHub `refs/tags` API returns tags alphabetically, not by commit date; tag-to-branch association requires walking commit graph or using `git log --tags --simplify-by-decoration` equivalent — use GitHub Releases API as fallback if tag resolution is unreliable
- Some repos (terminus.infra, lens.config) may have no `develop` branch — config must handle single-branch repos gracefully

---

## Sprint 4 — Deep Health Signals and Prometheus Metrics

**Goal:** All 5 Fourdogs microservices show appropriate health signals; Prometheus metrics (CPU, memory, pods running) are live on the Overview tab; Pods tab is fully functional showing all namespaces.

**Stories:**

| Story ID | Title | Estimate |
|----------|-------|---------|
| portal4-4-01 | Implement `/api/k8s/pods` and `/api/k8s/pods/all` sidecar routes | M |
| portal4-4-02 | Build Pods tab component (PodsTab.jsx) | M |
| portal4-4-03 | Implement `/api/metrics/query` sidecar route (Prometheus proxy; configure internal CA trust) | M |
| portal4-4-04 | Build FourDogsHealthPanel.jsx and wire per-microservice health signals | L |
| portal4-4-05 | Wire metrics.js to live Prometheus endpoints in MetricsPanel | S |

**Acceptance Criteria Summary:**
- Pods tab shows all pods in all cluster namespaces grouped by namespace; ready status, restart count, age visible
- CPU load, memory usage, and pods-running metrics populate from Prometheus on Overview tab
- Prometheus proxy handles internal CA TLS (CA cert mounted as ConfigMap, not skip-verify)
- FourDogsHealthPanel shows: central-ui (up/down), central API (up/down via `/v1/health`), emailfetcher (last import timestamp + 24h error count from `/v1/health/emailfetcher` if available, else "no signal"), etailpet-trigger (readiness probe result), etailpet-sales-trigger (readiness probe result)
- Pre-condition verified before S4-04: k8s Service exists for etailpet-trigger and etailpet-sales-trigger; if absent, a fourdogs-central Helm change is needed and this story is blocked
- All FourDogsHealthPanel health checks route through sidecar (no direct browser calls to internal cluster services)

**Risks:**
- fourdogs-central `/v1/health/emailfetcher` endpoint may not exist yet — this story is blocked on a fourdogs-central change; if not available by S4, emailfetcher cell shows "no signal" gracefully and the story is marked partial-complete pending fourdogs-central
- Prometheus may use self-signed TLS on trantor.internal — sidecar needs to trust the internal CA; tracked as a sidecar config item

---

## Cross-Sprint Dependencies

| Dependency | Blocks |
|------------|--------|
| S2: Go sidecar deployed and healthy | S3 (GitHub proxy), S4 (k8s proxy, Prometheus proxy) |
| S2: Sidecar CI build job wired | S2 deploy itself (image must be built before Helm chart can reference it) |
| S2: PAT ExternalSecret wired | S3 (private repo access) |
| S2: ServiceAccount and RBAC | S4 (pod listing) |
| S3: Tab navigation | S4 (Pods tab renders in tab nav) |
| fourdogs-central `/v1/health/emailfetcher` (external) | S4-04 emailfetcher signal (graceful "no signal" if absent) |
| fourdogs etailpet k8s Services exist | S4-04 readiness probe calls; verify before starting S4-04 |
| Vault PAT provisioned | S2-04 (pre-condition; operator action) |

---

## Definition of Done

Applied to all stories in this plan:

- [ ] Code merged to feature branch (`terminus-portal-portal4`)
- [ ] No TypeScript/ESLint errors (or Go vet errors for sidecar stories)
- [ ] Manual smoke test completed in dev environment — the story's acceptance criteria observed working
- [ ] No secrets committed to source code or visible in browser network traffic
- [ ] Helm chart rendered with `helm template` — no schema validation errors
- [ ] ArgoCD dev app syncs without errors after deploy
