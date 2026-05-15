---
feature: terminus-watchdog-llm-engine
doc_type: adversarial-review
phase: preplan
status: complete
verdict: pass-with-warnings
updated_at: 2026-05-13
reviewed_artifacts:
  - docs/terminus/watchdog/terminus-watchdog-llm-engine/brainstorm.md
  - docs/terminus/watchdog/terminus-watchdog-llm-engine/research.md
  - docs/terminus/watchdog/terminus-watchdog-llm-engine/product-brief.md
---

# PrePlan Adversarial Review — terminus-watchdog-llm-engine

**Date:** 2026-05-13  
**Phase:** preplan  
**Verdict:** PASS-WITH-WARNINGS  
**Finding Summary:** Critical: 0 | High: 4 | Medium: 5 | Low: 3

---

## Reviewed Artifacts

| Artifact | Status |
|---|---|
| `brainstorm.md` | ✅ Present |
| `research.md` | ✅ Present |
| `product-brief.md` | ✅ Present |

---

## Findings

### HIGH-1 — Graduation Loop Cannot Complete for Silent-Tier Classifications

**Source:** `research.md` (Q3 observation log schema, Q5 graduation criteria)

The graduation query requires `confirmed_correct = occurrence_count` — all occurrences must have a human verdict. However, observations where `action_tier = silent` produce no Discord message. The operator never sees them, never clicks Approve/Reject, and `human_verdict` remains NULL indefinitely. These rows can never satisfy the graduation criteria regardless of occurrence count or LLM confidence.

This is a structural gap in the feedback loop: the most common classification outcome (noise suppression via silent tier) is precisely the category you most want to graduate to hardcoded detectors, and it is the category that can never accumulate verdicts under the current design.

**Required resolution:** BusinessPlan must define a verdict pathway for silent-tier observations. Options include: (a) auto-confirm silent observations after N days without contradiction, (b) a periodic silent-review Discord digest that allows batch verdict entry, or (c) exclude silent-tier observations from graduation eligibility (if silent = trivially deterministic, hardcode it directly without the feedback loop).

---

### HIGH-2 — Observation Log Schema Missing `workflow_run_id` and `discord_message_id`

**Source:** `research.md` (Q3 schema, Q4 Discord wiring)

The E4 wiring stores `workflow_run_id` in the Discord button `custom_id` to survive pod restarts. The risk note in Q4 states: "persistent views must be re-registered at bot startup from open observation log entries." But the 17-field observation log schema contains no `workflow_run_id` or `discord_message_id` column. On pod restart, the bot has no way to reconstruct which Temporal workflow handle to bind to which Discord message.

**Required resolution:** TechPlan must add `workflow_run_id TEXT` and `discord_message_id TEXT` to the observation log schema. The bot startup sequence must query for open (non-resolved) escalations and re-register views using these stored IDs.

---

### HIGH-3 — Gateway Unreachability Fallback Not Defined

**Source:** `research.md` (Q1), `product-brief.md` (Technical Constraints)

The research defines fallback behavior for LLM parse failures (Pydantic validation → `needs-investigation`) and provider degradation within the gateway (BudgetTracker → next provider). But neither document addresses complete gateway unreachability: network partition, gateway pod crash, or Vault token expiry. With a 30-second `httpx` timeout per call, a down gateway will hold the asyncio task for 30 seconds per alert — during which the detection loop is blocked.

The existing MVP1 pattern wraps detector calls in `try/except Exception`; however, this only prevents crash, not blocking. A non-responsive gateway during a multi-alert event could effectively disable the watchdog for minutes at a time.

**Required resolution:** TechPlan must define circuit-breaker behavior: gateway unreachable → fall back to MVP1-style Discord alert without LLM enrichment, log a `gateway_unreachable` sentinel in the observation record, and ensure no call path blocks the detection loop for more than a configurable timeout (default: 5s fast-fail, not 30s).

---

### HIGH-4 — Inference Gateway Art.10 Production Readiness Debt Is an Undocumented Blocker

**Source:** `docs/terminus/inference/gateway/architecture.md` (Deferred Decisions section)

The terminus-inference-gateway architecture contains an explicit deferred TODO: *"domain constitution Art.10 requires route id, duration, token counts, outcome, and cost tag per request — must be addressed before production readiness."* This feature builds the inference gateway as a hard production dependency. If the gateway is constitutionally blocked from production readiness, this feature inherits that block without naming it.

**Required resolution:** BusinessPlan must surface the inference gateway Art.10 observability gap as a named cross-feature dependency with an owner, a resolution condition, and a gating decision (does this feature block on gateway production-ready status, or does it proceed with explicit acknowledgment of the gap?).

---

### MEDIUM-1 — `no-action` Used Ambiguously Across Two Semantic Layers

**Source:** `research.md` (Q2 classification enum, Q3 observation log schema)

`no-action` appears as both a **classification enum value** (meaning: alert is noise, suppress) and as a value in the observation log `action_taken` column. These overlap but are semantically distinct: `action_taken` should record what the watchdog actually did in response to the triage result, which can be `no-action` even for a `config-drift` classification if the pattern is currently suppressed. The dual use creates ambiguity in graduation queries and Discord routing logic.

**Resolution:** BusinessPlan should define `action_taken` as a separate derived field that maps from the combination of `llm_classification` and current suppression state, distinct from the classification enum.

---

### MEDIUM-2 — Prompt Design Does Not Include Few-Shot Examples

**Source:** `research.md` (Q2 system prompt design principles)

The system prompt design principles cover schema declaration, context priming, evidence chain-of-thought, and failure mode instruction — but no few-shot examples. Local Qwen 3.27B is a mid-size model; enum adherence without examples is unreliable for classification tasks. Temperature=0.1 reduces randomness but does not prevent the model from emitting valid-but-wrong enum values (e.g., `"needs_investigation"` instead of `"needs-investigation"`, or `"config_drift"` with underscores). The Pydantic validation layer will catch schema mismatches, but only after a round-trip to the gateway.

**Resolution:** TechPlan system prompt specification must include at least 2-3 classification examples (one per archetypal pattern: ArgoCD drift, Temporal stall, transient pod restart) to anchor the enum vocabulary.

---

### MEDIUM-3 — Alert Rate Storm Not Rate-Limited on the Caller Side

**Source:** `research.md` (Q1), `brainstorm.md` (scope)

A mass-failure event (k3s node loss affecting 20+ ArgoCD apps simultaneously) could generate dozens of alert entries in a single detection cycle. The research scopes homelab volume at "5-30 triage calls/day" and does not address burst behavior. The gateway's BudgetTracker limits spend, not call concurrency. At 30s gateway timeout × N concurrent calls via asyncio, the detection loop could be saturated and the watchdog effectively unresponsive during the event it most needs to handle well.

**Resolution:** TechPlan must specify a rate-limiting mechanism on the watchdog side: either an asyncio Semaphore capping concurrent LLM calls (suggested: 3), a per-minute call quota, or a queue-and-drain pattern that preserves detection loop responsiveness.

---

### MEDIUM-4 — Pattern Stability Criterion Is Unimplementable As Stated

**Source:** `research.md` (Q5 graduation criteria)

The graduation criterion "Same `alert_text` structure across all occurrences" has no concrete definition of structural equivalence for dynamic k8s/ArgoCD output, which always includes timestamps, pod UIDs, resource versions, and commit SHAs. As stated, this criterion would never be satisfiable in practice — two ArgoCD drift alerts for the same app will differ by at least the `sync.revision` value.

**Resolution:** TechPlan must replace this criterion with a concrete structural equivalence definition: for example, "same `pattern_id` (LLM classification value) AND same `source` AND same `resource_namespace`" — or a normalized similarity threshold using field extraction rather than raw text comparison.

---

### MEDIUM-5 — Concurrent Discord Signals Produce Undefined Behavior

**Source:** `research.md` (Q4 Discord → Temporal signal wiring)

The design is architected for a single operator. The `on_interaction()` handler calls `handle.signal("human_response", {...})` with no guard against duplicate signals. If two people have access to the Discord server and both click Approve on the same message, the Temporal workflow receives two `human_response` signals. The `wait_condition(lambda: self._human_response is not None)` fires on the first; the second signal is silently queued in Temporal and may trigger unexpected behavior in a replay scenario.

**Resolution:** E4 story must specify signal idempotency: either restrict the Discord server to a single operator channel, add a `correlation_id` check so duplicate signals are discarded by the workflow, or use a Temporal `update` (which supports rejection) instead of a `signal`.

---

### LOW-1 — Three-Tier Autonomy Model Inconsistently Named Across Artifacts

The brainstorm uses "silent fix / escalate-with-options / never-touch." The product brief delivery section omits "never-touch" from the three-tier description. The research action tier introduces `notify` as a fourth tier between `silent` and `escalate`. The terminology is not aligned. This creates risk of scope confusion in BusinessPlan and story authoring.

---

### LOW-2 — PVC Provisioning Not Acknowledged as Chart Delta

MVP1 uses no persistent storage. Adding SQLite on a PVC requires a new `PersistentVolumeClaim` in the Helm chart. The product brief lists "SQLite on PVC" as a constraint but does not flag it as an infrastructure addition. Given the production-only deployment constraint, this needs explicit acknowledgment in the TechPlan chart scope.

---

### LOW-3 — Temporal SDK Version Claim Is Unverified

Research states "`temporalio >= 1.3.0` confirmed as supported" for `get_workflow_handle().signal()` with a payload dict, with no citation. TechPlan should verify against the actual pinned version in the watchdog `pyproject.toml` before committing to the E4 design.

---

## Party-Mode Blind-Spot Challenge

**Creed (Product Manager):** The brief positions pattern graduation as the mechanism that makes the LLM less necessary over time — but graduation requires an engineer to write a `BaseDetector` subclass and cut a production release. In a solo-operator homelab, one graduation = one full dev cycle. If the friction is too high, the observation log fills with qualified candidates and nothing gets promoted. The graduation loop's value depends on it actually being used. Has the cost per graduation been examined, and is the loop realistic at solo-operator scale — or is the real value just the observation log as a living audit trail?

**Perturabo (Architect):** The research chooses SQLite/aiosqlite because "no new database services" — but terminus-watchdog already writes to a Temporal backend backed by PostgreSQL. The observation log is a write-heavy, update-heavy table (every verdict updates a row) with indexed queries needed for graduation analysis. SQLite's single-writer model could become a bottleneck under concurrent asyncio writes. More importantly: Temporal workflow state is already the durability layer for this process. Why was storing observation records as Temporal workflow data (search attributes or a side-effect list) rejected in favor of a separate file-backed store?

**Artemis (QA):** The success criterion states "Pydantic validation catches 100% of malformed LLM outputs without crashing the detection loop." The MVP1 test pattern is: one test per alert path, one per suppression path, one per error path — all scoped to individual detectors. The LLM triage client is a cross-cutting component outside any single detector. Who owns the acceptance tests for the LLM path, and what does the BDD `Given/When/Then` look like for a schema-invalid response — for example, Qwen returns `{"classification": "config-drift", "confidence": "high", "action_tier": "escalate"}` where `confidence` is a string, not a float?

### Blind-Spot Questions

1. **Graduation realism:** Will you actually cut a production release when a pattern qualifies, or is the graduation loop aspirational? If the real value is the observation log as an ops record, say so in the product brief.

2. **SQLite vs Temporal state:** Did you consider Temporal workflow state or search attributes as the observation persistence layer? What drove the SQLite decision?

3. **Multi-user Discord:** What breaks if someone else with Discord server access clicks an approval button? Is that a realistic scenario for your setup?

4. **System prompt versioning:** When a new MVP1 detector is added, the system prompt's pattern table is stale until manually updated. Who owns that update, and how does it get deployed?

5. **Temporal workflow retention:** The design creates one triage workflow per alert with a 24-hour approval window. After 30 days of production use, how many completed workflows will exist in Temporal history? Is the history retention window scoped in TechPlan?

---

## Verdict

**PASS-WITH-WARNINGS**

The core approach — LLM reasoning inserted between alert detection and Discord routing, using the existing inference gateway with transparent provider failover, Temporal as the durability and approval substrate, and a feedback loop that makes the LLM incrementally less necessary — is architecturally sound. No finding invalidates the approach or requires a scope reset.

### Conditions

**Must be addressed before BusinessPlan can be accepted:**
- HIGH-1: Silent-tier verdict pathway
- HIGH-4: Inference gateway Art.10 cross-feature dependency named and owned

**Must be addressed before TechPlan can be accepted:**
- HIGH-2: Observation log schema — add `workflow_run_id`, `discord_message_id`
- HIGH-3: Gateway circuit-breaker / unreachability fallback
- MEDIUM-2: Few-shot examples in system prompt specification
- MEDIUM-3: Caller-side rate-limiting for alert storm
- MEDIUM-4: Concrete pattern stability definition
- MEDIUM-5: Concurrent Discord signal idempotency

**Should be resolved before story authoring:**
- MEDIUM-1: `no-action` semantic disambiguation
- LOW-1: Autonomy tier terminology alignment across artifacts
