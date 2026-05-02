---
feature: portal4
doc_type: finalizeplan-review
status: approved
goal: "Validate the complete portal4 planning set is internally consistent, cross-feature safe, and dev-ready; confirm sprint S1 can start immediately"
key_decisions: []
open_questions: []
depends_on: []
blocks: []
updated_at: "2026-05-02T00:00:00Z"
---

# FinalizePlan Review: portal4

**Reviewed:** 2026-05-02  
**Phase:** finalizeplan  
**Preceding review:** expressplan-adversarial-review (pass-with-warnings, all findings resolved)  
**Overall Rating:** **approved — ready for dev**

---

## Summary

The complete planning set (business-plan, tech-plan, sprint-plan + expressplan adversarial review with post-review corrections) is internally consistent and cross-feature safe. All three open questions from the expressplan review were resolved before finalizeplan entry. The party-mode challenge round (architect, PM, QA, scrum master) surfaced one new medium finding (S2 sprint story count now 5 — table header shows 4, minor inconsistency) and confirmed the planning set is development-ready. The `prometheus-wiring` feature at finalizeplan-complete confirms Prometheus is deployed. No blocking cross-feature conflicts. One governance overlap note: the `releaseorchestrator` feature at preplan covers similar territory to portal4's Release Pipeline tab — flagged for awareness only, not a blocker at this scope. S1 can start immediately on the `portal4` branch.

---

## Party-Mode Challenge Round

### Architect (Perturabo) — Technical review

**Challenge:** The nginx `proxy_pass localhost:8080` for `/api/*` will fail for any request whose response body is large (e.g., listing pods across all namespaces on a large cluster). Has response buffering and timeout been considered?

**Response:** The cluster is a homelab k3s with ≤10 namespaces and ≤50 pods. nginx default buffer sizes (proxy_buffer_size 4k, proxy_buffers 8 4k) are sufficient. Sidecar sets a 10s upstream timeout. Not a concern at this scale.

**Challenge:** The sidecar distroless image won't have a CA bundle by default. The TechPlan says "mount the internal CA cert as a ConfigMap" — but the ConfigMap doesn't exist yet and isn't a story. Is this a blocker for S4?

**Response:** The ConfigMap creation is implicit in story S4-03 "Implement /api/metrics/query sidecar route (Prometheus proxy; configure internal CA trust)". The AC explicitly mentions "CA cert mounted as ConfigMap, not skip-verify". The ConfigMap creation is part of that story's implementation. Acceptable — the story is self-contained.

**Challenge:** The sidecar receives the GitHub PAT as an env var. If the sidecar logs the env or panics with a stack trace that includes env vars, the PAT could leak to logs. Has this been addressed?

**Response:** Valid concern. Adding a note to tech-plan ADR-2 and to the S2-01 (sidecar scaffold) story: sidecar must not log env vars in any code path; use structured logging that explicitly omits auth headers/env. **New medium finding M-F1.**

**Verdict:** Technical architecture is sound. One new finding added.

---

### PM (Lord Commander Creed) — Business value review

**Challenge:** The sprint overview table in the sprint-plan shows S2 as "4 stories" but S2 actually has 5 stories (portal4-2-01 through portal4-2-05). Minor but creates confusion when tracking sprint progress.

**Response:** Confirmed — table header is stale from before the adversarial review patch added story portal4-2-02 (sidecar CI build job). **New low finding L-F1.** Fix the table.

**Challenge:** Success criterion "Version watermark (VITE_APP_VERSION) displayed in portal header/footer" — this is S3. If S3 is delayed, S4 deployment can't easily be identified by version. Is there a fallback?

**Response:** The version watermark is cosmetic. S4 can be deployed and verified without it. ArgoCD image SHA provides deployment identity. Not a blocker.

**Verdict:** Business plan and success criteria are complete and achievable. One low finding (table inconsistency).

---

### QA (Watch-Captain Artemis) — Testing and readiness review

**Challenge:** The Definition of Done says "No TypeScript/ESLint errors (or Go vet errors for sidecar stories)" but doesn't mention `go build` succeeding or container image building cleanly. A Go module can pass `go vet` but fail to build with missing deps.

**Response:** Valid gap. Adding `go build ./...` to DoD for sidecar stories explicitly. **New low finding L-F2.** The existing "Manual smoke test" DoD item would catch a failing image in practice, but the DoD should be explicit.

**Challenge:** There are no E2E test stories in the sprint plan, despite the tech plan mentioning Playwright E2E tests. Are tests out of scope for this sprint plan?

**Response:** The tech plan lists E2E as a testing strategy aspiration, not a committed story. For a homelab single-operator tool, manual smoke tests per the DoD are the primary test gate. E2E scaffold is a future enhancement. This is a known accepted scope boundary.

**Verdict:** Testing strategy is appropriate for scope and team size. Two low findings.

---

### SM (High Chaplain Grimaldus) — Sprint planning completeness

**Challenge:** Sprint 1 has 4 stories estimated as S/S/S/M. The total is approximately 7 points. Sprint 2 has 5 stories at M/M/M/L/M — approximately 17 points. This is a 2.4x size difference between sprints. Is this realistic for single-operator execution?

**Response:** S1 is intentionally light — it's a setup sprint with known, bounded changes. S2 is the heavy infrastructure sprint. Single-operator pace means sprints don't have fixed time-boxes; they complete when done. The size difference is acceptable.

**Challenge:** Story portal4-2-04 (ESO ExternalSecret) is marked L but depends on Vault being provisioned. Vault PAT is confirmed present. The story is large because of Vault config validation — is this the right estimate?

**Response:** L is appropriate. ESO ExternalSecret + ClusterSecretStore validation + testing the sync cycle is genuinely L for first-time setup on this cluster pattern, even with the PAT present.

**Challenge:** Are story IDs consistent with the LENS story file naming convention (`{featureId}-{sprintNumber}-{storyNumber}` per sprint-planning reference)?

**Response:** Yes — `portal4-1-01`, `portal4-2-01`, etc. Consistent with the convention.

**Verdict:** Sprint plan is well-structured for single-operator execution. S2 story count inconsistency in the overview table is the same as PM finding L-F1.

---

## New Findings from Party-Mode Round

| # | Severity | Source | Finding | Resolution |
|---|---------|--------|---------|-----------|
| M-F1 | Medium | Architect | Sidecar must not log env vars or auth headers in any code path (PAT leak via logs) | Add explicit constraint to ADR-2 in tech-plan and to S2-01 AC: "sidecar must not log request auth headers or environment variables" |
| L-F1 | Low | PM / SM | Sprint overview table shows S2 as "4 stories" — should be 5 after adversarial review patch | Fix overview table in sprint-plan (minor) |
| L-F2 | Low | QA | DoD lacks explicit `go build ./...` check for sidecar stories | Add to Definition of Done for sidecar-tagged stories |

**All findings:** 0 critical, 0 high, 1 medium, 2 low → **approved — no blockers**

---

## Governance Cross-Check

### Features Reviewed for Overlap

| Feature | Phase | Overlap Assessment |
|---------|-------|--------------------|
| `portal3` | finalizeplan-complete | Predecessor; portal4 extends it. No conflict. |
| `prometheus-wiring` | finalizeplan-complete | **Positive:** Confirms Prometheus deployed at `https://prometheus.trantor.internal`. S4 dependency met. |
| `releaseorchestrator` | preplan | **Awareness:** `releaseorchestrator` will likely build a more complete release orchestration system. portal4's Release Pipeline tab is read-only and advisory — not a competing control plane. Risk: if `releaseorchestrator` defines its own release config schema, portal4's `releasePipeline.js` may need alignment. Currently preplan only, not a blocker. |
| `temporal` | preplan | portal4 adds a Temporal service card. Card may show UNREACHABLE if Temporal isn't deployed when S1 runs — acceptable; the card is additive only. |
| `grafana` | preplan | portal4 fixes Grafana card CSS. No overlap — CSS fix is cosmetic. |
| `deliverycontrol` | preplan | Similar naming to Release Pipeline; no active overlap at this phase. |

**Governance impact:** None blocking. `releaseorchestrator` flagged for future alignment awareness.

---

## Merge Readiness Assessment

| Criterion | Status |
|-----------|--------|
| All expressplan open questions resolved | ✅ (3/3 resolved before finalizeplan entry) |
| No critical or high findings in either review | ✅ |
| Sprint S1 has no unresolved blockers | ✅ |
| Sprint S2 pre-conditions met (Vault PAT confirmed) | ✅ |
| Sidecar CI registry confirmed (GHCR, same org) | ✅ |
| `portal4` branch exists | ✅ |
| `portal4-plan` branch exists | ✅ |
| Cross-feature conflicts | None blocking |

**Decision: approved for planning PR creation.**

---

## Action Items Before Step 3 Bundle

- [ ] Fix sprint overview table S2 story count (4 → 5) — L-F1
- [ ] Add `go build ./...` to DoD for sidecar stories — L-F2  
- [ ] Add PAT log-safety constraint to tech-plan ADR-2 and S2-01 AC — M-F1
