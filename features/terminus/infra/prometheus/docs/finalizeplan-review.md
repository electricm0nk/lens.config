---
feature: prometheus
doc_type: finalizeplan-review
status: complete
updated_at: "2026-04-15T01:00:00Z"
---

# Adversarial Review: prometheus / finalizeplan

**Reviewed:** 2026-04-15T01:00:00Z
**Source:** phase-complete
**Overall Rating:** pass-with-warnings

---

## Summary

Full planning artifact set reviewed: prd, ux-design, architecture, businessplan-adversarial-review, techplan-adversarial-review. No cross-feature blockers. Governance impact cross-check finds no conflicts: `grafana` datasource is pre-wired and requires no changes, `influxdb-monitoring` namespace co-ownership is acknowledged in the architecture, `k3s` has no structural dependency. One medium finding (OQ-3 dashboard JSON authorship must be resolved in story acceptance criteria) and three low findings. No criticals or blockers.

**Recommended next action:** Proceed to bundle — epics, stories, implementation readiness, sprint plan, story files.

---

## Governance Impact Cross-Check

| Feature | Impact | Action Required |
|---------|--------|----------------|
| `grafana` | Datasource already wired in `grafana/values.yaml` — no change needed | None |
| `influxdb-monitoring` | Shares `monitoring` namespace — confirmed no conflict, namespace ownership declared in architecture | None |
| `k3s` | Scrape targets are in-cluster — no topology change | None |
| `secrets` | No new ESO ExternalSecrets for MVP | None |

---

## Findings

### Critical

_None._

### High

_None._

### Medium

| # | Dimension | Finding | Recommendation |
|---|-----------|---------|----------------|
| F1 | Coverage Gaps | OQ-3 (dashboard JSON authorship) unresolved — architecture specifies ConfigMap delivery but not the PromQL, units, thresholds, or variable wiring for the two custom dashboards. This will surface as ambiguity in story acceptance criteria. | In the epics/stories bundle: resolve OQ-3 by specifying that dashboard story ACs require implemented PromQL per the UX panel spec. Use community dashboard 1860 (Node Exporter Full) as the baseline for the operator dashboard; the workload health dashboard requires custom PromQL from the UX design spec. |

### Low

| # | Dimension | Finding | Recommendation |
|---|-----------|---------|----------------|
| F2 | Coverage Gaps | `monitoring` namespace is a hard runtime pre-requisite (owned by `influxdb-infra`) — not yet in any readiness checklist. | Include as a pre-deploy gate item in `implementation-readiness.md`. |
| F3 | Assumptions | `forceDeployDashboards: true` behavior with `grafana.enabled: false` is unverified across chart minor versions. If it does not emit ConfigMaps, the dashboard story needs a fallback path. | Add as a story-level assumption: acceptance criteria require confirming ConfigMaps appear in `monitoring` namespace after sync. Fallback: embed dashboard JSON directly in the custom `prometheus-infra` ConfigMaps. |
| F4 | Logic | Architecture's file structure defines the full repo shape but no story boundaries. Bundle step will establish these. | No action before bundle; track in epics. |

---

## Accepted Risks

| Finding | Acceptance Rationale |
|---------|----------------------|
| F3 (forceDeployDashboards) | Verifiable at first deploy; fallback path is low-effort (move JSON to custom ConfigMaps). |
| F4 (no pre-existing story boundaries) | Normal at FinalizePlan entry; bundle step creates them. |

---

## Party-Mode Challenge

**Riya (Product):** The journey from "I see a node CPU spike" to "I know which pod caused it" isn't spelled out in the stories yet. Make sure story acceptance criteria go beyond "Prometheus pod is Running" — they need to verify that a real query returns real data in Grafana.

**Clara (Platform ops):** ~~Grafana sidecar watchMethod concern~~ — **closed**: Grafana runs in its own container; sidecar watches dynamically. No restart required for new dashboard ConfigMaps to appear.

---

## Gaps You May Not Have Considered

1. Are there any acceptance-criteria-level details in FR-1 through FR-5 that are too vague to close a story (e.g. "scrape k3s control-plane metrics" — what does done look like)? Define done in terms of a visible Grafana panel, not just a running pod.
2. Does the sprint plan need an explicit verification/smoke story, or is the validation playbook in the architecture sufficient? The influxdb-monitoring pattern included Semaphore verify playbooks; consider whether the same is needed here for operational consistency.
3. The `prometheus-plan` branch currently contains no commits beyond the branch creation point. All planning docs are staged in `main`-tracked paths. Confirm the commit-and-push in Step 1 lands the review artifact on `prometheus-plan` before the Step 2 PR is created.

---

## Open Questions Carried Forward

| # | Question | Resolution Path |
|---|----------|----------------|
| OQ-3 | Dashboard JSON authorship / PromQL spec | Resolve in story ACs during epics bundle |
| OQ-5 | CRD sync wave race on first deploy | Validate during implementation; note in readiness doc |
| M3 (from techplan) | `forceDeployDashboards` behavior with `grafana.enabled: false` | Verify on first dev sync; fallback documented |
