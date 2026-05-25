# ExpressPlan Adversarial Review — OmniRoute Gateway Migration

**Feature:** terminus-inference-omniroute-gateway-migration  
**Phase:** expressplan  
**Review Date:** 2026-05-25  
**Review Type:** party-mode blind-spot challenge (3-reviewer model)  
**Verdict:** PASS WITH REQUIRED ACTIONS

---

## Review Summary

The expressplan artifacts (business-plan.md, tech-plan.md, sprint-plan.md) are coherent and complete for the express track. Three independent reviewers challenged the artifacts from product/business, technical architecture, and sprint execution perspectives. No plan-blocking issues were found, but five medium/high issues are captured as required FinalizePlan inputs.

---

## Reviewer 1 — Cynic (Product/Business)

**Artifact reviewed:** business-plan.md

### Challenge 1: Auth bridge hand-wave
The business plan lists "Vault JWT authentication bridge" as in-scope but provides no substance on feasibility. The phrase "thin ingress auth adapter required" is a placeholder for an unknown effort. If the auth bridge turns out to be a new Go service (vs. an Nginx annotation), this migration involves shipping a new service, not just adopting a Docker image.

**Severity:** High  
**Finding:** ADV-1 — Auth bridge scope is unbounded without implementation path committed

### Challenge 2: Decommission gate is unclear
The success criterion says `terminus-inference-gateway` should be decommissioned after cutover, but the parallel window is 24h (only in the tech plan). If post-cutover contract validation reveals issues, the parallel window stretches indefinitely. A time-bound gate for decommission should be explicit in the business plan.

**Severity:** Low  
**Finding:** Not blocking — tech plan addresses this adequately for express track

### Challenge 3: OmniRoute config state risk minimized
The claim that OmniRoute config can be exported from its SQLite-backed data volume and re-created from YAML is asserted without validation. This is a DR assumption, not a tested pattern.

**Severity:** Medium  
**Finding:** ADV-4 — GitOps round-trip proof-of-concept needed before accepting this as a mitigation

---

## Reviewer 2 — Architect (Technical)

**Artifact reviewed:** tech-plan.md

### Challenge 1: Auth bridge implementation path deferred without preliminary recommendation
The tech plan presents two options (Nginx auth-url annotation vs. Go auth proxy) and explicitly defers resolution. Story 2.1 flags this as "must be resolved before story begins." The tech plan should provide a preliminary recommendation even if final validation is deferred.

**Preliminary recommendation (for FinalizePlan input):** Attempt Nginx `auth-url` annotation first. If a Vault `/v1/auth/token/lookup-self` endpoint is accessible from Nginx's auth-url context, annotation-based auth is preferred (no new service). Only escalate to Go proxy if Nginx auth-url validation confirms it is not viable.

**Severity:** High  
**Finding:** ADV-1 — Requires explicit resolution gate before Sprint 2

### Challenge 2: combo-batch timeout with parallel_slots=1
`combo-batch` timeout is listed as `0 (caller-enforced)`. With `llamacpp parallel_slots=1`, a long-running chat request blocks any concurrent batch request at the llamacpp queue level. OmniRoute's queue handling under this constraint is uncharted. There is no acceptance criterion for maximum acceptable queue wait.

**Severity:** Medium  
**Finding:** ADV-4-b — Sprint 1 Story 1.1 or Story 1.3 should include a llamacpp queue-depth test acceptance criterion

### Challenge 3: OmniRoute config GitOps claim needs validation
OmniRoute's persistence layer is SQLite-based. "Export config as committed YAML" assumes OmniRoute has a stable import/export mechanism. This must be validated in Sprint 1 before being treated as a mitigation.

**Severity:** Medium  
**Finding:** ADV-4

### Challenge 4: Architecture.md delivery timing vs. Org-3 gate
Architecture.md is a Story 3.1 deliverable (last sprint, pre-cutover). Org-3 requires architecture documentation before dev phase begins. Moving it to Sprint 3 risks it being in-flight when the dev phase gate check occurs.

**Recommendation:** Add architecture.md stub as a FinalizePlan deliverable. Full architecture.md can be finalized in Sprint 3 (when auth bridge implementation is known), but a stub covering the known decisions must exist before dev phase begins.

**Severity:** Medium  
**Finding:** ADV-2 — Architecture.md stub required as FinalizePlan output, not Sprint 3

---

## Reviewer 3 — Scrum Master (Execution)

**Artifact reviewed:** sprint-plan.md

### Challenge 1: Story 1.3 covers all 5 combos — too wide
All five route profile combos plus the route-profiles manifest are combined into a single story. If combo-batch strict-fail configuration is ambiguous in OmniRoute's UI, it blocks all five combos from marking the story complete. Recommend splitting: core combos (chat, interactive, triage) as Story 1.3a, specialized combos (batch with strict-fail, code-completion) as Story 1.3b.

**Severity:** Low  
**Finding:** ADV-3 — Sprint execution risk; defer split decision to FinalizePlan

### Challenge 2: Auth bridge uncertainty should be a Sprint 1 spike, not Sprint 2 story
Story 2.1 flags "unresolved design decision" mid-sprint. This uncertainty should be de-risked in Sprint 1 as a spike/timebox (e.g., "2-hour spike: validate Nginx auth-url against Vault JWT endpoint"), not discovered as a blocker mid-Sprint 2.

**Severity:** High  
**Finding:** ADV-1 — Add auth bridge spike to Sprint 1; resolve before Sprint 2 begins

### Challenge 3: Chore C-1 (Article 16 .todo approval) has no sprint assignment or reviewer
The .todo entry has been created but is pending review approval. The sprint plan lists it as a chore with no sprint assignment. Since approval is required before the dev phase begins, it should be explicitly assigned to Sprint 1 with a named reviewer and an acceptance criterion.

**Severity:** Medium  
**Finding:** ADV-5 — Chore C-1 needs sprint assignment and reviewer

### Challenge 4: Chore C-2 has no sprint assignment
Minor: Dependabot/Renovate rule for OmniRoute version pin notification has no sprint assignment.

**Severity:** Low  
**Finding:** Minor; assign to Sprint 1 alongside Story 1.1

---

## Consolidated Findings

| Finding | Severity | Type | Resolution Path |
|---------|----------|------|----------------|
| ADV-1: Auth bridge implementation path unresolved; must be committed before Sprint 2 | High | Planning gap | Add auth bridge spike to Sprint 1; resolve in FinalizePlan before dev begins |
| ADV-2: Architecture.md delivery in Sprint 3 may miss Org-3 gate | Medium | Compliance risk | Produce architecture.md stub in FinalizePlan; full doc in Sprint 3 |
| ADV-3: Story 1.3 too wide — batch strict-fail risk should be isolated | Low | Sprint risk | Consider splitting in FinalizePlan |
| ADV-4: OmniRoute GitOps round-trip claim needs validation; combo-batch queue depth under parallel_slots=1 | Medium | Technical assumption | Add proof-of-concept acceptance criteria to Sprint 1.1/1.3 |
| ADV-5: Article 16 .todo approval gate has no sprint owner or reviewer | Medium | Compliance gate | Assign Chore C-1 to Sprint 1 with named reviewer |

---

## Verdict

**PASS WITH REQUIRED ACTIONS**

No plan-blocking issues. All five findings are surfaced as required inputs for FinalizePlan. The expressplan artifacts satisfy the express-track gate for phase advance.

**Condition for phase-complete:** Findings ADV-1, ADV-2, and ADV-5 must be explicitly addressed in FinalizePlan before the dev phase begins. ADV-3 and ADV-4 are implementation-phase inputs.

---

*Review conducted: 2026-05-25*  
*Reviewer model: party-mode 3-reviewer blind-spot challenge*
