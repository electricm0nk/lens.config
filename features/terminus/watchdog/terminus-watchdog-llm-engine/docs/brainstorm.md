---
feature: terminus-watchdog-llm-engine
doc_type: brainstorm
status: draft
updated_at: 2026-05-13T12:00:00Z
---

# Brainstorm — Watchdog LLM Engine (terminus-watchdog MVP2)

## Context

terminus-watchdog MVP1 is a production Python asyncio service running 11 hardcoded detection
patterns against ArgoCD REST, Temporal gRPC, Kubernetes Watch, and Loki HTTP. It routes Discord
alerts for known patterns reliably but has no mechanism for unknown or novel errors — anything
outside the 11 patterns is either silently missed or produces noise.

This brainstorm captures the discovery conversation that led to the MVP2 scope definition, frames
the problem space, surfaces constraints, and defines open questions for research and planning.

---

## Discovery Conversation Summary (May 10, 2026)

The operator runs a k3s homelab with ArgoCD, Temporal, Proxmox, and supporting infrastructure
managed by a small portfolio of services. The daily pain is not a single system failing — it is
that **failures are invisible until the operator notices manually**. ArgoCD apps sit OutOfSync.
Temporal workflows stall. k3s nodes degrade. None of these have an intelligent observer.

Evaluated alternatives (OpenClaw, Hermes Agent, ZeroClaw, NanoClaw) were explored and rejected
as the primary substrate:
- **OpenClaw** is messaging-first and optimized for a chatbot-with-shell-access pattern; fighting
  the framework to build an autonomous ops observer.
- **Hermes Agent** adds a new framework dependency where the existing stack (Temporal + asyncio) is
  already the right substrate.
- For autonomous IT ops: **existing stack wins** (Temporal + LiteLLM/inference gateway + asyncio
  workers).
- OpenClaw and Hermes remain candidates for the **personal assistant** use case (airfare, reservations,
  phone calls) — a separate initiative entirely out of scope here.

---

## Problem Framing

### The Gap LLM Fills

A rule-based detector can tell you something is wrong. It cannot tell you **why** or **what to do**.
The 11 MVP1 patterns are precise but brittle. Novel errors — new versions, config drift,
infrastructure surprises — generate log noise that is not classified.

The LLM step is **narrow and specific**: read a context dump of logs + resource state, return a
structured classification. The LLM is not running the workflow. It is a reasoning accelerator
surfacing hypotheses that a human would need minutes to form.

### Self-Healing vs. Escalation

The operator wants an agent that **acts and reports**, not one that asks first.

Three-tier autonomy model:
1. **Known safe patterns** → silent fix + audit log
2. **Unknown or risky** → Discord message with structured diagnosis + options; Temporal signal on
   operator reply
3. **Never touch** → destructive ops (delete namespace, nuke PVCs) always escalate

The trust boundary is explicit: anything touching money (cloud API keys) or irreversible actions
gets a human confirm. Everything else should self-heal.

### Temporal as the Safety Net

The LLM can hallucinate. The Temporal workflow catches it. LLM returns a classification → workflow
code makes the actual decision based on classification schema. Invalid classification → workflow
escalates. The durable, replayable nature of Temporal workflows makes LLM-in-the-loop safe.

---

## Key Ideas Explored

### Idea 1: In-Process LLM Activity (Preferred Direction)

Run LLM inference as a Temporal activity inside the existing terminus-watchdog Python process.
- Gather context (ArgoCD state, kubectl describe, Loki logs, Temporal history)
- Call inference gateway via standard HTTP (OpenAI-compatible endpoint)
- LLM returns structured JSON classification
- Workflow branches on classification

**Why preferred:** No new service boundary, reuses existing asyncio design, consistent with MVP1
architecture (single process, task group model). Temporal handles durability and replay.

### Idea 2: Separate Reasoning Service

Stand up a dedicated reasoning microservice that consumes raw event data and returns
classifications. Watchdog posts context, polls for result.

**Why not preferred for MVP2:** Adds operational overhead, new deployment, new API contract.
The in-process model is simpler and sufficient for the call frequency (a few triage events/day).
Could be revisited if load or multi-service reuse justifies separation.

### Idea 3: OpenClaw Integration

Use OpenClaw as the shell-access chatbot layer, bolted onto the existing k8s tooling.

**Explicitly rejected:** Complex runtime dependency, external messaging requirement, security
concerns (CVSS 8.8 RCE vulnerability in Feb 2026 release). The operator's stack is better served
by native Temporal activities.

---

## Scope Signals from Discovery

The MVP2 roadmap (pre-existing in `docs/terminus/watchdog/roadmap.md`) defined 5 epics:

| Epic | Name | Story Points | Signal from Discovery |
|------|------|-------------|----------------------|
| E1 | LLM Analysis Foundation | 25 | Confirmed: core value delivery. Start here. |
| E2 | Pattern Learning Feedback Loop | 16 | Confirmed: graduate novel findings to hardcoded patterns |
| E3 | ArgoCD Notifications Webhook | 18 | Confirmed: real-time event ingestion replaces polling |
| E4 | Interactive Remediation Wiring | 22 | Confirmed: Discord signal/reply for human-in-the-loop |
| E5 | Agent Output Adapter | 16 | **Revised:** reuse existing terminus-inference-gateway; E5 becomes a wiring story |

**Total:** 97 story points (may reduce with E5 scope revision)

---

## Key Constraints Surfaced

| Constraint | Source | Impact |
|------------|--------|--------|
| **Inference failover required** | Operator requirement | LLM activity must tolerate Qwen adapter instability; failover to Anthropic or OpenAI must be transparent to the workflow |
| **Use existing inference gateway** | Operator direction | Do NOT use LiteLLM directly; route through terminus-inference-gateway which already provides provider abstraction and adapter pluggability |
| **Production-only deployment** | Watchdog constitution | No dev namespace; all changes delivered PR → merge → tag → release |
| **Extend MVP1 Discord bot** | Operator direction | Do not create a second Discord integration; extend the existing MVP1 bot with structured message formats and reply-signal handling |
| **No destructive autonomy** | Trust model | Never auto-execute namespace deletion, PVC removal, or any irreversible resource operation |
| **Single process model preserved** | MVP1 architecture | LLM reasoning runs as an asyncio-native activity inside the existing event loop architecture; not a sidecar or separate process |
| **Salvor boundary deferred** | Operator direction | MVP2 is standalone within terminus-watchdog; Salvor integration (if any) is a future feature |

---

## Open Questions for Research Phase

1. **Inference gateway routing for watchdog:** Does terminus-inference-gateway support a
   configured fallback chain (local Qwen → Anthropic → OpenAI)? What does the watchdog client
   need to configure to get failover behavior transparently? Read architecture.md and
   provider-routing docs.

2. **LLM prompt design for triage:** What is the optimal prompt structure for ArgoCD/Temporal/k8s
   triage? What output schema constrains the LLM classification reliably enough for autonomous
   action? Research existing agent triage prompt patterns.

3. **Observation log schema:** What fields must the observation log capture to support the
   pattern-learning feedback loop (E2)? Does this need a persistent store (SQLite, PostgreSQL)
   or is file-based append sufficient for MVP2 scale?

4. **Discord reply-as-signal implementation:** How does the existing MVP1 Discord bot receive
   and route reply events? What changes are needed to wire a Discord message reply into a
   Temporal signal that resumes a waiting workflow?

5. **Pattern graduation criteria:** What threshold (error count, operator confirmation, time window)
   should trigger promotion of an LLM-classified pattern to a hardcoded detector rule in E2?

6. **ArgoCD webhook vs. polling for E3:** ArgoCD supports webhook notification. Is the added
   complexity of webhook infrastructure justified over enhanced polling for the homelab scale?
   What are the latency trade-offs?

---

## Priority Signals

- **E1 is the non-negotiable first deliverable.** Without it, no other epic has a foundation.
- **E4 (Discord signal/reply) is the human-in-the-loop safety valve** and enables operator trust
  in autonomous action; plan it before E1 stories are finalized to ensure the escalation path
  is designed from day one.
- **E5 scope reduction** (reuse gateway vs. build adapter) should be validated against the
  terminus-inference-gateway architecture before sprint planning to right-size E5 effort.
- **E3 (webhook)** can be deferred post-E1/E2 if polling covers the latency requirement for
  MVP2 acceptance.

---

## Explicit Out of Scope

- Personal assistant features (airfare watching, reservations, phone calls) — separate initiative
- OpenClaw, Hermes Agent, or any third-party agent framework as the primary runtime
- Salvor IT-team orchestration — future initiative; watchdog feeds in but does not depend on it
- GPU-accelerated inference optimization — out of scope for watchdog; inference gateway handles this
- Production Temporal ReleaseWorkflow — watchdog uses direct ArgoCD deploy, not Temporal release
