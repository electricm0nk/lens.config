# Adversarial Review: fourdogs-etailpet-api-sales / techplan

**Reviewed:** 2026-04-27T00:00:00Z
**Source:** phase-complete
**Overall Rating:** pass-with-warnings

## Summary

The TechPlan artifact (`architecture.md`) is well-structured and grounded in the established
`transactions` patterns. The component design, auth flow, retry policy, observability, and
security model are all sound. Two Medium findings require attention before Sprint 1 begins: the
probe HTTP server is absent from the `main.go` component design (developers will need to implement
it without guidance), and the concurrent trigger safety under retry is undocumented. No Critical
findings. TechPlan may complete and advance to FinalizePlan, with the Medium findings tracked as
story-level clarifications.

---

## Findings

### Critical

*None.*

---

### High

*None.*

---

### Medium

| # | Dimension | Finding | Resolution |
|---|-----------|---------|-------------|
| M1 | Coverage Gaps | Probe HTTP server was absent from `main.go` component design | **Resolved** — `ProbeServer` goroutine, port `:8080`, `/healthz`/`/readyz` endpoints, and `atomic.Bool` ready-state added to `architecture.md` (2026-04-27) |
| M2 | Complexity / Risk | Concurrent trigger safety under retry was undocumented | **Resolved** — single-goroutine ticker model with blocking `select` documented in `architecture.md`; concurrent execution is structurally impossible (2026-04-27) |

---

### Low

| # | Dimension | Finding | Recommendation |
|---|-----------|---------|----------------|
| L1 | Logic Flaws | `architecture.md` frontmatter has `feature: fourdogs-etailpet-api-sales` (full feature ID) while all other docs use `feature: sales`. | Update to `feature: sales` for consistency with the feature slug convention. |
| L2 | Coverage Gaps | CI/CD image build pipeline is not documented. The Helm chart needs an image reference, but the architecture doesn't state where `cmd/etailpet-sales-trigger` is built and pushed. | Add a note to architecture.md: "Image build is handled by the existing `fourdogs-central` CI pipeline; confirm a build job exists for the new binary or add one as part of `sales-1-04`." |
| L3 | Assumptions | ESO Secret timing: if the ExternalSecret is not populated before the first pod starts, the pod exits 1. No sync-wave annotation or `secretKeyRef.optional: false` is documented. | Note in the Helm chart section: add an ArgoCD sync-wave annotation or confirm ESO pre-population before the Deployment is applied. |

---

## Accepted Risks

| # | Finding | Accepted By | Rationale |
|---|---------|-------------|-----------|
| H2 (carried from businessplan) | Fire-and-forget: trigger success ≠ email delivered | Todd Hintzmann, 2026-04-27 | Accepted at businessplan phase; no change at techplan. |
| H1 (carried from businessplan) | No fallback plan if spike fails | Todd Hintzmann, 2026-04-27 | Accepted at businessplan phase; no change at techplan. |

---

## Party-Mode Challenge

**Perturabo (Architect):** The probe HTTP server is missing from the component design. You have a Helm chart with liveness and readiness probes, but `main.go` only shows a `time.Ticker` loop and a `signal.NotifyContext`. Nothing in the architecture says "start an HTTP server on port X in a separate goroutine." If the developer implements exactly what's in `architecture.md`, the probes will fail and the deployment will never become healthy. Add the HTTP server to the component design.

**Magos Domina (Developer):** Two things a developer will ask immediately when picking up `sales-1-01`: (1) Where in `internal/etailpet/` does `LegacyClient` live — same file as `PosClient`, or a new `legacy_client.go`? (2) Does retry logic live in `LegacyClient` itself, or in a shared `retry.go`? The architecture shows `LegacyClient.TriggerSalesReport` but says nothing about file layout or retry ownership. A guess is a decision without a review.

**Watch-Captain Artemis (QA):** The spike gate `sales-0-01` must validate six confirmation points, but the architecture says nothing about how the spike is executed. If it's an ad-hoc curl, the token flow and date format findings may not generalize to the `LegacyClient` implementation. The spike should use minimal actual client code, not a manual shell script.

---

## Gaps You May Not Have Considered

1. **The probe HTTP server is absent from the main.go design.** How does the readiness probe endpoint run alongside the ticker loop? Port, paths, liveness vs. readiness response logic?
2. **ESO Secret timing.** If the ExternalSecret isn't populated before the first pod start, the pod exits 1. Does the deploy process handle this, or does the chart need a sync-wave annotation?
3. **Concurrent trigger execution.** If retries are in flight and the ticker fires, can two trigger calls run simultaneously? The architecture should state whether this is prevented and how.
4. **CI/CD image build pipeline.** The Helm chart needs an image reference. Where is the binary image built and pushed — existing `fourdogs-central` CI, or a new job needed?
5. **`target_repos` missing from `feature.yaml`.** Developers picking up stories won't know which repo to work in from governance alone.

---

## Open Questions Surfaced

1. Should `fourdogs-central` be registered in `feature.yaml.target_repos` before stories are assigned?
2. Is the probe HTTP server a shared utility in `fourdogs-central` (reused from other workers), or must it be implemented from scratch?
3. Does the `fourdogs-central` CI pipeline already build Docker images per binary, or does a new pipeline job need to be added in `sales-1-04`?
4. What ArgoCD sync-wave or ESO wait strategy is used for the Deployment ↔ ExternalSecret ordering?
