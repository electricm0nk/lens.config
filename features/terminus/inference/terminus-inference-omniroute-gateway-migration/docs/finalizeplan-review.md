# FinalizePlan Review — OmniRoute Gateway Migration

**Feature:** terminus-inference-omniroute-gateway-migration  
**Phase:** finalizeplan  
**Review Date:** 2026-05-25  
**Review Type:** party-mode adversarial — phase-complete  
**Verdict:** PASS-WITH-WARNINGS

---

## Pre-Review Fixes Applied

The following accepted findings from `expressplan-adversarial-review.md` were applied to staged planning documents before bundle generation:

| Finding | Applied Fix | Document |
|---------|------------|---------|
| ADV-1: Auth bridge uncertainty — should be Sprint 1 spike, not Sprint 2 blocker | Added Spike S-1.0 to Sprint 1 with 2h timebox; output `auth-bridge-decision.md` gates Story 2.1 | sprint-plan.md |
| ADV-2: architecture.md stub required as FinalizePlan deliverable (Org-3 hard gate) | Created `architecture.md` stub with confirmed decisions; auth bridge path deferred to Spike S-1.0 / Story 3.1 | architecture.md (new) |
| ADV-4: OmniRoute GitOps round-trip claim needs validation | Added config round-trip AC and readiness probe AC to Story 1.1 | sprint-plan.md |
| ADV-4b: combo-batch queue depth under parallel_slots=1 | Added concurrent-request queue-depth test AC to Story 1.3 | sprint-plan.md |
| ADV-5: Chore C-1 (.todo approval) has no sprint assignment or reviewer | Assigned to Sprint 1; reviewer: Todd Hintzmann; acceptance criteria added | sprint-plan.md |
| Decommission consumer verification gate | Added named consumer verification (Watchdog, Kaylee, Hermes) as Story 3.2 AC before 24h window starts | sprint-plan.md |

---

## Review Findings

### Reviewer 1 — Creed (Product/Business)

**Finding FP-1: Decommission gate was time-based, not outcome-based**
*Severity:* Medium  
*Status:* APPLIED — Story 3.2 AC updated to require all three named consumers to verify at least one successful request through OmniRoute before the 24h parallel window begins.

**Finding FP-2: No OpenAI fallback cost ceiling in business plan risks**
*Severity:* Low  
*Status:* DEFERRED — The tech-plan guardrails (Section: Guardrails) include OpenAI fallback max-retry and cost tags per combo. A hard cost ceiling (e.g., $X/month max fallback spend) is not defined and requires operational visibility into OmniRoute usage analytics before it can be set meaningfully. Deferred to post-dev observation window.

---

### Reviewer 2 — Perturabo (Architect)

**Finding FP-3: architecture.md missing — Org-3 hard gate violation**
*Severity:* Critical (Org-3 hard gate)  
*Status:* APPLIED — `architecture.md` stub produced as FinalizePlan deliverable. Full finalization deferred to Story 3.1 after Spike S-1.0 confirms auth bridge path. Org-3 hard gate is now satisfied.

**Finding FP-4: OmniRoute version "TBD" — sprint-start blocker**
*Severity:* Medium  
*Status:* PARTIALLY APPLIED — Story 1.1 open question explicitly states: "Identify OmniRoute semver version to pin by checking `docker.io/diegosouzapw/omniroute` available tags." This is a required story-start action in Sprint 1.

**Finding FP-5: route-profiles.yaml format unspecified**
*Severity:* Medium  
*Status:* DEFERRED — The route-profiles.yaml is a Story 1.3 output. Its schema is implied by the combo table in `tech-plan.md`. No formal schema definition is produced at FinalizePlan; Story 1.3 acceptance criteria require the manifest to include all fields from the tech-plan combo table. An author may define the schema during implementation; a review gate confirms completeness at story acceptance.

---

### Reviewer 3 — Grimaldus (SM/Dev)

**Finding FP-6: Story 1.3 may be too wide (ADV-3 carry-over)**
*Severity:* Low  
*Status:* DEFERRED — Splitting Story 1.3 (core combos vs. specialized combos) is a dev-phase decision. The sprint-plan notes the risk; the implementing dev decides whether to split at sprint-start. No blocking action required.

**Finding FP-7: No target repo mapping in bundle (deferred from ADV-5 context)**
*Severity:* Medium  
*Status:* APPLIED — Target repo mapping table added to sprint-plan.md identifying `terminus.infra` and `terminus-inference-gateway` as primary implementation surfaces per sprint.

**Finding FP-8: Chore C-2 had no sprint assignment**
*Severity:* Low  
*Status:* APPLIED — Chore C-2 assigned to Sprint 1 with reviewer (Todd Hintzmann) and acceptance criteria.

---

## Blind-Spot Challenge Open Questions

The following questions from the party-mode challenge round are captured as story-level open questions (not plan blockers):

| Question | Assigned To | Gate |
|----------|------------|------|
| Is `gateway-vault-token` K8s secret the consumer-facing token or a service-account token? | Story 1.1 open question; must resolve before Spike S-1.0 completes | Spike S-1.0 output gate |
| OmniRoute readiness probe: delay Ready until `/health` returns 200 | Story 1.1 AC added | Story 1.1 |
| In-flight request drain during Story 3.2 ingress re-point (up to 300s for batch routes) | Story 3.2 AC updated (graceful drain before re-point) | Story 3.2 |
| Can OpenAI fallback guarantee `json_object` response format for triage route? | Deferred — test in Story 2.3 triage contract test; if OpenAI does not support json_object reliably, triage fallback must be disabled | Story 2.3 |

---

## Constitution Compliance Status

| Gate | Status | Notes |
|------|--------|-------|
| Org-3: Architecture documentation | PASS | architecture.md stub created at FinalizePlan |
| Org-4: No data exfiltration | PASS | Self-hosted OmniRoute; NetworkPolicy enforced |
| Org-7: TDD Red-Green | PASS | Auth proxy Go path: TDD AC in Story 2.1 |
| Org-8: BDD acceptance criteria | PASS | BDD scenarios in Stories 2.3, 3.2 |
| Org-9: Security first | PASS | All credentials in Vault; none committed |
| Org-16: Internal infrastructure primacy | PASS | .todo entry committed; review assigned (Chore C-1, Sprint 1) |
| Svc-1: Route profile completeness | PARTIAL | route-profiles.yaml defined as Story 1.3 output; manifest format implied by tech-plan combo table |
| Svc-4: Guardrails | PASS | Guardrail values in tech-plan and sprint-plan combo tables |
| Svc-5: Controlled rollout | PASS | Rollout plan in tech-plan; rollback procedure verified in Story 3.2 |
| Svc-6: No implicit batch fallback | PASS | combo-batch strict=no-fallback; BDD AC in Story 2.3 |

---

## Verdict

**PASS-WITH-WARNINGS**

All hard-gate violations resolved (Org-3 architecture.md stub produced). Two medium findings partially applied (FP-4 OmniRoute version, FP-5 route-profiles.yaml schema) with explicit deferral rationale. Four low/medium findings fully applied. Three blind-spot questions captured as story-level open questions.

**Bundle generation may proceed.**

---

## Intentional Deferrals

| Finding | Deferral Rationale | Resolution Gate |
|---------|-------------------|----------------|
| FP-2: OpenAI cost ceiling | Requires operational data from OmniRoute analytics before a meaningful ceiling can be set | Post-dev observation window |
| FP-5: route-profiles.yaml formal schema | Author defines during Story 1.3 implementation; review gate at story acceptance | Story 1.3 acceptance |
| FP-6: Story 1.3 split | Dev-phase decision at sprint-start | Dev phase |
| Blind-spot 4: OpenAI triage json_object | Test in Story 2.3; disable fallback if OpenAI doesn't guarantee format | Story 2.3 |

---

*Review conducted: 2026-05-25*  
*Reviewer model: party-mode 3-reviewer adversarial*
