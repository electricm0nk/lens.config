---
feature: portal4
doc_type: implementation-readiness
status: approved
goal: "Assess whether portal4 is sufficiently bounded, understood, and verified to hand off into implementation."
key_decisions:
  - "Go sidecar as second container — ADR-1 through ADR-7 documented and approved in tech-plan"
  - "S1 can start immediately on the portal4 branch"
  - "S2 pre-condition (Vault PAT) confirmed present before finalizeplan gate"
open_questions: []
depends_on: []
blocks: []
updated_at: "2026-05-02T00:00:00Z"
---

# Implementation Readiness — portal4

## Summary

| Category | Status |
|---|---|
| Scope | ✅ READY |
| Architecture | ✅ READY |
| Repo Context | ✅ READY |
| Pre-conditions | ✅ READY |
| Cross-feature Conflicts | ✅ NONE BLOCKING |
| Verification Path | ✅ READY |
| Operational Risks | ⚠️ NOTED |
| **Overall** | **✅ PROCEED** |

---

## Readiness Assessment

### Scope Readiness

- 18 stories across 4 sprints; all story IDs assigned and files present in `stories/`
- Sprint sequence enforced: S2 (sidecar) must be deployed before S3 and S4 data features
- Each sprint has a clear deploy gate and AC summary
- fourdogs-central emailfetcher dependency is conditionally blocked but gracefully handled — story 4-04 has a "no signal" fallback; not a blocker for Sprint 4 start

### Architecture Readiness

- 7 ADRs documented in tech-plan.md covering all key technical decisions (sidecar topology, ESO PAT, SA RBAC, Prometheus proxy, GitHub proxy, tab navigation, releasePipeline config)
- Go sidecar architecture is well-understood: nginx proxies `/api/*` to `localhost:8080`; sidecar handles all external API calls
- Security constraint documented: sidecar must not log auth headers or env vars (PAT leak prevention)
- Internal CA trust for Prometheus is explicitly in S4-03 AC

### Repo and Environment Readiness

- Target repo: `TargetProjects/terminus/portal/terminus-portal/` — React/Vite SPA + nginx container; already has CI, tests, and ArgoCD deployment
- Feature branch `portal4` exists and is checked out
- GHCR org confirmed for sidecar image registry
- Vault PAT confirmed present at `terminus/github::Portal_PAT`

### Pre-condition Status

| Pre-condition | Status | Notes |
|---|---|---|
| Vault PAT `terminus/github::Portal_PAT` | ✅ Confirmed | Verified by operator before finalizeplan |
| GHCR org for sidecar image | ✅ Confirmed | Same org as other images |
| `portal-github-pat` ESO ClusterSecretStore | ⏳ S2 work | Created in story portal4-2-04 |
| `prometheus-wiring` deployed | ✅ Confirmed | phase: finalizeplan-complete in governance |
| fourdogs-central `/v1/health/emailfetcher` | ⚠️ Unknown | Graceful fallback if absent; not a blocker |

### Verification Readiness

- S1–S3 (React stories): `npm test` + `npm run build` + manual smoke test in dev
- S2 (Go sidecar): `go vet ./...` + `go build ./...` + `helm template` + ArgoCD sync + `/api/healthz` smoke test
- S4 (metrics): manual observation of Prometheus data populating MetricsPanel in dev browser
- Definition of Done applies uniformly across all stories

### Cross-Feature Assessment

- `prometheus-wiring` (finalizeplan-complete): Prometheus confirmed deployed — S4 Prometheus proxy dependency met
- `portal3` (finalizeplan-complete): portal4 is an additive extension; no regressions expected
- `releaseorchestrator` (preplan): portal4 Release Pipeline tab is read-only advisory, not a control plane — no conflict
- No other active features in `terminus/portal` compete with portal4 scope

---

## Risks

| Risk | Severity | Mitigation |
|---|---|---|
| fourdogs-central `/v1/health/emailfetcher` not available by S4 | Medium | Story 4-04 has explicit "no signal" graceful fallback; partial-complete acceptable |
| PAT log-safety — sidecar must not log auth headers or env vars | Medium | Explicit constraint in tech-plan ADR-2 and story 2-01 AC |
| Prometheus internal CA TLS misconfiguration in sidecar | Medium | Story 4-03 AC explicitly requires CA cert mounted as ConfigMap, not skip-verify |
| nginx `proxy_pass localhost:8080` buffer config for large responses | Low | Homelab k3s scale makes this a non-issue; operator can tune if needed |
| `releaseorchestrator` future config schema may conflict with `releasePipeline.js` | Low | Portal4 is read-only advisory; alignment can happen at that feature's design time |

---

## Readiness Decision

**Gate Status:** ✅ PASSED

Portal4 is ready for `/dev`. The implementation branch exists, all pre-conditions are verified or have graceful fallbacks, all 18 story files are written, and the target repo supports the needed change pattern.

**Start with:** Sprint 1 — `portal4-1-01` (Fix Grafana card CSS overflow in ServiceGrid)
