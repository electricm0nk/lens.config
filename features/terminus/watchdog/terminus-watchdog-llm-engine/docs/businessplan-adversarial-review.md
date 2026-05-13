---
feature: terminus-watchdog-llm-engine
doc_type: adversarial-review
phase: businessplan
status: complete
verdict: pass-with-warnings
updated_at: 2026-05-13
reviewed_artifacts:
  - docs/terminus/watchdog/terminus-watchdog-llm-engine/prd.md
  - docs/terminus/watchdog/terminus-watchdog-llm-engine/ux-design.md
---

# BusinessPlan Adversarial Review — terminus-watchdog-llm-engine

**Date:** 2026-05-13  
**Phase:** businessplan  
**Verdict:** PASS-WITH-WARNINGS  
**Finding Summary:** Critical: 0 | High: 2 | Medium: 4 | Low: 3

---

## Reviewed Artifacts

| Artifact | Status |
|---|---|
| `prd.md` | ✅ Present |
| `ux-design.md` | ✅ Present |

---

## Findings

### HIGH-1 — DEP-1 (Inference Gateway Art.10) Has No Decision

**Source:** `prd.md` Section 6, OQ-2

The PRD correctly names the inference gateway Art.10 observability gap as a blocker candidate (DEP-1). The resolution condition is defined as "governance decision before BusinessPlan can be accepted by both teams." However, in a solo-operator context there is no second team. OQ-2 remains open with no decision recorded.

The blocker is real: the terminus-inference-gateway architecture explicitly defers Art.10 compliance (route id, duration, token counts, outcome, cost tag per request) as a TODO before production readiness. If this feature enters dev against a gateway that is constitutionally blocked from production readiness, it inherits the block.

**Required resolution:** Record an explicit decision before TechPlan begins: either (a) waiver granted — this feature ships with an Art.10 follow-on story registered in the backlog, acknowledging the gateway is not yet Art.10-compliant, or (b) the inference gateway Art.10 story will be scoped into TechPlan as a prerequisite before dev begins. OQ-2 must be closed.

---

### HIGH-2 — Temporal Search Attribute Registration Is an Unverified Architectural Dependency

**Source:** `prd.md` Section 6 DEP-3, OQ-1

The Temporal-first observation store (Option B) — the confirmed architecture — depends on custom search attributes being registerable on the current self-hosted Temporal deployment. This requires:
1. The Temporal server to support custom search attribute registration (`tctl` admin op)
2. A Visibility backend (Elasticsearch or equivalent) that supports custom attribute filtering in `list_workflows` queries

Some minimal self-hosted Temporal deployments run without Elasticsearch and fall back to basic visibility, which does not support `list_workflows` with custom search attribute filters. If basic visibility only is available, Option B's batch analysis queries are not possible and the observation store design is unviable without a SQLite fallback.

OQ-1 is tagged "needed by TechPlan" but this is a feasibility question that determines whether the entire architecture is valid — it should be verified before TechPlan is authored, not during it.

**Required resolution:** Verify before TechPlan begins: (a) is `tctl namespace search-attribute add` available on the current Temporal deployment? (b) is a Visibility backend (Elasticsearch) configured? If both answers are yes, Option B proceeds. If not, the SQLite fallback (original research.md design) must be reinstated and the PRD updated.

---

### MEDIUM-1 — `custom_id` Length Constraint Creates Approval Flow Risk

**Source:** `ux-design.md` Section 8

Discord `custom_id` maximum length is 100 characters. The UX spec acknowledges the risk and suggests "use hash or truncated timestamp+pattern combination." The example format `approve:terminus-triage-2026-05-13-fourdogs-central-abc123` is already 57 characters without the full Temporal `workflow_run_id` (a UUID is 36 chars, bringing the total to 93 — near the limit with no margin for longer pattern IDs or resource names).

If the `custom_id` exceeds 100 characters for any alert, Discord rejects the button registration. The approval flow silently breaks: the embed is sent, the buttons appear to register, but clicks produce no interaction event. This is a silent failure mode.

**Resolution:** TechPlan must define the `custom_id` format precisely: use a stable short hash of `workflow_run_id` (e.g., first 16 chars of SHA-256 hex) to guarantee the `action:{hash}` format never exceeds 30 characters total. The Temporal workflow must be retrievable by the full run ID; only the button encoding uses the hash with a lookup table in memory.

---

### MEDIUM-2 — Silent-Tier Auto-Confirm Creates Misleading Observation Completeness

**Source:** `prd.md` FR-3, E2 acceptance criteria

Silent-tier observations auto-confirm after 7 days as `HumanVerdict=auto-confirmed`. These are contributed to the batch analysis pool. However, if a silent-tier classification was incorrect (the alert was actually a real problem the operator fixed manually), there is no mechanism to retroactively record an `incorrect` verdict. The 7-day window simply runs out and the observation is marked correct.

For graduation analysis, `no-action` classifications (the most common silent-tier outcome) would not meet graduation criteria anyway (they don't lead to detector candidates). But `transient-glitch` or `known-degraded-dependency` with `action_tier=silent` might accumulate auto-confirmed verdicts that look like perfect graduation candidates even if some were misclassified.

**Resolution:** E2 acceptance criteria should explicitly state that for graduation analysis, `HumanVerdict=auto-confirmed` counts as neutral (not confirmed correct). Only `HumanVerdict=correct` (explicit operator approval) counts toward graduation threshold. Update PRD graduation criteria accordingly.

---

### MEDIUM-3 — Prometheus Observability Surface Missing from UX Design

**Source:** `prd.md` NFR-3, `ux-design.md`

The PRD defines two Prometheus metrics: `watchdog_llm_triage_calls_total` and `watchdog_llm_triage_latency_seconds`. The UX design correctly scopes the Discord interface only. However, there is no reference to how the operator monitors LLM subsystem health in aggregate — rising gateway unreachability rates, latency degradation, or classification distribution shifts. These are visible in Prometheus/Grafana but the UX design does not mention this monitoring surface exists.

A sustained gateway-unreachable condition produces gray "LLM unavailable" embeds one at a time. Without a Prometheus view, the operator cannot easily tell if this is a single transient event or a 4-hour outage.

**Resolution:** Add a note in `ux-design.md` Section 1 or Section 7 that the Prometheus metrics endpoint (`/metrics`) and the existing Grafana ServiceMonitor are the secondary observability surface for aggregate LLM subsystem health. This closes the UX design's coverage of all operator-facing surfaces.

---

### MEDIUM-4 — `never-touch` Resource List Is Unspecified

**Source:** `prd.md` FR-2, `ux-design.md` Template 3

The `never-touch` action tier is defined and correctly handled in both PRD and UX design. Template 3 shows a PVC example. Neither document defines where the never-touch resource list lives or how the classification routing code determines that a given resource is never-touch.

Without this definition, the TechPlan architecture story cannot be written: it cannot specify the config key, the config format, or the lookup logic in the triage routing layer.

**Resolution:** TechPlan must define never-touch resource designation. Recommended approach: a `never_touch` config section in Helm values (list of resource patterns: type/name regex), loaded as a watchdog ConfigMap, and checked in the `TriageWorkflow` before any action tier is acted upon. This must be defined before E1 story authoring.

---

### LOW-1 — 70% Confidence Warning Threshold Is Unjustified

**Source:** `ux-design.md` Template 2 (low-confidence variant)

The 70% threshold for displaying a `⚠️` confidence warning is not justified in the UX design or PRD. It is a reasonable starting point but should be made explicit and adjustable (a config value or Helm value) rather than hardcoded. TechPlan should define it as a configurable parameter.

---

### LOW-2 — E3 Deferred Scope Has No Explicit Backlog Disposition

**Source:** `prd.md` Section 3 (E3), Section 7 (Out of Scope)

E3 (ArgoCD Notifications webhook) is correctly deferred post-MVP. However, neither the PRD nor the product brief specifies how E3 is tracked after this feature closes — separate backlog feature, follow-on epic, or simply informal. Given the lifecycle tracks features to completion, a disposition note is needed.

**Resolution:** Add a note to the E3 section: "E3 deferred to a separate follow-on feature. No lifecycle tracking in this feature."

---

### LOW-3 — UX Template Widths Are Notional

**Source:** `ux-design.md` Sections 2–4

The ASCII embed diagrams use fixed-width 72-char columns. Discord embed widths are client-variable. The diagrams communicate intent but should not be interpreted as pixel-accurate by the TechPlan implementer. Add a note in Section 8: "ASCII templates are schematic only; actual Discord embed layout is client-dependent."

---

## Party-Mode Blind-Spot Challenge

**Artemis (QA):** The E5 BDD test "gateway unreachable, 5-second fast-fail" requires a test that either waits 5 real seconds or uses a mock that fires at 5s. Without timeout virtualization, 30 such tests in a suite adds 2.5 minutes to CI runtime. How do the E5 acceptance tests handle clock/timeout mocking so they run at normal pytest speed?

**Perturabo (Architect):** Option B assumes Temporal Visibility API with custom search attribute filtering. For self-hosted Temporal, this requires Elasticsearch as the Visibility backend. Some minimal Temporal setups (including common Helm chart defaults with SQLite or Postgres-only configurations) do not include Elasticsearch, and the Visibility API in basic mode only supports workflow ID and status filtering — not custom search attributes. This is the same question as OQ-1 but from a different angle: even if `tctl` accepts attribute registration, the queries will silently return no results if the backend doesn't index them. Verify the full stack, not just the admin command availability.

**Creed (PM):** The PRD success criterion for E2 is: "after 30 days of production use, at least one novel pattern has met graduation criteria and been promoted to a hardcoded detector." But the updated E2 design requires a human to actively engage a dev agent or write the detector. This is out of scope for the feature itself. Consider reframing: "at least one novel pattern has met graduation criteria and is surfaced in the batch analysis report" — which the feature can deliver without a human developer action.

### Blind-Spot Questions

1. **Temporal Visibility backend:** Is Elasticsearch (or equivalent) configured in your self-hosted Temporal setup? Basic visibility alone makes Option B batch queries unworkable.

2. **Timeout mocking strategy:** How will the E5 gateway-unreachable test handle the 5-second timeout in pytest without real wall-clock delay?

3. **E2 success criterion:** Should the graduation success criterion be reframed to what the feature can deliver (surfacing the candidate) vs. what requires a human action (writing the detector)?

4. **never-touch config location:** Do you want never-touch resources in Helm values, a ConfigMap, or hardcoded in Python config? This determines TechPlan architecture for FR-5.

5. **7-day auto-confirm timer:** Does your Temporal server handle 7-day sleep durations reliably? This is a scheduling concern for self-hosted Temporal.

---

## Verdict

**PASS-WITH-WARNINGS**

The PRD is coherent, complete, and correctly incorporates all settled design decisions from PrePlan and the session dialogue. The Temporal-first architecture is a sound design decision. The BDD acceptance criteria structure is solid and Art.8-compliant. The UX design is appropriately scoped to the Discord surface with all state transitions covered.

No finding requires a scope reset or invalidates the approach.

### Conditions

**Must be resolved before TechPlan begins:**
- HIGH-1: Record an explicit decision on inference gateway Art.10 (waiver or prerequisite story). Close OQ-2.
- HIGH-2: Verify Temporal search attribute registration and Visibility backend availability. Close OQ-1. Define SQLite fallback if Option B is unviable.

**Must be addressed before story authoring (TechPlan outputs):**
- MEDIUM-1: Define `custom_id` format with guaranteed 100-char compliance
- MEDIUM-4: Define `never-touch` resource list config location and format

**Should be incorporated in TechPlan:**
- MEDIUM-2: Auto-confirmed observations count as neutral (not correct) for graduation criteria
- MEDIUM-3: Add Prometheus/Grafana as secondary observability surface reference in UX design
