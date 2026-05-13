---
feature: terminus-watchdog-llm-engine
doc_type: prd
status: review-complete
goal: "Add an LLM reasoning step to terminus-watchdog that classifies novel alerts via the inference gateway and records triage history in Temporal for operator-assisted pattern learning."
key_decisions:
  - Temporal-first observation store (Option B) — no SQLite, no PVC, no aiosqlite
  - E2 scope: batch analysis pool for dev agent, not automated graduation
  - Art.8 hard gate: LLM integration stories require BDD Given/When/Then acceptance tests
  - Circuit-breaker: gateway unreachable → 5s fast-fail, fallback to MVP1-style alert (no LLM enrichment)
  - Silent-tier verdict pathway: auto-confirm silent observations after 7 days without contradiction (accepted risk — 7d timer reliability not verified; not blocking)
  - Temporal workflow history retention: 30-day namespace retention policy
  - Elasticsearch deployment added to scope as DEP-4 (prerequisite for Option B Temporal Visibility)
  - never-touch resource list in Helm values (never_touch config section)
  - E2 graduation success criterion: surfaced in batch analysis report; graduation action deferred if no qualifying pattern exists within first window
open_questions:
  - OQ-1: CLOSED — Elasticsearch will be deployed as DEP-4; Temporal search attribute registration feasible once Elasticsearch Visibility backend is in place
  - OQ-2: CLOSED — Option B selected: inference gateway Art.10 story scoped as prerequisite in TechPlan; blocks dev start, not TechPlan authoring
depends_on:
  - terminus-inference-gateway (Art.10 observability gap — see Cross-Feature Dependencies)
  - terminus-infra (triage routing profile in ROUTING_CONFIG_PATH)
blocks: []
updated_at: 2026-05-13T14:00:00Z
stepsCompleted: [step-01-init, step-02-discovery, step-03-problem, step-04-goals, step-05-epics, step-06-requirements, step-07-acceptance, step-08-constraints, step-09-dependencies, step-10-review]
---

# Product Requirements Document — Watchdog LLM Engine

**Feature:** `terminus-watchdog-llm-engine`  
**Service:** `terminus/watchdog`  
**Track:** `full`  
**Owner:** Todd Hintzmann  
**Date:** 2026-05-13  
**Status:** Draft — pending BusinessPlan adversarial review

---

## 1. Problem Statement

### Background

terminus-watchdog MVP1 is a production Python asyncio service running 11 hardcoded detection patterns against ArgoCD REST, Temporal gRPC, Kubernetes Watch, and Loki HTTP. It routes Discord alerts reliably for known patterns. Its fundamental limitation: it only knows what it was programmed to know.

The daily operational pain is not catastrophic failure — it is **silent coverage gaps**. Novel errors (new service versions, Loki patterns outside known signatures, ArgoCD sync errors slightly different from the 11 expected forms) are invisible. The operator discovers them manually, often minutes or hours after they occurred.

### The Gap

A rule-based detector tells you something is wrong. It cannot tell you:
- **Why** — what is the causal hypothesis?
- **What to do** — is this a transient glitch, config drift, resource pressure, or something requiring human judgment?
- **How confident are we?** — is this signal worth waking someone up?

The 11 MVP1 patterns are precise but brittle. Every novel error is a coverage miss until a developer writes a new detector.

### The Opportunity

LLMs can read a context bundle of logs, resource state, and error messages and return a structured classification in under two seconds. The terminus-inference-gateway is already running in the cluster with transparent provider failover (local Qwen → Anthropic → OpenAI). The watchdog does not need new AI infrastructure — it needs a reasoning step and a feedback loop.

---

## 2. Goals

### Primary Goal

Insert an LLM reasoning step between alert detection and Discord routing. When the watchdog observes an alert with no matching hardcoded detector, the LLM classifies it: what kind of problem, a causal hypothesis, recommended action, and action tier (silent / notify / escalate / never-touch). The watchdog routes accordingly.

### Secondary Goal

Record every LLM triage call as a Temporal observation workflow. Operators confirm or reject classifications via Discord. Over time, this produces an auditable evidence pool that developers can query to identify patterns ready for promotion to hardcoded detectors.

### Non-Goals

- **Not a general assistant.** LLM classifies ops alerts only. Personal assistant use cases are a separate initiative.
- **Not new AI infrastructure.** The inference gateway exists; this feature adds a caller.
- **Not autonomous beyond defined safe patterns.** The never-touch tier enforces human confirmation for all destructive operations.
- **Not a replacement for hardcoded detectors.** LLM classification is the on-ramp; hardcoded detectors remain the end state for well-understood patterns.
- **Not a real-time webhook pipeline.** E3 (ArgoCD Notifications webhook) is a post-MVP enhancement; polling is correct for MVP.
- **Not a system prompt version manager.** System prompt versioning is a separate future feature.
- **Not multi-operator.** Single-operator Discord approval flow; multi-user safety is deferred.

---

## 3. User Stories (Epic Summary)

### E5 — Gateway Wiring (critical path, sprint 1)

**As the** watchdog operator,  
**I want** the watchdog to call the inference gateway with a `triage` routing profile  
**so that** alert context can be classified by the LLM using the existing provider failover stack.

**Scope:** Wire `httpx` async client to `POST /v1/chat/completions`. Add `TriageClient` wrapper with Pydantic schema validation and `needs-investigation` fallback on parse error. Add ESO-managed `INFERENCE_GATEWAY_TOKEN` secret. Add dependency story to terminus-infra for `triage` routing profile in `ROUTING_CONFIG_PATH`.

**Acceptance Criteria (BDD — Art.8 hard gate):**

```
Given the inference gateway is healthy and returns a valid JSON classification
When the watchdog calls TriageClient.classify(alert_context)
Then the watchdog receives a TriageResponse with populated classification, confidence, hypothesis, action_tier, and evidence fields

Given the inference gateway returns syntactically valid JSON that fails Pydantic schema validation
When the watchdog calls TriageClient.classify(alert_context)
Then the watchdog receives a TriageResponse with classification="needs-investigation", confidence=0.0, action_tier="escalate"
And the detection loop does not crash or block

Given the inference gateway is unreachable (connection refused, DNS failure)
When the watchdog calls TriageClient.classify(alert_context) with a 5-second timeout
Then the watchdog fast-fails after 5 seconds (not 30s httpx default)
And the alert is routed as an MVP1-style notification with no LLM enrichment
And a gateway_unreachable event is recorded in the observation Temporal workflow
And the detection loop continues normally

Given the inference gateway returns a degraded=true signal (provider fallback active)
When the watchdog processes the TriageResponse
Then the response is used normally and the degraded flag is logged in the observation record
```

---

### E1 — LLM Analysis Foundation (critical path, sprint 1)

**As the** watchdog operator,  
**I want** the watchdog to classify novel alerts using LLM inference  
**so that** I receive a structured hypothesis and recommended action in Discord within 60 seconds of detection, instead of seeing nothing.

**Scope:** Integration of `TriageClient` (E5) into the detection loop. Context bundle assembly (alert text, Loki snippets, ArgoCD/Temporal resource state). Classification routing: `silent` → observation log only; `notify` → Discord embed no buttons; `escalate` → Discord embed with approval buttons; `never-touch` → Discord embed with escalation only, no approval option. Rate-limiting: asyncio Semaphore capping concurrent LLM calls at 3. Alert-storm protection: queue-and-drain for detection loop responsiveness.

**Acceptance Criteria (BDD — Art.8 hard gate):**

```
Given a novel alert arrives that matches no hardcoded detector
When the LLM classification returns action_tier="escalate"
Then a Discord embed is sent with classification, confidence, hypothesis, evidence, and Approve/Reject buttons
And a Temporal triage workflow is started with the observation context
And the detection loop completes the alert within 60 seconds of initial detection

Given a novel alert arrives that matches no hardcoded detector
When the LLM classification returns action_tier="silent"
Then no Discord message is sent
And a Temporal triage workflow is started with observation context and auto-confirm=true
And the detection loop continues without operator input

Given a novel alert arrives with action_tier="never-touch"
When the LLM classifies the alert
Then a Discord embed is sent with classification and hypothesis, with no Approve button
And the Discord message is marked as requiring human investigation only
And the Temporal workflow waits for a manual signal with no auto-approval path

Given 25 novel alerts arrive simultaneously (simulated alert storm)
When the asyncio Semaphore (max_concurrent=3) is at capacity
Then remaining triage calls are queued and processed in order
And all 25 alerts eventually produce Discord messages or observation records
And the detection loop heartbeat continues firing during the queue drain

Given the LLM call succeeds but the Temporal workflow fails to start
When TriageClient returns a valid TriageResponse
Then the alert is routed via Discord as a notify (not silently dropped)
And the Temporal failure is logged at ERROR level with the alert context

Given the watchdog restarts while open triage workflows are running
When the bot starts up
When the Temporal client queries for open (RUNNING) triage workflows
Then all open workflows with pending human signals have their Discord views re-registered
And the Approve/Reject buttons for existing escalation messages remain functional
```

**Security constraints (Art.9):**

- Alert context bundles must never contain credential values, API tokens, or secret key material. A scrubbing pass is required before the context bundle is sent to the gateway.
- The `INFERENCE_GATEWAY_TOKEN` must be loaded from the ESO-managed secret at startup; it must never be logged.
- The LLM system prompt must not include any information about production credentials, secret paths, or vault paths.

**AI safety constraints (Art.15):**

- The `never-touch` tier is a hard boundary enforced in code, not by LLM instruction. No LLM output can produce an auto-execute action for resources designated `never-touch` in the watchdog config.
- All LLM-generated remediation actions (if approved via Discord) must be logged to Temporal workflow history before execution.

---

### E4 — Interactive Remediation Wiring (sprint 2)

**As the** watchdog operator,  
**I want** to approve or reject a watchdog remediation action from Discord  
**so that** I can authorize safe infrastructure changes without leaving Discord.

**Scope:** Discord persistent views (`timeout=None`) with Approve/Reject buttons. `custom_id` encodes Temporal `workflow_run_id`. `on_interaction()` calls `get_workflow_handle(workflow_run_id).signal("human_response", payload)`. Bot startup re-registers open views by querying Temporal for RUNNING triage workflows. Idempotency: duplicate signals (two users clicking Approve) are guarded by a Temporal workflow state check before acting.

**Acceptance Criteria (BDD — Art.8 hard gate):**

```
Given a watchdog triage workflow is waiting for human_response
When the operator clicks Approve in the Discord embed
Then the Temporal workflow receives a human_response signal with verdict="approve"
And the workflow executes the recommended remediation action
And the Discord embed is updated to show "Approved by operator"
And the triage workflow records the verdict in the observation search attributes

Given a watchdog triage workflow is waiting for human_response
When the operator clicks Reject in the Discord embed
Then the Temporal workflow receives a human_response signal with verdict="reject"
And no remediation action is executed
And the Discord embed is updated to show "Rejected — no action taken"

Given a triage workflow is in RUNNING state waiting for human_response
When the watchdog pod restarts
Then on startup, the bot queries Temporal for all RUNNING triage workflows
And re-registers Discord persistent views for each open workflow using the stored workflow_run_id
And the Approve/Reject buttons continue to function for pre-restart messages

Given the operator has already approved an escalation
When a second user clicks Approve on the same Discord message
Then the Temporal workflow ignores the duplicate signal (idempotency check passes)
And no double remediation is executed

Given a triage workflow has been waiting for human_response for more than 24 hours
When the workflow timeout fires
Then the workflow closes with status=expired
And the Discord message is updated to show "Approval window expired — no action taken"
And the observation record is marked with verdict=expired
```

---

### E2 — Pattern Learning Feedback Loop (sprint 3)

**As the** watchdog operator,  
**I want** a queryable history of every LLM triage call with human verdicts  
**so that** developers can identify patterns ready for promotion to hardcoded detectors.

**Scope — Temporal-first (Option B):**

All observation data is stored as Temporal workflow state via search attributes. No SQLite, no PVC, no aiosqlite dependency. The Temporal `TriageWorkflow` carries all triage metadata as search attributes:

| Search Attribute | Type | Description |
|---|---|---|
| `LLMClassification` | Keyword | classification enum value |
| `LLMConfidence` | Float | confidence score 0.0–1.0 |
| `ActionTier` | Keyword | silent / notify / escalate / never-touch |
| `PatternId` | Keyword | alert source pattern identifier |
| `AlertSource` | Keyword | argocd / temporal / k8s / loki |
| `HumanVerdict` | Keyword | correct / incorrect / expired / NULL |
| `PatternGraduated` | Bool | true when promoted to hardcoded detector |
| `AlertTs` | Datetime | alert timestamp |

Batch analysis queries use the Temporal Visibility API (`list_workflows` with search attribute filters). At homelab scale (150–900 observations per 30-day window), in-memory aggregation over the result set is sufficient. No SQL required.

**Silent-tier verdict pathway:** Silent-tier triage workflows auto-confirm after 7 days if no operator contradiction is recorded. This allows silent observations to accumulate human-equivalent verdicts for batch analysis without Discord prompts. *Accepted risk: 7-day Temporal sleep timer reliability on self-hosted deployment is not pre-verified. If timer misfires, the worst outcome is an auto-confirm that never fires — the observation remains unvoted, which is equivalent to the current (no-observation) state. Ops runbook entry required in TechPlan.*

**Graduation process — batch/manual (not automated):**
1. Operator or scheduled script queries Temporal for completed TriageWorkflows meeting criteria: ≥5 occurrences, ≥0.90 avg confidence, HumanVerdict=correct or auto-confirmed, 0 incorrect verdicts, within 30-day window.
2. Results surfaced as a Discord report (not auto-acted upon).
3. Developer reviews and writes a `BaseDetector` subclass — this is the dev-agent handoff.
4. After detector is deployed, the matching Temporal workflows are updated: `PatternGraduated=true`.

**E2 success criterion:** After 30 days of production use, at least one novel pattern has met graduation criteria and is *surfaced in the batch analysis report*. The graduation action (writing the hardcoded detector) is deferred if no qualifying pattern is available within the first observation window — this feature delivers the tooling; the detector authoring is a separate dev action. *HumanVerdict=auto-confirmed counts as neutral (not confirmed correct) for graduation threshold calculations. Only HumanVerdict=correct counts toward the ≥5 occurrences threshold.*

**Temporal history retention:** 30-day retention policy on the Temporal namespace is set at the server level (ops runbook entry). Graduated workflows may be archived before pruning at the operator's discretion.

**Acceptance Criteria (BDD — Art.8 hard gate):**

```
Given a TriageWorkflow completes with classification="config-drift", confidence=0.92, action_tier="escalate"
When the operator marks the verdict as "correct" via Discord
Then the workflow search attribute HumanVerdict is updated to "correct"
And the observation is queryable via Temporal Visibility with HumanVerdict=correct

Given a TriageWorkflow completes with action_tier="silent"
When 7 days pass without a contradiction verdict from the operator
Then the workflow auto-confirms the observation (HumanVerdict="auto-confirmed")
And the observation is included in batch analysis queries

Given the operator requests a batch analysis report
When the developer queries Temporal Visibility for workflows meeting graduation criteria
Then the query returns all workflows where:
  - LLMClassification is consistent (same PatternId)
  - occurrence_count >= 5 within 30 days
  - avg(LLMConfidence) >= 0.90
  - no HumanVerdict="incorrect" in the set
And the results are returned as a structured list for developer review

Given 30 days have elapsed since a TriageWorkflow started
When the Temporal namespace retention policy fires
Then workflows older than 30 days are pruned
And PatternGraduated=true workflows may be archived first per ops runbook
```

---

### E3 — ArgoCD Notifications Integration (post-MVP, deferred)

Deferred. Polling (30-second interval) is the correct choice for MVP at homelab scale. E3 would add an inbound webhook receiver (`/webhook/argocd` route on existing aiohttp server, port 9090) when detection latency requirements demand sub-5-second response. Deferred until E1 and E2 are validated in production.

---

## 4. Functional Requirements

### FR-1 — Triage Client

- `TriageClient` MUST implement `classify(alert_context: AlertContext) -> TriageResponse` as an async method
- MUST use `httpx` async client with a 5-second timeout (not the httpx default 30s)
- MUST call `POST /v1/chat/completions` on the inference gateway with `response_format: {"type": "json_object"}` and temperature 0.1
- MUST validate the response against `TriageResponse` Pydantic model; on validation failure, MUST return a safe `needs-investigation` fallback response, never raise
- MUST record `gateway_unreachable` in the Temporal observation workflow when the gateway is not reachable
- MUST NOT log prompt text, alert raw text, or response content at any log level
- MUST scrub alert context of any credential-matching patterns before sending to gateway (Art.9)

### FR-2 — Classification Schema

Six-value classification enum:

| Value | Meaning |
|---|---|
| `transient-glitch` | Error likely self-resolves; monitor only |
| `known-degraded-dependency` | Upstream dependency issue; watchdog cannot remediate |
| `config-drift` | ArgoCD OutOfSync or live drift; approve-to-sync appropriate |
| `resource-pressure` | Pod OOM, eviction, CPU throttle |
| `needs-investigation` | LLM cannot classify confidently; human must inspect |
| `no-action` | Alert is noise; suppress recommended |

Four-value action tier enum:

| Value | Meaning |
|---|---|
| `silent` | Log to Temporal observation workflow only; no Discord message |
| `notify` | Send Discord embed; no remediation buttons |
| `escalate` | Send Discord embed with Approve/Reject buttons |
| `never-touch` | Human-protected resource; always escalate; no auto-approval path |

**never-touch resource configuration:** The set of never-touch resources is defined in Helm values under a `never_touch` config section (list of resource patterns: type, namespace/name regex). This config is mounted as a watchdog ConfigMap and loaded at startup. The `TriageWorkflow` checks this list before any action tier routing — the never-touch designation is enforced in workflow code, not by LLM instruction (Art.15). Config format and key structure are defined in TechPlan.

### FR-3 — Observation Workflow

- Every LLM triage call MUST start a Temporal `TriageWorkflow` with search attributes as defined in E2
- `TriageWorkflow` MUST be idempotent on the (alert_ts, pattern_id) key — no duplicate workflows for the same alert event
- Silent-tier workflows MUST auto-confirm after 7 days if no human verdict is recorded (accepted risk: timer reliability not pre-verified; ops runbook must document manual verification step)
- Approval-window workflows MUST expire after 24 hours if no human signal is received
- Workflow IDs MUST be stable across pod restarts for open workflows (encode alert_ts + pattern_id in workflow ID)
- Search attribute registration MUST be performed as a pre-deployment ops step via `tctl namespace search-attribute add` (prerequisite: DEP-4 Elasticsearch Visibility backend deployed)

### FR-4 — Discord Integration

- Discord approval embeds MUST include: classification, confidence (as percentage), hypothesis, action tier, evidence array, and Approve/Reject buttons for `escalate` tier
- `custom_id` for buttons MUST encode the Temporal `workflow_run_id` for durable signal routing
- Persistent views (`timeout=None`) MUST be re-registered at bot startup for all RUNNING TriageWorkflows
- Duplicate signals (two users clicking the same button) MUST be idempotency-checked before executing remediation

### FR-5 — Rate Limiting

- Concurrent LLM calls MUST be capped at 3 via asyncio Semaphore
- The detection loop heartbeat MUST continue firing during LLM queue drain (non-blocking)
- Queued triage calls MUST be processed FIFO

### FR-6 — Internal Infrastructure Primacy (Art.16)

- All LLM inference MUST route through the terminus-inference-gateway
- Direct calls to OpenAI, Anthropic, or any cloud LLM API from the watchdog are prohibited
- The `triage` routing profile in the gateway's `ROUTING_CONFIG_PATH` defines the provider stack; watchdog does not manage provider configuration

---

## 5. Non-Functional Requirements

### NFR-1 — Latency
- Alert-to-Discord pipeline for triage-eligible alerts: ≤ 60 seconds end-to-end
- LLM classification adds ≤ 5 seconds to the pipeline when local Qwen is healthy
- Gateway unreachability fast-fail: ≤ 5 seconds (not 30s httpx default)

### NFR-2 — Reliability
- LLM path failures (gateway down, parse error, Temporal start failure) MUST NOT degrade MVP1 detection patterns
- The detection loop MUST continue operating at normal polling frequency regardless of LLM queue depth

### NFR-3 — Observability
- All triage calls MUST be logged at INFO level with: alert source, LLM classification result, action tier, latency_ms, and provider_used
- Prometheus `ServiceMonitor` MUST expose `watchdog_llm_triage_calls_total` (by classification, tier) and `watchdog_llm_triage_latency_seconds` (histogram)
- Gateway unreachability events MUST be logged at WARN level

### NFR-4 — Security (Art.9)
- Alert context bundles must not contain credential values, API tokens, or secret key material
- `INFERENCE_GATEWAY_TOKEN` must be loaded from ESO-managed secret and must never be logged
- LLM system prompt must not reference production secret paths or vault paths

### NFR-5 — AI Safety (Art.15)
- `never-touch` tier is enforced in workflow code, not by LLM instruction
- All LLM-approved remediation actions are logged to Temporal history before execution
- Temporal workflow is the authority on what action was taken; LLM output is a classification input only

---

## 6. Cross-Feature Dependencies

### DEP-1 — Inference Gateway Art.10 Observability

**Target feature:** terminus-inference-gateway  
**Issue:** The gateway architecture defers Art.10 compliance (route id, duration, token counts, outcome, cost tag per request) as a TODO.  
**Owner:** Todd  
**Decision (Option B):** The inference gateway Art.10 story will be scoped into TechPlan as a prerequisite before dev begins. This feature cannot enter dev phase until the gateway Art.10 story is merged. The terminus-watchdog-llm-engine TechPlan must include a gateway Art.10 prerequisite story as a named dependency on the dev critical path.  
**Status:** CLOSED — decision recorded. TechPlan must define and sequence the gateway Art.10 story.  
**Blocking?** Blocks dev start; does not block TechPlan authoring.

### DEP-2 — Terminus-Infra Triage Routing Profile

**Target feature:** terminus-infra (platform ops)  
**Issue:** The `triage` routing profile in `ROUTING_CONFIG_PATH` is owned by terminus-infra, not terminus-watchdog. E5 cannot be tested in production until this profile exists.  
**Owner:** platform ops  
**Resolution condition:** terminus-infra story added and merged before E5 dev story is worked.  
**Blocking?** Blocks E5 and therefore the entire feature critical path.

### DEP-3 — Temporal Search Attribute Registration

**Target:** Temporal server admin  
**Issue:** Temporal custom search attributes must be registered via `tctl` admin command on the Temporal namespace before TriageWorkflow can use them. This requires a Visibility backend that supports custom attribute indexing (see DEP-4).  
**Owner:** platform ops / Todd  
**Resolution condition:** DEP-4 (Elasticsearch) deployed and healthy; then run `tctl namespace search-attribute add` for each attribute in the E2 table. Add to TechPlan infrastructure checklist as a pre-dev prerequisite step.  
**Status:** Unblocked pending DEP-4 completion.

---

### DEP-4 — Elasticsearch Visibility Backend for Temporal (NEW — in scope)

**Target:** terminus-infra  
**Issue:** Option B (Temporal-first observation store) requires Temporal Visibility API with custom search attribute filtering. Self-hosted Temporal with basic visibility only (no Elasticsearch) does not support `list_workflows` filtered by custom attributes. Elasticsearch must be deployed as the Temporal Visibility backend to make Option B viable.  
**Owner:** Todd / terminus-infra  
**Scope decision:** Elasticsearch deployment is added to this feature's scope (or as a companion terminus-infra story). It is on the operator's infrastructure backlog and is being pulled forward as a prerequisite.  
**Resolution condition:** Elasticsearch deployed and configured as Temporal Visibility backend; Temporal server updated to route search attribute queries to Elasticsearch; smoke test: `tctl namespace search-attribute add` succeeds and a test workflow is findable by custom attribute.  
**Blocking?** Blocks DEP-3 and therefore E2 (Option B batch analysis). E5 and E1 are not blocked by DEP-4.

---

## 7. Out of Scope

- SQLite observation log (superseded by Temporal-first Option B)
- PVC for watchdog (not required; no persistent storage needed)
- aiosqlite dependency (not required)
- Automated pattern graduation (patterns are surfaced for human review; dev agent writes the detector)
- System prompt version management (future feature)
- Multi-user Discord safety controls (known risk; deferred)
- ArgoCD Notifications webhook (E3 — post-MVP)
- OpenClaw or Hermes Agent integration (separate personal assistant initiative)
- Direct cloud LLM API calls (prohibited by Art.16)

---

## 8. Open Questions

| # | Question | Owner | Needed By | Status |
|---|---|---|---|---|
| OQ-1 | Can Temporal search attributes be registered on the current Temporal deployment? | Todd / platform ops | TechPlan | **CLOSED** — Elasticsearch being added as DEP-4; Option B confirmed viable once DEP-4 completes |
| OQ-2 | What is the resolution to DEP-1 (inference gateway Art.10)? | Todd | TechPlan start | **CLOSED** \u2014 Option B: gateway Art.10 story scoped as TechPlan prerequisite; blocks dev start, not TechPlan |
| OQ-3 | What is the pinned `temporalio` SDK version in watchdog `pyproject.toml`? Must be ≥ 1.3.0 for `signal()` with payload. | Dev story E4 | TechPlan | Open |
| OQ-4 | What Temporal namespace retention policy is currently configured? Is 30-day feasible? | Todd / ops | TechPlan | Open |
