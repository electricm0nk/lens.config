---
feature: terminus-watchdog-llm-engine
doc_type: product-brief
status: draft
updated_at: 2026-05-13
inputDocuments:
  - docs/terminus/watchdog/terminus-watchdog-llm-engine/brainstorm.md
  - docs/terminus/watchdog/terminus-watchdog-llm-engine/research.md
---

# Product Brief — Watchdog LLM Engine (terminus-watchdog MVP2)

**Owner:** Todd Hintzmann  
**Date:** 2026-05-13  
**Feature:** `terminus-watchdog-llm-engine`  
**Service:** `terminus/watchdog`  
**Track:** `full`

---

## The Problem

terminus-watchdog MVP1 monitors the terminus homelab reliably — but only for failures it already knows about. Its 11 hardcoded detection patterns cover ArgoCD drift, Temporal stalls, k3s node pressure, and a handful of known error signatures. Anything outside that set is invisible.

The real operational pain is not a single catastrophic failure. It is the slow accumulation of novel failures that no rule covers: a new service behaving unexpectedly after a version bump, a Loki log pattern that doesn't match any detector, an ArgoCD sync error that is slightly different from the known one. These failures go unnoticed until the operator manually discovers them — often minutes or hours later.

The watchdog sees everything. It currently understands only 11 things.

---

## The Opportunity

Large language models can read a context dump of logs, resource state, and error messages and return a structured hypothesis about what is wrong and what to do — in under two seconds. This is exactly the reasoning gap the watchdog needs to fill.

The value is not general AI autonomy. It is a narrow, high-confidence reasoning step inserted at the right place: between "the watchdog detected something" and "a human has to figure out what it means."

The terminus-inference-gateway is already running in the cluster. It exposes an OpenAI-compatible endpoint with transparent provider fallback (local Qwen → Anthropic → OpenAI). The watchdog does not need a new AI infrastructure component — it needs wiring.

---

## What This Feature Builds

**Watchdog LLM Engine** adds a reasoning layer to terminus-watchdog that classifies novel alerts using the inference gateway and implements a feedback loop that graduates confirmed patterns into hardcoded detectors over time.

### Core Capabilities

**E1 — LLM Analysis Foundation**  
When the watchdog observes an alert that does not match a hardcoded detector — or when a detector fires but an operator wants richer context — the watchdog calls the inference gateway with a structured context bundle (alert text, Loki log snippets, ArgoCD/Temporal resource state). The gateway returns a JSON classification: what category of problem it is, a causal hypothesis, a recommended action, and an action tier (silent / notify / escalate). The watchdog routes the alert accordingly — suppressing noise, enriching Discord notifications, or offering remediation buttons.

**E2 — Pattern Learning Feedback Loop**  
Every LLM-classified observation is written to a SQLite observation log. Operators confirm or reject classifications via Discord. When a classification meets the graduation threshold (≥ 5 confirmed occurrences, ≥ 0.90 average confidence, 0 false positives over 30 days), it is flagged as a candidate for promotion to a hardcoded `BaseDetector`. This is the mechanism by which the LLM makes itself less necessary over time — novel patterns become deterministic code.

**E3 — ArgoCD Notifications Integration** *(optional post-MVP enhancement)*  
Adds an inbound webhook receiver so ArgoCD can push state change events directly to the watchdog, reducing detection latency from 30 seconds (polling) to near-instant. This is infrastructure work, not a core value story — defer until E1 and E2 are validated in production.

**E4 — Interactive Remediation Wiring**  
Adds Discord message component buttons (Approve / Reject) that send Temporal workflow signals back to the watchdog's active triage workflow. The operator sees a structured embed with the LLM's hypothesis and recommended action, clicks Approve, and the workflow executes the safe remediation step. Human confirmation is required only for risky operations; safe patterns (ArgoCD approve-to-sync for known-good drift) execute automatically.

**E5 — Agent Output Adapter** *(wiring story)*  
Connects the watchdog to the inference gateway via the existing `httpx` async client. Adds the `triage` routing profile to the gateway config (terminus-infra story), the ESO-managed `INFERENCE_GATEWAY_TOKEN` secret, and a `TriageClient` wrapper with Pydantic schema validation and graceful fallback to `needs-investigation` on parse error.

---

## What This Feature Is Not

- **Not a general assistant.** The LLM classifies ops alerts. It does not answer questions, manage schedules, or handle tasks outside the watchdog's detection scope. (Personal assistant use cases are a separate initiative.)
- **Not a new AI service.** The inference gateway already exists. This feature adds a caller, not infrastructure.
- **Not autonomous beyond defined safe patterns.** The three-tier autonomy model (silent fix / escalate-with-options / never-touch) is the hard boundary. Destructive operations (namespace deletion, PVC removal) always require human confirmation.
- **Not a replacement for hardcoded detectors.** LLM classification is the path into the observation log; the graduation loop is the path out. The end state is fewer LLM calls, not more.

---

## Why Now

MVP1 has been in production since early 2026. The alert routing is reliable. The gap is not stability — it is coverage. The inference gateway shipped in Q1 2026. The provider-routing layer is live. The cost and latency profile of local Qwen inference makes per-alert classification affordable (< $0.001 per triage call when local; < $0.003 on cloud fallback). The substrate is ready; the wiring is the work.

---

## Operator Impact

| Before | After |
|---|---|
| Novel error → silent miss | Novel error → structured triage within seconds |
| Human must form hypothesis from raw logs | LLM surfaces hypothesis and evidence chain in Discord embed |
| Repeated novel errors stay novel forever | Confirmed patterns graduate to deterministic detectors |
| Operator must manually approve every sync | Known-safe remediations execute automatically with audit trail |
| 11 patterns covered | Coverage grows continuously via pattern learning |

---

## Technical Constraints

| Constraint | Impact |
|---|---|
| Production-only deployment (no dev namespace) | Testing uses staging data; no parallel dev environment |
| In-process Temporal activity (no new service) | LLM call is a Temporal activity; replayable and auditable |
| Watchdog remains single-asyncio-process | LLM triage client runs as an async task, non-blocking |
| inference-gateway token required | 1 new ESO-managed secret; follows existing pattern |
| SQLite on PVC for observation log | No new database service; append-only, queryable |
| Discord persistent views for 24h approval windows | Views re-registered at bot startup from open observation log entries |

---

## Success Criteria

**Operator success:**  
- An alert for a previously-unknown error pattern surfaces a structured Discord embed with classification, hypothesis, and recommended action within 60 seconds of detection
- The operator can click Approve in Discord and have the watchdog execute the recommended action without switching to any other tool
- After 30 days of production use, at least one novel pattern has met graduation criteria and been promoted to a hardcoded detector

**Technical success:**  
- LLM classification adds < 5 seconds to the alert-to-Discord pipeline for triage-eligible alerts
- Pydantic validation catches 100% of malformed LLM outputs without crashing the detection loop
- Observation log query surfaces graduation candidates in < 100ms
- Zero cloud inference spend from triage calls when local Qwen is healthy

**Operational success:**  
- MVP1 detection patterns are unaffected — zero regression in existing alert routing
- Watchdog chart upgrade is a drop-in; no operator-visible behavior change for existing alerts

---

## Delivery Approach

**Sequence:** E5 (gateway wiring) → E1 (LLM classification) → E4 (Discord signal wiring) → E2 (observation log + pattern learning) → E3 (webhook, post-MVP).

E5 and E1 are the critical path. E4 depends on E1. E2 depends on E4 (human verdict requires Discord interaction). E3 is optional and independent.

**Full track:** This feature follows the full lifecycle track (PrePlan → BusinessPlan → TechPlan → FinalizePlan → Dev → Complete). The TechPlan will own architecture decisions for the triage client, observation log schema, and Temporal workflow structure.

**Story point estimate:** 97 points across 5 epics (E3 scope may reduce if deferred post-MVP). Target sprint allocation: E5+E1 in Sprint 1, E4 in Sprint 2, E2 in Sprint 3.
