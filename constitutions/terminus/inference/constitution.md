# Service Constitution: terminus-inference

**Layer:** service
**Name:** inference
**Parent:** terminus
**Ratified:** 2026-04-03T00:00:00Z
**Status:** active

---

## Preamble

This constitution governs the `inference` service within the terminus domain. The terminus-inference service is the local AI inference gateway for the terminus platform: it accepts requests from platform consumers, selects and routes to locally hosted models via LiteLLM, and returns normalized responses under a stable contract. This constitution inherits all org-level and terminus-domain rules and adds service-specific implementation controls for route profiles, contract safety, batch execution, guardrails, and rollout mechanics.

---

## Inheritance Validation Record

This service constitution is strictly additive relative to the org and terminus domain constitutions. It does not remove, narrow, weaken, or create exceptions to any parent article.

---

## Articles

### Article 1: Route Profile Completeness

**Rule:** Every route exposed by the terminus-inference service must have a fully declared route profile before the route is implemented or promoted. A route profile must include at minimum: route identifier, model target (or model selection policy), request timeout, retry policy (count and backoff), escalation behavior on exhausted retries (error or fallback route), and cost tag. Routes without complete profiles are not permitted in any deployed audience tier. A route that lacks any required profile field is treated as misconfigured and must not serve traffic.

**Rationale:** Routes without explicit profiles shift operational decisions to implicit runtime behavior — timeouts default to system limits, retries default to none, escalation defaults to unhandled errors. In a gateway that may be invoked by latency-sensitive platform consumers and long-running batch jobs simultaneously, implicit behavior is a reliability and cost risk.

**Evidence Required:** A route profile manifest must exist in the service repository listing all routes and their complete field values. Implementation code must enforce that no route is registered without a corresponding profile entry. Architecture documentation for any new route must include the profile values as part of the route specification.

**Gate:** informational
**Status:** active

---

### Article 2: Gateway Contract Gate

**Rule:** The terminus-inference service must maintain a gateway contract test suite that verifies the stable API contract against all actively supported routes. This test suite must pass before any change to the inference service is promoted to the next audience tier. The contract test suite must cover: request schema validation, response schema validation, error response shape, and any normalization guarantees (such as model-agnostic response field mappings). A promotion that cannot produce a passing contract test run is blocked.

**Rationale:** The terminus domain constitution (Article 9) requires a single stable API contract for all consumers. The gateway contract test is the enforcement mechanism that makes that rule verifiable. Without a passing test gate, contract stability is aspirational rather than enforced.

**Evidence Required:** The contract test suite must exist in the service repository and must be runnable in CI. Promotion artifacts must include confirmation that the contract test suite passed for the build being promoted. When a breaking change is introduced, the contract version must increment and the test suite must be updated before the change can be promoted.

**Gate:** informational
**Status:** active

---

### Article 3: Batch Safety Gate

**Rule:** Any batch execution route — defined as a route intended to process multiple items sequentially or in parallel under a single job invocation — must pass a batch safety gate before promotion. The batch safety gate must verify: (a) the job emits checkpoint events at defined intervals (as required by terminus domain Article 10); (b) the job can resume from the most recent checkpoint without reprocessing completed items (resume-on-failure smoke test); (c) the job enforces a maximum item count or maximum runtime limit (see Article 4). A batch route that has not passed this gate may not serve production traffic.

**Rationale:** Batch jobs without checkpointing can lose hours of work to a single transient failure and, on resume, either reprocess everything or fail silently on partial data. Without a resume smoke test, checkpoint logic is untested theory. Without an enforced ceiling, a batch job can consume unbounded compute and cost.

**Evidence Required:** Batch routes must include a checkpoint schema and a resume contract in their route profile (Article 1). CI must include a batch safety gate step that executes the resume smoke test against a controlled input set. Promotion artifacts must confirm the gate passed for the build being promoted.

**Gate:** informational
**Status:** active

---

### Article 4: Guardrail Requirements

**Rule:** All routes in the terminus-inference service must operate under active guardrails. At minimum: (a) every route must enforce a maximum request duration (wall-clock timeout, not just model-level timeout); (b) every route must enforce a maximum output token count or iteration count appropriate to the route type; (c) batch routes must enforce a maximum job runtime and a maximum item count per invocation. Guardrails must be configuration-driven and visible in the route profile manifest (Article 1). Hard-coded limits in application logic that are not surfaced in the route profile do not satisfy this requirement.

**Rationale:** LLM inference is inherently unbounded in output volume and duration. Without explicit guardrails, a single misbehaving request or runaway batch job can exhaust GPU resources, block other consumers, and accumulate unexpected cost. Guardrails must be visible in configuration so they can be reviewed, adjusted, and audited without reading source code.

**Evidence Required:** Route profiles must include all applicable guardrail values. Implementation code must enforce guardrails from the profile values — not from separate hard-coded constants. Reviews must confirm that guardrail enforcement is tested and that no route can silently exceed its declared limits.

**Gate:** informational
**Status:** active

---

### Article 5: Controlled Rollout and Rollback

**Rule:** Any change that modifies inference behavior — including model updates, prompt template changes, routing policy adjustments, guardrail value changes, or fallback logic changes — must be deployed under a controlled rollout plan. The plan must specify: the target audience tier, rollout scope (percentage, canary, or full), success criteria (observable metrics or test outcomes that confirm correct behavior), and a rollback procedure (the exact steps to revert the change and restore prior behavior). A rollout without a documented rollback procedure is prohibited.

**Rationale:** Inference behavior changes are difficult to validate fully in pre-production because model behavior is probabilistic and consumer workloads vary. A controlled rollout limits blast radius during initial exposure. A documented rollback procedure ensures that operators and agents can restore known-good behavior without ambiguity when a change produces unexpected outcomes.

**Evidence Required:** Initiative artifacts for inference-affecting changes must include a rollout plan section with all four required elements (target tier, scope, success criteria, rollback procedure). Rollback procedures must reference specific configuration states, model tags, or deployment artifacts — not general instructions.

**Gate:** informational
**Status:** active

---

### Article 6: No Implicit Fallback on Batch Routes

**Rule:** Batch routes must not use implicit fallback behavior. If a batch route's primary model is unavailable, the batch job must fail explicitly with a logged error rather than silently falling back to an alternative model. An explicit fallback on a batch route — where the fallback model is declared in the route profile (Article 1), its output schema is compatible, and its inclusion is deliberate policy — is permitted. Silent or implicit fallback (where the route profile does not declare the fallback and the fallback was not reviewed as part of the route design) is prohibited on all batch routes.

**Rationale:** Batch jobs process large volumes of items, often overnight and unattended. A silent fallback to a different model can produce semantically incorrect results at scale — potentially consuming large amounts of compute and cost on output that will be silently wrong. An explicit failure is always preferable to a silent wrong answer in a batch context. When a fallback is genuinely appropriate, it must be a deliberate policy decision, not a default behavior.

**Evidence Required:** Batch route profiles must explicitly declare whether a fallback is permitted and, if so, which fallback model is authorized and under what condition. Implementation code for batch routes must confirm that no unlisted fallback path can be triggered. Reviews must confirm that any fallback invocations in batch routes are traceable to a declared profile entry.

**Gate:** informational
**Status:** active

---

### Article 7: Go as Preferred Implementation Language

**Rule:** Go is the preferred implementation language for services within the terminus-inference service boundary. New services, gateways, and components introduced to this service should default to Go unless a compelling, documented justification exists for an alternative. Justifications must be recorded in the initiative's architecture documentation before implementation begins. This rule does not apply retroactively to existing services implemented in other languages.

**Rationale:** Go's concurrency model (goroutines), single-binary deployment artifact, and `net/http` standard library make it structurally well-suited for API gateway and inference proxy workloads. Go is also gaining mainstream adoption within the broader organization, reducing the long-term cost of code review, onboarding, and maintenance. Standardizing on Go at the service boundary avoids a heterogeneous language mix that increases cognitive overhead for operators and developers working across the inference topology.

**Evidence Required:** Architecture documentation for any new terminus-inference component implemented in a language other than Go must include a section titled "Language Justification" explaining why Go was not selected and what properties of the alternative language are required for the use case.

**Gate:** informational
**Status:** active

---

## Ratification Record

| Date | Action | Summary |
|------|--------|---------|
| 2026-04-03T00:00:00Z | Ratified | Initial terminus-inference service constitution — 6 articles covering route profiles, gateway contract gate, batch safety, guardrails, controlled rollout/rollback, and no-implicit-fallback on batch routes |
| 2026-04-03T13:36:41Z | Amended | Added Article 7: Go as preferred implementation language for terminus-inference services |

---

## Inheritance

**Parent:** [terminus/constitution.md](/home/todd/Projects/TargetProjects/lens/lens-governance/constitutions/terminus/constitution.md)
**Children:** none
**Inheritance rule:** All child constitutions must be strictly additive. No child may remove, weaken, or create exceptions to any org-, domain-, or service-level article.
