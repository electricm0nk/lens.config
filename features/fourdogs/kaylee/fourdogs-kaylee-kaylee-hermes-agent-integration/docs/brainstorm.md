---
feature: fourdogs-kaylee-kaylee-hermes-agent-integration
doc_type: brainstorm
status: complete
updated_at: "2026-05-30"
---

# Brainstorm — Kaylee Hermes Agent Integration

## Problem Statement

Kaylee (the fourdogs AI assistant) is unreliable and has no useful capabilities. Her current
architecture implements bespoke tool dispatch and message assembly inline in the chat route,
calling `terminus-inference-gateway` directly with hard-coded tooling. Every new capability
requires code changes and has been a source of recurring errors.

The goal: replace Kaylee's inference and tool layer entirely with a Hermes agent profile.
Kaylee becomes a thin adapter; all intelligence, memory, tool execution, and learning
capabilities live in the Hermes agent.

## Current State

- FastAPI Python backend on k3s (`fourdogs-kaylee` namespace)
- Chat sessions in Postgres (`chat_sessions`, `chat_messages`)
- Calls `terminus-inference-gateway` with assembled messages + inline tool definitions
- `tools/dispatcher.py` does ad hoc tool dispatch (fragile, hard to extend)
- Single `index.html` UI
- Single user (Betsy), no auth on chat API
- Already goes through `GATEWAY_URL` / `GATEWAY_VAULT_TOKEN` / `GATEWAY_MODEL` env vars

## Target State Vision

Kaylee can:
- Answer questions about orders and sales accurately
- Interact with the ordering system directly
- Interact with the inventory system directly
- Call EtailPet APIs directly when needed
- Read and send emails
- Learn and retain domain context over time (via Hermes memory)

## Integration Architecture Options

### Option A: Thin Proxy (Full Delegation) — Recommended for MVP
Kaylee backend becomes a pure proxy: receive message → POST to Hermes agent API → return response.
Session management moves entirely to Hermes. All tool calling lives in Hermes agent profile.
Kaylee's Postgres is retained only for UI-level session routing (session IDs the UI already uses).

**Pros:** Maximum simplicity, Hermes owns all the hard stuff, easy to upgrade
**Cons:** Hermes becomes a dependency; no local fallback

### Option B: Smart Adapter (Hybrid)
Kaylee keeps its session management in Postgres. Calls Hermes per-message with session context.
Kaylee owns: auth boundary, rate limiting, session persistence.
Hermes owns: LLM inference, memory, tool execution.

**Pros:** Clear separation; Kaylee can evolve independently
**Cons:** Two places where session state could diverge; more surface to maintain

### Option C: Hermes Native (Direct UI)
Remove Kaylee's backend; UI talks directly to Hermes agent API. Kaylee backend becomes a thin
auth/routing shim only.

**Pros:** Eliminates backend as failure point
**Cons:** Solid wall harder to maintain; Hermes API surface exposed to UI — not recommended given isolation requirement

## Isolation Wall Options

The "solid wall" between Kaylee and everything else:

### Wall Option 1: Agent Profile Isolation — Recommended for initial phase
Dedicated Hermes agent profile `kaylee-fourdogs` with its own:
- System prompt scoped to fourdogs domain only
- Tool whitelist (only fourdogs tools)
- Separate memory/context store
- Dedicated API credential from Kaylee backend → Hermes

### Wall Option 2: Network + Auth Isolation
Kaylee backend holds a dedicated API key/token for Hermes. Hermes profile restricts tool scope
by credential. Enforced at protocol level.

### Wall Option 3: Separate Hermes Install — **SELECTED**
Dedicated Hermes + Honcho stack deployed for fourdogs exclusively. True infrastructure air gap.

**Business separation rationale (primary driver):** If the fourdogs business is ever sold or
spun off, it must be disentangable from Todd's personal infrastructure and personal Hermes
install on `hermes.trantor.internal`. A shared Hermes means the buyer inherits Todd's personal
memory store, agent history, and infrastructure dependencies.

**Deployment target:** New dedicated Proxmox VM on `trantor.internal` — same physical
infrastructure is fine. VMs are easy to migrate or hand off. The separation is logical:
own hostname, own memory DB, own credentials, own Hermes config. Provisioning follows the
existing `hermes.trantor.internal` pattern (OpenTofu + Ansible in `terminus.infra`).

**Additional benefits:** Complete credential separation, no risk of cross-contamination between
personal agent context and business agent context, independently upgradeable, independently
backupable. If fourdogs ever moves off trantor, the VM migrates as a unit.

## Tool Strategy

| Tool | Data Source | Implementation |
|---|---|---|
| Order queries | `fourdogs_central` DB | Native Hermes tool (Python function) |
| Sales queries | `fourdogs_central` DB | Native Hermes tool |
| Customer lookup | `fourdogs_central` DB | Native Hermes tool |
| Inventory queries | `fourdogs_central` DB | Native Hermes tool |
| EtailPet API | EtailPet REST API | HTTP tool call from Hermes |
| Email read | Existing emailfetcher service | Hermes tool → emailfetcher API (avoid duplication) |
| Email send | SMTP via emailfetcher or direct | Hermes tool → emailfetcher API |

**Preferred approach:** Native Hermes tools for DB queries (already have the query patterns from
existing `tools/` directory). Use emailfetcher service API for email (don't duplicate the
existing email ingestion work).

## Migration Strategy Options

### Approach 1: Feature Flag (Strangler Fig) — Recommended
Add `USE_HERMES_AGENT=true` env var. When set, route to Hermes; otherwise fall back to old path.
Gradually shift, then remove old code. Lower risk, testable in production.

### Approach 2: Big Bang Replace
Create new Hermes agent profile, rewrite Kaylee backend, deploy in one sprint.
Risk: Long period with no working Kaylee.

### Approach 3: Parallel Deployment
Deploy second endpoint alongside old Kaylee. Once validated, swap. Clean rollback path.

## Expanded Capability Drivers (Full Hermes Stack)

The full Hermes Agent + Honcho stack unlocks capabilities that Honcho-only never could:

### Discord Integration
Hermes Agent has native channel connectors. The fourdogs Discord server can be wired
to the Kaylee agent profile directly — users in the store's Discord can query orders,
inventory, and sales without opening the Kaylee web UI. This is a material business
capability: staff and Betsy get an always-available assistant in the channel where
they already work.

### Multi-Agent Expansion
A single fourdogs Hermes install can host multiple named agents with distinct
personalities, system prompts, and tool scopes — all sharing the same Honcho memory
backend:

| Agent | Purpose |
|---|---|
| `kaylee` | Primary customer and orders assistant |
| `inventory-watch` | Proactive alerts on low stock, sales velocity anomalies |
| `email-processor` | Triage and respond to customer emails |
| `ops-summary` | Daily business snapshot for Betsy |

New agents can be added without touching Kaylee's code. Hermes handles routing,
isolation, and tool whitelisting per profile.

### Learning and Skill Accumulation
Honcho's deriver builds peer representations over time. A Honcho-only integration
would give Kaylee memory. A Hermes+Honcho integration gives the entire fourdogs
agent fleet shared, accumulating domain knowledge — every conversation makes every
agent smarter about fourdogs.

## Key Open Questions

1. **Hermes streaming support:** CONFIRMED — Honcho dialectic API supports `stream: true` (SSE `text/event-stream`)
2. **Hermes agent API contract:** CONFIRMED via live OpenAPI inspection — Honcho FastAPI on port 8000; `POST /v3/workspaces/{workspace_id}/peers/{peer_id}/chat` with `DialecticOptions` body (`query`, `session_id`, `stream`, `reasoning_level`)
3. **Separate install hosting:** Where does the fourdogs Hermes+Honcho stack live? Options: k3s in `fourdogs` namespace; dedicated Proxmox VM; or hybrid (Honcho in k3s, Hermes on a VM)
4. **Tool credential handling:** EtailPet API key, DB connection string — must use Vault or Hermes secrets mechanism; must not be in agent config plaintext
5. **Honcho database:** Personal Hermes uses `hermes_memory` DB. Fourdogs Hermes needs its own isolated DB (`fourdogs_hermes_memory` or similar) on the Patroni cluster
6. **Honcho LLM backend:** Personal Hermes points at llamacpp on worker-07 directly. Fourdogs Hermes should point at the inference-gateway (the fourdogs-governed endpoint)
7. **Session continuity:** Kaylee uses Postgres session IDs — Honcho has its own workspace/session model; Kaylee needs to map between them (store Honcho session ID in its Postgres)
8. **Peer/workspace setup:** Honcho requires a workspace + peer setup before chat. Research needed: how to provision a fourdogs `kaylee-betsy` peer and `fourdogs` workspace programmatically
9. **Emailfetcher integration:** Can Hermes call the existing emailfetcher service API rather than re-implementing email logic?

## Risks and Assumptions

- **Assumption:** Hermes is stable enough to be a dependency for Kaylee (given the `terminus.hermes` work is v0.1.1)
- **Assumption:** The existing `tools/` query patterns can be ported to Hermes tools without significant rework
- **Risk:** Hermes tool credential management may not yet have a Vault-native path — needs investigation in research
- **Risk:** Session ID mapping between Kaylee Postgres and Hermes session model may require a translation layer
- **Risk:** The emailfetcher API surface may not be stable or accessible from Hermes yet

## Decisions Made in Brainstorm

1. **Architecture choice:** Option A (Thin Proxy / Full Delegation) for the first iteration. Kaylee becomes a Hermes wrapper.
2. **Isolation:** Wall Option 3 — **Separate Hermes install for fourdogs**. Not the personal
   `hermes.trantor.internal` instance. Business separation is the primary driver: fourdogs
   must be independently sellable/disentangleable from personal infrastructure.
3. **Hermes stack components:** Two components required — **Hermes** (the agent gateway, ACP protocol on 9119) and **Honcho** (the memory/knowledge FastAPI platform on 8000). Both must be deployed separately for fourdogs.
4. **Integration API:** Kaylee calls the **Hermes Agent ACP endpoint** (port 9119). Hermes owns
   all conversation management, tool dispatch, and learning. Hermes internally uses Honcho for
   memory. Kaylee does NOT call Honcho directly — that is Hermes's concern.
5. **LLM backend:** **OmniRoute** (terminus-inference-gateway with OmniRoute provider routing)
   is the LLM backend for both Hermes and Honcho. Direct llamacpp / Qwen3 is NOT used for
   business-critical functions — the grammar-slot exhaustion bug makes it too unstable.
   OmniRoute provides failover to cloud providers and stable routing.
6. **Tool implementation:** Kaylee's 14 existing tools become **native Hermes skills**. No tool
   dispatch remains in Kaylee backend. HTTP delegation to emailfetcher for email functions.
7. **Migration:** Feature flag approach (Strangler Fig) to reduce risk.
8. **Priority order for tools:** Orders + Sales first; Inventory second; EtailPet API third; Email fourth.

## Constitution Compliance Notes

- Article 4 (No Confidential Data Exfiltration): All Hermes agent calls stay on the internal network (hermes.trantor.internal); no external data transmission.
- Article 9 (Security First): EtailPet API key, DB credentials, and Hermes token must all be managed via Vault ESO — not env vars with plaintext secrets.
- Article 16 (Internal Infrastructure Primacy): Hermes and Honcho run on internal infra.
  OmniRoute routes to cloud providers as needed — this is acceptable as OmniRoute is a
  governed gateway, not direct SaaS AI coupling. Tool calls and business data stay internal.
- **Business separation (operational):** Fourdogs Hermes install must have no shared state, no shared credentials, and no shared databases with the personal `hermes.trantor.internal` instance. This is a hard architectural constraint, not a soft preference.
- FD-2 (Loki Logging): Kaylee's updated backend must continue to emit structured JSON logs. Hermes adapter layer must also log calls and errors in FD-2 format.
- FD-1 (Doc Sync): Merges to develop that change routing or tool integration must update `docs/fourdogs/app-arch.md`.
