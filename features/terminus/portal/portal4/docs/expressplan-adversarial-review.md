---
feature: portal4
doc_type: adversarial-review
status: complete
overall_rating: pass-with-warnings
reviewed_at: "2026-05-02T00:00:00Z"
---

# Adversarial Review: portal4

**Reviewed:** 2026-05-02  
**Overall Rating:** pass-with-warnings

## Summary

The business plan and tech plan are internally consistent and well-scoped. The Go sidecar architecture is sound, the ADR rationale is solid, and the phased rollout strategy is realistic. Three high-severity gaps were identified: the fourdogs-central health stub endpoint is an unregistered cross-feature dependency that blocks sprint 4 delivery; the sidecar container requires a Go build pipeline that is absent from the sprint plan; and the same fourdogs-central gap creates a sprint 4 delivery risk with no existing mitigation story. Five medium findings cover operational provisioning and minor implementation complexity. No critical findings. Recommended action: add missing stories to sprint plan and acknowledge accepted risks before proceeding to sprint execution.

---

## Findings

### High

| # | Dimension | Finding | Recommendation |
|---|-----------|---------|----------------|
| H1 | Coverage Gaps | fourdogs-central `/v1/health/emailfetcher` endpoint is required by S4-04 (FourDogsHealthPanel) but is an external dependency on the `fourdogs-central` repo with no tracking story or cross-feature registration. If this endpoint does not exist when S4 begins, the emailfetcher health signal in the portal is blocked. | Add a prerequisite story in S4 that checks for endpoint availability and falls back gracefully ("no signal" state). Separately, create a fourdogs-central ticket for the health stub. Register fourdogs-central as an informal dependency in the sprint plan notes. |
| H2 | Coverage Gaps | The Go sidecar requires a Dockerfile and a CI build job (Semaphore) to build and push the image to a container registry. Neither the Dockerfile location nor the CI job change are in the sprint plan. Without this, S2 cannot deploy. | Add a story to S2: "Add sidecar Dockerfile and CI build job." |
| H3 | Complexity and Risk | Sprint 4 has 5 stories, the longest dependency chain, and is blocked by two external factors (fourdogs-central stub + Prometheus TLS). If both are unresolved, 2 of 5 S4 stories cannot complete. | Make fourdogs-central stub verification the first story in S4. Document Prometheus TLS configuration requirements in the S4 Prometheus proxy story before starting. |

### Medium / Low

| # | Severity | Dimension | Finding | Recommendation |
|---|---------|-----------|---------|----------------|
| M1 | Medium | Coverage Gaps | `VITE_APP_VERSION` version watermark requires CI to inject the git tag at build time (e.g., `VITE_APP_VERSION=$(git describe --tags --abbrev=0)`). The Semaphore CI job for the portal does not currently do this. No story covers the CI change. | Small addition to S3's version watermark story: include the CI yaml change to set `VITE_APP_VERSION` from `git describe`. |
| M2 | Medium → Resolved | Assumptions | **RESOLVED 2026-05-02:** `kubectl get svc` confirmed no Services and no container ports or probes on either trigger worker deployment. Health signal for both workers is k8s pod phase + restart count via the existing `/api/k8s/pods` sidecar route — no HTTP probe, no Service, no fourdogs-central Helm change needed. | Update FourDogsHealthPanel S4-04 AC to use pod status, not probe result. |
| M3 | Medium | Complexity | The tech plan specifies "latest tag reachable from this SHA" as part of the sidecar GitHub proxy response. GitHub REST API has no direct "latest tag on branch" endpoint. The `git/matching-refs` endpoint returns tags alphabetically, not by commit date. Resolving the most recent tag requires listing commits and cross-referencing against tags — potentially multiple API calls per repo. | Use GitHub Releases API as primary source (`/repos/:owner/:repo/releases/latest`). Fall back to `git/matching-refs?ref=tags` sorted by semver if no release exists. Accept that repos with no releases show "no tag" gracefully. |
| M4 | Medium → Resolved | Assumptions | **RESOLVED 2026-05-02:** `vault kv get terminus/github` confirmed present with `Portal_PAT` key. S2 pre-condition is met. | No action needed. |
| M5 | Medium | Assumptions | `https://prometheus.trantor.internal` TLS certificate is issued by the internal CA (cert-manager or Vault PKI). The Go sidecar's default TLS client config uses the system root CA bundle. If the internal CA is not in the system root bundle (likely in a distroless image), TLS verification will fail. | Mount the internal CA cert as a ConfigMap in the sidecar container, or configure `InsecureSkipVerify: true` for the Prometheus client only (document this explicitly). Preferred: mount the CA cert. |

---

## Accepted Risks

| Risk | Rationale |
|------|-----------|
| fourdogs-central health stub may not be available by S4 | Accepted: S4-04 is designed to show "no signal" gracefully if the endpoint is absent. Delivery of the stub is a separate fourdogs initiative item, not a portal4 blocker for the other 4 S4 stories. |
| Single operator — no peer review on code changes | Accepted: homelab single-operator project. LENS adversarial review substitutes for peer code review at the planning stage. Story-level DoD includes manual smoke tests. |

---

## Open Questions Surfaced

1. ~~Does a k8s Service exist for fourdogs-etailpet-trigger and fourdogs-etailpet-sales-trigger?~~ **Resolved 2026-05-02:** No Services or probes exist; k8s pod status via `/api/k8s/pods` is the health signal.
2. ~~Is `vault kv get terminus/github` accessible now, and does the `Portal_PAT` key exist?~~ **Resolved 2026-05-02:** Key confirmed present.
3. ~~What container registry is used for the portal sidecar image?~~ **Resolved 2026-05-02:** Same GHCR org as other images.
