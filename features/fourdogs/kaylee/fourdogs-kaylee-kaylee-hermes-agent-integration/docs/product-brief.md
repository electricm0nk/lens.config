---
title: "Product Brief: Kaylee Hermes Agent Integration"
feature: fourdogs-kaylee-kaylee-hermes-agent-integration
doc_type: product-brief
status: draft
created: "2026-05-30"
updated: "2026-05-30"
inputs:
  - docs/fourdogs/kaylee/fourdogs-kaylee-kaylee-hermes-agent-integration/brainstorm.md
  - docs/fourdogs/kaylee/fourdogs-kaylee-kaylee-hermes-agent-integration/research.md
---

# Product Brief: Kaylee Hermes Agent Integration

## Executive Summary

Kaylee, the fourdogs AI assistant, is broken. Her inference backend exhausts grammar slots
under normal load — producing roughly a 30% failure rate when all 14 business tools are
present. She has no memory of previous conversations, cannot learn about the business over
time, and is locked inside a single web UI with no path to reach Betsy and staff where they
actually work. Every new capability requires Kaylee code changes and has reliably introduced
new failure modes.

The solution is architectural: replace Kaylee's inference and tool layer with
**Hermes Agent + Honcho**, a production-grade open-source agent platform (Nous Research +
Plastic Labs, both MIT-licensed). Kaylee's FastAPI backend becomes a thin proxy — receive a
message, POST to Hermes, return the response. All intelligence, tool execution, memory, and
channel routing live in Hermes. Kaylee's 14 existing business tools wrap into a lightweight
MCP server attached to the Hermes profile. Honcho accumulates domain knowledge across every
conversation, making every agent on the install smarter over time.

The result: a reliable, learning, multi-channel business assistant — deployed on a dedicated
infrastructure node with complete business isolation, ready to hand off if fourdogs is
ever sold or spun off.

## The Problem

Kaylee's current architecture has three compounding failure modes:

**Reliability failure.** The terminus-inference-gateway (llamacpp, Qwen3-35B MoE) has a
confirmed grammar-slot exhaustion bug when 12 or more tool schemas are sent in a single
request. This causes roughly 30% of Kaylee's inference calls to return HTTP 400, with no
retry logic in place. Business-critical functions — checking orders, looking up customers —
fail at 1-in-3 attempts. This is not a configuration issue; it is a structural incompatibility
between the current tool grammar implementation and the Qwen3 thinking model's slot budget.

**Intelligence ceiling.** Kaylee starts every conversation cold. After months in production,
she knows nothing persistent about fourdogs products, Betsy's preferences, recurring orders,
or seasonal patterns. The inline `tools/dispatcher.py` dispatch layer means adding or
changing a tool requires Python changes, test updates, and a full deployment cycle. There is
no mechanism for the system to get better on its own.

**Channel lock-in.** Kaylee exists only in the Kaylee web UI. Betsy and store staff use
Discord for their working communication. There is no abstraction layer for channels — adding
Discord would require reimplementing the entire session, dispatch, and response loop from
scratch. The architecture does not support multi-channel growth.

## The Solution

Deploy a **dedicated Hermes Agent + Honcho stack** at `hermes-fourdogs.trantor.internal`
and reduce Kaylee's backend to a thin proxy with a feature flag:

```
Web UI → Kaylee backend → POST /v1/chat/completions:8642 → Hermes Agent
                                                                ↓
Discord (store server)  ───────────────────────────→  kaylee profile
                                                                ↓
                                               MCP server (14 business tools)
                                                                ↓
                                            Honcho dialectic + memory enrichment
                                                                ↓
                                          OmniRoute → stable LLM inference
```

**Hermes Agent** (Nous Research, open source) is the agent gateway layer. It owns
conversation management, tool dispatch via the MCP protocol, agent persona (SOUL.md, hot-
reloaded), multi-channel connectors (Discord plugin, OpenAI-compatible API, and more), and
Honcho memory integration. The fourdogs `kaylee` profile is one named agent on the dedicated
install — additional profiles can be added without touching Kaylee's code.

**Honcho** (Plastic Labs, open source) is the memory platform. It builds peer representations
across every conversation turn via an async deriver worker. The dialectic API synthesizes
accumulated knowledge into every response. Over time, Honcho makes every agent on the
fourdogs install smarter about the domain — shared knowledge, independent profiles.

**Fourdogs MCP Server** is a lightweight FastAPI service that wraps Kaylee's 14 existing tool
functions and exposes them via the Model Context Protocol. Hermes discovers and registers
these at startup. No tool dispatch code remains in Kaylee's backend.

**OmniRoute** routes all LLM calls through the terminus-inference-gateway with automatic
failover. The grammar-slot exhaustion bug is a non-issue at this layer — OmniRoute presents
a stable, provider-agnostic inference surface to both Hermes and Honcho.

## What Makes This Different

**The grammar-slot bug disappears by architecture.** Hermes manages tool dispatch natively
through MCP; it does not send raw JSON tool grammars to llamacpp. OmniRoute provides
failover when the local provider is unhealthy. The ~30% failure rate becomes a non-event.

**Business separation is a hard design constraint.** The fourdogs Hermes install is
completely isolated from the personal `hermes.trantor.internal` instance: own VM, own
databases, own Honcho memory, own credentials. If fourdogs is sold or spun off, the agent
infrastructure transfers as a single VM — no entanglement with personal tools, personal
conversation history, or personal infrastructure.

**Infrastructure-minimal and open source.** Hermes and Honcho are configured, not forked.
The only net-new code is a thin MCP server shim. No proprietary agent platforms, no SaaS
dependencies for core intelligence.

## Who This Serves

**Betsy** is the primary user — the fourdogs business owner who queries orders, checks sales
performance, looks up customers, and needs accurate answers while managing the store. She
currently uses the Kaylee web UI but operates primarily in Discord. Success means she can
ask Kaylee a question in Discord without switching tabs.

**Store staff** are secondary users — Discord access when the store's Discord server has the
Kaylee bot configured. Primary use cases: checking order status for a customer on the phone,
looking up current inventory for a specific product.

## Success Criteria

- HTTP 400 failure rate on tool-call requests drops from ~30% to <1%
- Kaylee correctly answers order and sales queries on first attempt in ≥95% of interactions
- Session context carries across turns within a conversation (Honcho session continuity)
- `USE_HERMES_AGENT=true` runs for 5 consecutive business days without regression after
  Phase 1 deployment
- Discord integration routes a test message through the `kaylee` profile and returns a
  tool-grounded response
- No plaintext secrets in any Hermes or Honcho config file — all credentials from Vault

## Scope

**Phase 1 — In scope:**

- Dedicated `hermes-fourdogs.trantor.internal` VM via OpenTofu + Ansible (`terminus.infra`)
- Honcho deployment with isolated `honcho_fourdogs` database, OmniRoute LLM config,
  pgvector extension, and schema migration at first boot
- Hermes Agent `kaylee` profile with OmniRoute provider, fourdogs `SOUL.md` persona, and
  MCP server registration
- Fourdogs MCP server: 14 existing tools ported; Orders + Sales tools prioritized for Phase 1
- Kaylee backend: thin proxy with `USE_HERMES_AGENT` feature flag, `hermes_session_id`
  column on `chat_sessions`, hermes adapter module
- Architecture document (`architecture.md`) covering Hermes+Honcho+Kaylee+OmniRoute
  integration — required before story authoring
- End-to-end smoke test confirming feature flag path before default is flipped to `true`

**Out of scope (Phase 1):**

- Discord store server bot integration (Phase 2)
- Additional agent profiles: `inventory-watch`, `email-processor`, `ops-summary` (Phase 2–3)
- EtailPet API tools in MCP server (Phase 2)
- Email read/send via emailfetcher API (Phase 3)
- Honcho dialectic reasoning level tuning and peer model optimization
- Kaylee UI redesign

## Vision

In 6–12 months, `hermes-fourdogs.trantor.internal` is the intelligence platform for the
fourdogs business. Betsy asks questions in Discord and gets tool-grounded, memory-enriched
answers without opening a browser. The agent fleet grows to four profiles: `kaylee` handles
customer and order inquiries, `inventory-watch` proactively surfaces low-stock and velocity
alerts, `email-processor` triages and drafts customer email responses, and `ops-summary`
delivers Betsy's daily business snapshot. Honcho's accumulated domain knowledge makes every
agent smarter with every conversation — the system learns the fourdogs business
automatically.

The architecture also creates the clean exit ramp: if fourdogs is sold, the
`hermes-fourdogs.trantor.internal` VM is the scope of the transfer. One VM, one database
set, one credential scope.

## Open Items and Risks

| Item | Impact | Resolution path |
|------|--------|-----------------|
| Honcho version pin: personal install is v3.0.7; current main has breaking config changes | Deployment config correctness | Pin to tested tag; verify in BusinessPlan technical research |
| Embeddings provider: Honcho pgvector requires `/v1/embeddings`; OmniRoute support unconfirmed | Schema locked at first migration if wrong | Verify OmniRoute embeddings endpoint before Honcho config is finalized |
| Hermes Agent state DB: unclear if Hermes requires its own Postgres beyond Honcho | Provisioning scope | Inspect personal install service config and Hermes source before infrastructure sprint |
| OmniRoute accessibility from hermes-fourdogs VM (host network, not k3s) | LLM connectivity | Confirm NodePort / metallb IP reachable at `inference-gateway.trantor.internal` |
| Vault secret delivery to non-k8s VM: ESO pattern does not apply | Credential provisioning | Research Vault Agent sidecar or Vault API pull pattern for VM services |
| Kaylee Postgres session ID → Honcho session model mapping | Session continuity fidelity | Validate lazy-provisioning approach in spike before story authoring |

## Constitution Compliance

- **Article 3 (architecture.md required):** `architecture.md` is scoped into Phase 1 as a
  blocking prerequisite before any implementation stories are authored.
- **Article 4 (no confidential data exfiltration):** Hermes and Honcho run on
  `trantor.internal`. OmniRoute is the governed LLM gateway; tool calls and business data
  do not leave the internal network. Cloud provider inference is permitted only through
  OmniRoute's governed routing policy.
- **Article 9 (security first):** EtailPet API key, DB credentials, `DISCORD_BOT_TOKEN`,
  Hermes session tokens, and OmniRoute API keys must all be provisioned from Vault. No
  plaintext secrets in any config file, `.env`, or environment variable in source control.
- **FD-1 (develop-branch doc sync):** Merges to develop that change Kaylee routing,
  tool integration, or agent architecture must update `docs/fourdogs/app-arch.md`.
