---
stepsCompleted: [step-01-init, step-02-surfaces, step-03-conversation-dx, step-04-operator-dx, step-05-error-design, step-06-phase2-reference]
inputDocuments:
  - docs/fourdogs/kaylee/fourdogs-kaylee-kaylee-hermes-agent-integration/prd.md
  - docs/fourdogs/kaylee/fourdogs-kaylee-kaylee-hermes-agent-integration/product-brief.md
  - docs/fourdogs/kaylee/fourdogs-kaylee-kaylee-hermes-agent-integration/research.md
workflowType: ux-design
date: 2026-05-30
author: Todd
feature: fourdogs-kaylee-kaylee-hermes-agent-integration
surfaceType: integration-proxy
uiSurfaces: [chat-ui-conversation, operator-flag-control, error-degradation, discord-phase2]
---

# UX Design Specification — Kaylee Hermes Agent Integration

**Author:** Todd Hintzmann
**Date:** 2026-05-30
**Feature:** fourdogs-kaylee-kaylee-hermes-agent-integration
**Phase:** businessplan

---

## Surface Declaration

**No visual redesign in this feature.** Kaylee's web UI form and layout are preserved
unchanged. "Kaylee web UI redesign or UX changes" is explicitly out of scope (NG6 from PRD).

The experience surfaces this specification governs are:

| Surface | User | Experience Goal |
|---------|------|-----------------|
| Conversation experience | Betsy (and store staff via web UI) | Zero perceived degradation; noticeable improvement in reliability and coherence |
| Session memory contract | Betsy | Context persists across turns without Betsy needing to repeat herself |
| Streaming continuity | Betsy | Responses stream to the UI at the same latency or faster; no buffering regression |
| Operator feature flag DX | Todd (operator) | Clean flag flip, observable validation, low-friction rollback |
| Error and degradation UX | Betsy | Hermes unavailability shows a clear, human message — not a raw stack trace or silent failure |
| Discord interaction design | Betsy + store staff (Phase 2) | Defined here as reference contract to enable Phase 2 without rework |

---

## Surface 1: Conversation Experience (Betsy — Primary)

### Design Principle

> The best experience improvement is invisible. Queries that previously failed 30% of the time now just work. Betsy should never notice the architectural change — only the absence of failures.

### What Changes From Betsy's Perspective

| Experience Aspect | Before | After Phase 1 |
|-------------------|--------|---------------|
| Order query reliability | ~70% success — frequent "I couldn't process that" failures | ≥99% success — tool-grounded answer returned reliably |
| Sales query reliability | Same grammar-slot failure pattern | Same — ≥99% success |
| Within-session context | Each turn is independent; Kaylee may not recall what was asked 2 turns ago | Context is maintained across the full conversation session |
| Cross-session memory | Every session starts cold | Honcho deriver builds a persistent peer representation; Kaylee becomes progressively more context-aware across conversations over time |
| Response latency | Baseline (direct llamacpp) | Equal to or better than baseline; OmniRoute routing eliminates grammar-slot retry latency |

### Session Memory Contract

**Within a session:** Hermes maintains full conversation history. Betsy can ask follow-up
questions without re-stating context. Example:

> **Betsy:** What was the total sales value yesterday?
> **Kaylee:** [sales figure]
> **Betsy:** How does that compare to last Wednesday?
> **Kaylee:** [comparison] — understands "that" refers to yesterday's total without Betsy re-stating it

**Across sessions:** Honcho's deriver accumulates a peer representation of the `kaylee`
session peer. Over time, Kaylee's responses may reflect accumulated domain knowledge
(fourdogs product categories, recurring customer patterns, Betsy's usual query patterns).

**Memory isolation contract:** Honcho memory for `fourdogs-kaylee` is completely isolated
from Todd's personal Hermes install memory. Business conversations do not cross into
personal context and vice versa.

### Streaming Contract

The Kaylee thin proxy must maintain full streaming fidelity:

- Responses stream to the web UI token-by-token as Hermes produces them
- No buffering: the proxy forwards the `text/event-stream` response from Hermes to the
  browser without accumulation
- Time-to-first-token (TTFT) must not regress from the pre-integration baseline
- Long tool-call responses (e.g., listing 20 orders) stream in the same chunked pattern
  as before

**Streaming failure handling:** If the Hermes connection drops mid-stream, the UI must
surface an error message at the current cursor position — it must not silently hang or
show a partial truncated response as if it were complete.

### First Message After Flag Flip

When `USE_HERMES_AGENT` is first set to `true`, existing Kaylee sessions have no prior
Honcho memory. This is an expected, acceptable cold-start condition — Honcho builds memory
progressively. No special UX message is required. Betsy will not perceive this differently
from starting a new conversation.

**Acceptable behaviour:** First session after flag flip responds normally with zero memory
enrichment. Enrichment accumulates over subsequent sessions.

---

## Surface 2: Operator Feature Flag DX (Todd — Deployer)

### Design Principle

> The flag flip must be observable and reversible. The operator needs to know the system
> is routing to Hermes before declaring success, and must be able to undo in under
> one deployment cycle if a regression surfaces.

### Flag Lifecycle States

```
USE_HERMES_AGENT=false (default until smoke test passes)
         │
         ▼  [Smoke test pass]
USE_HERMES_AGENT=true (validation phase — 5 business days)
         │
         ▼  [Stability confirmed]
USE_HERMES_AGENT=true (production default — legacy handler can be pruned in future sprint)
         │
         OR
         ▼  [Regression detected]
USE_HERMES_AGENT=false (rollback — redeployment, no data loss)
```

### Flag Flip Procedure (Operator Runbook Requirement)

The architecture document (`architecture.md`, required in TechPlan) must include a
flag flip runbook with:

1. Pre-flip validation: Hermes health check (`GET /v1/health` at `hermes-fourdogs.trantor.internal:8642`) returns `{"status": "ok"}`
2. Pre-flip validation: Honcho health check (`GET /health` at port 8000) returns healthy
3. Pre-flip validation: MCP server registered — `hermes tools list` shows 14 tools
4. Smoke test passage (FR7.1–FR7.3 in PRD) before flag is set
5. Flag set: `USE_HERMES_AGENT=true` in Kaylee deployment env (ArgoCD or equivalent)
6. Post-flip: First real query logged with `hermes_session_id` in Kaylee logs — confirms traffic is flowing to Hermes
7. Rollback path: Set `USE_HERMES_AGENT=false` → redeploy → verify legacy handler responds

### Observability for Operator

The operator must be able to confirm routing state without reading Kaylee source code:

- Kaylee startup log must include `hermes_agent_enabled: true/false` (via FD-2 structured log)
- Each Hermes-routed request must log `hermes_session_id`, `hermes_response_status`, and latency
- Loki query to confirm Hermes routing is active: `{service="kaylee"} | json | hermes_agent_enabled="true"`

### Rollback Guarantee

Setting `USE_HERMES_AGENT=false` must unconditionally restore the legacy
`legacy_chat_handler` path. No Hermes, Honcho, or MCP call is made when the flag is
`false`. The `hermes_session_id` column in `chat_sessions` is nullable — legacy path
operates identically to pre-integration behaviour.

---

## Surface 3: Error and Degradation UX

### Design Principle

> Never show Betsy a stack trace, a raw HTTP error code, or a silent hang. Every
> degraded state must produce a human-readable response that lets her know what to do.

### Error State Map

| Error Condition | System Behaviour | User-Facing Message |
|-----------------|------------------|---------------------|
| Hermes service unreachable (connection refused on port 8642) | Kaylee Hermes adapter returns 503 with structured error | "I'm having trouble connecting right now. Please try again in a moment." |
| Hermes returns non-200 (including HTTP 400 from Hermes itself) | Adapter logs `hermes_error_status` + body; returns 502 to UI | "I wasn't able to get an answer. Please try rephrasing or try again shortly." |
| Hermes stream drops mid-response | Proxy detects incomplete stream; emits error event | "My response was interrupted. Here's what I had so far: [partial text]. Please try again." |
| MCP tool timeout (tool call exceeds timeout budget) | Hermes surfaces tool error; response includes best-effort answer | Hermes handles gracefully; user sees "I wasn't able to retrieve live data for that — [best-effort response]" |
| MCP tool returns empty result set | Expected condition — tool found no matching records | "I didn't find any matching records for that query." |
| Honcho deriver unavailable | Hermes continues normally — Honcho deriver is async and non-blocking for inline responses | No user-visible degradation; deriver outage means memory accumulation pauses, not response failure |
| OmniRoute inference gateway unavailable | Hermes returns error; Kaylee adapter surfaces degraded response | "I'm not able to think right now — OmniRoute is unavailable. Please try again in a few minutes." |

### Error Message UX Rules

1. **Never expose internal hostnames, port numbers, or stack traces** in user-facing error messages
2. **Always suggest a next action** — "try again", "rephrase", "contact the store"
3. **Never silently fail** — a request that receives no response is worse than a clear error message
4. **Distinguish transient from permanent** — connection errors suggest "try again"; tool schema errors suggest "rephrase"

### Timeout UX

The Kaylee thin proxy must enforce a total response timeout. If Hermes does not begin
streaming within `HERMES_TIMEOUT_SECONDS` (to be specified in architecture.md), the
proxy must close the connection and return a timeout error message to the UI.

**Timeout error message:** "Kaylee is taking longer than expected to respond. Please try again."

---

## Surface 4: Discord Interaction Design (Phase 2 Reference)

> This section is scoped out of Phase 1. It is documented here to establish the
> interaction contract so Phase 2 can build on this spec without requiring a new
> UX design sprint.

### Discord Bot Invocation Patterns

**Mention-based trigger (primary):**
```
@Kaylee what orders came in today?
@Kaylee check inventory for [product name]
```

**Command prefix (secondary — for structured commands):**
```
/kaylee orders today
/kaylee sales last-week
```

**Direct message:**
Users can DM the Kaylee bot for private queries that should not appear in the store channel.

### Discord Response Design

- **Streaming over Discord:** Discord does not support streaming. Kaylee Discord connector must:
  - Show a typing indicator while Hermes processes
  - Post the complete response as a single message when available
  - For long responses (>1800 chars), split into multiple messages with clear continuation markers: `[1/2]`, `[2/2]`

- **Tool-grounded responses must include a data source indicator:**
  ```
  📦 Orders as of [timestamp]: 3 orders received today...
  ```

- **Error states in Discord:**
  - Transient: "I'm having trouble right now — try again in a moment. 🔄"
  - Tool not available: "I couldn't retrieve live data for that. Try asking again or check the web UI."

### Discord Channel Scoping

- Kaylee Discord bot should only respond in designated channels or to DMs
- No response in general/random channels unless explicitly mentioned
- Channel permissions defined in bot configuration — not in Kaylee code

### Discord Session Model

Each Discord user/channel combination maps to a unique Hermes session context. Session
continuity within a Discord thread carries across the thread's lifetime via
`X-Hermes-Session-Id` constructed from `guild_id:channel_id:user_id` or thread ID.

---

## UX Implementation Notes for TechPlan

The following UX requirements must be translated into technical decisions during the
TechPlan phase:

| UX Requirement | Technical Decision Needed |
|----------------|--------------------------|
| Streaming fidelity via thin proxy | Hermes SSE forward handling in Kaylee's FastAPI proxy |
| Mid-stream error injection | Proxy must be able to inject an error event into an already-open SSE stream |
| Timeout enforcement | `HERMES_TIMEOUT_SECONDS` env var; `asyncio.wait_for()` around Hermes call |
| Startup log `hermes_agent_enabled` | Added to Kaylee app startup structured log (FD-2) |
| Per-request `hermes_session_id` in logs | Kaylee Hermes adapter log contract (FD-2) |
| Discord connector typing indicator | `async with channel.typing():` before Hermes call |
| Discord long-message splitting | Utility function in Discord plugin; not Kaylee backend concern |
