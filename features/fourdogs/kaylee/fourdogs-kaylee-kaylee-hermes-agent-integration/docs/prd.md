---
feature: fourdogs-kaylee-kaylee-hermes-agent-integration
doc_type: prd
status: draft
goal: "Replace Kaylee's unreliable inline inference layer with Hermes Agent + Honcho to eliminate tool-call failures, add persistent memory, and enable multi-channel access."
key_decisions:
  - "Architecture: Option A Thin Proxy — Kaylee becomes a pure proxy to Hermes"
  - "Isolation: Wall Option 3 — dedicated hermes-fourdogs VM, not the personal install"
  - "Tool integration: MCP server (not native Hermes skills)"
  - "LLM backend: OmniRoute (terminus-inference-gateway), not direct llamacpp"
  - "Migration: Strangler Fig feature flag (USE_HERMES_AGENT)"
  - "Scope: Phase 1 is web-UI-only; Discord is Phase 2"
open_questions:
  - "OmniRoute /v1/embeddings support and dimension count for Honcho pgvector schema"
  - "Vault Agent or Vault API pull pattern for non-k8s VM credential delivery"
  - "MCP server deployment placement: hermes-fourdogs VM vs k3s vs embedded"
  - "OmniRoute NodePort/metallb IP accessibility from host-network hermes-fourdogs VM"
  - "Honcho version pin: v3.0.7 (known-good from personal install) vs current main"
  - "Hermes Agent state DB requirement (beyond Honcho's honcho_fourdogs DB)"
depends_on:
  - terminus-inference-omniroute-gateway-migration
blocks: []
updated_at: "2026-05-30T00:00:00Z"
stepsCompleted: [step-01-init, step-02-discovery, step-02b-vision, step-02c-executive-summary,
  step-03-success, step-04-journeys, step-05-domain, step-06-innovation, step-07-project-type,
  step-08-scoping, step-09-functional, step-10-nonfunctional, step-11-polish, step-12-complete]
inputDocuments:
  - docs/fourdogs/kaylee/fourdogs-kaylee-kaylee-hermes-agent-integration/product-brief.md
  - docs/fourdogs/kaylee/fourdogs-kaylee-kaylee-hermes-agent-integration/research.md
  - docs/fourdogs/kaylee/fourdogs-kaylee-kaylee-hermes-agent-integration/brainstorm.md
workflowType: prd
---

# Product Requirements Document — Kaylee Hermes Agent Integration

**Author:** Todd Hintzmann
**Date:** 2026-05-30
**Feature:** fourdogs-kaylee-kaylee-hermes-agent-integration
**Phase:** businessplan

---

## 1. Executive Summary

Kaylee is the fourdogs AI assistant. Her current architecture exhausts grammar slots on
the llamacpp inference backend, causing ~30% of tool-call requests to fail with HTTP 400.
She has no persistent memory, cannot learn about the business over time, and exists only
in the web UI — no Discord, no multi-channel access.

This feature replaces Kaylee's inference and tool layer entirely with
**Hermes Agent + Honcho** — an open-source agent platform (Nous Research + Plastic Labs,
both MIT licensed). Kaylee's FastAPI backend becomes a thin proxy. Hermes owns all
conversation management, tool dispatch via MCP, persona, and channel routing. Honcho
provides persistent cross-conversation memory and dialectic reasoning enrichment.

The integration runs on a **dedicated Proxmox VM** (`hermes-fourdogs.trantor.internal`)
with complete logical isolation from Todd's personal Hermes install — own databases, own
credentials, independently relocatable if fourdogs is ever sold.

The result: a reliable, learning, multi-channel business assistant that eliminates the
grammar-slot bug by architecture and never starts a conversation cold again.

---

## 2. Problem Statement

### 2.1 Reliability Failure (Critical)

The terminus-inference-gateway (llamacpp, Qwen3-35B MoE) has a confirmed grammar-slot
exhaustion bug when 12 or more tool schemas are sent in a single request. This causes
approximately 30% of Kaylee's inference calls to return HTTP 400. Business-critical
functions — checking orders, looking up customers, querying sales — fail at 1-in-3
attempts. This is a structural incompatibility between the current tool grammar
implementation and the Qwen3 thinking model's slot budget. It cannot be fixed by
configuration tuning; it requires architectural change.

### 2.2 Intelligence Ceiling (High)

Kaylee starts every conversation cold. After months in production, she knows nothing
persistent about fourdogs products, Betsy's preferences, recurring orders, or seasonal
patterns. Adding or changing a tool requires Python changes, test updates, and a full
deployment cycle. There is no mechanism for the system to improve on its own.

### 2.3 Channel Lock-In (Medium)

Kaylee exists only in the web UI. Betsy and store staff use Discord as their primary
working channel. Adding Discord would require re-implementing the full session, dispatch,
and response loop in Kaylee's backend. The architecture does not support channel expansion.

---

## 3. Goals

### 3.1 Primary Goals (Phase 1)

| ID | Goal |
|----|------|
| G1 | Eliminate the grammar-slot HTTP 400 failure pattern by replacing direct llamacpp inference with Hermes + OmniRoute |
| G2 | Preserve full Kaylee business capability — all 14 existing tools functional via MCP server |
| G3 | Add persistent session memory via Honcho — context carries across turns and accumulates over time |
| G4 | Deploy Hermes+Honcho on a dedicated, business-isolated VM with complete credential separation |
| G5 | Migrate via feature flag (Strangler Fig) to ensure zero downtime and clean rollback path |
| G6 | All credentials sourced from Vault — no plaintext secrets in any config file or source control |

### 3.2 Non-Goals (Phase 1)

| NG | Out of Scope |
|----|--------------|
| NG1 | Discord integration for the fourdogs store server (Phase 2) |
| NG2 | Additional agent profiles: `inventory-watch`, `email-processor`, `ops-summary` |
| NG3 | EtailPet API tools in the MCP server (Phase 2) |
| NG4 | Email read/send via emailfetcher API (Phase 3) |
| NG5 | Honcho dialectic tuning and peer model optimization |
| NG6 | Kaylee web UI redesign or UX changes |
| NG7 | Direct llamacpp inference from Hermes (only OmniRoute is permitted for business functions) |
| NG8 | Sharing memory, databases, or state with the personal `hermes.trantor.internal` install |

---

## 4. Success Criteria

| ID | Criterion | Measurement |
|----|-----------|-------------|
| SC1 | Tool-call HTTP 400 failure rate ≤1% | Monitored over 5 consecutive business days with `USE_HERMES_AGENT=true` |
| SC2 | Kaylee correctly answers order and sales queries on first attempt | ≥95% of tested interactions |
| SC3 | Session context carries across turns | Honcho session continuity verified — follow-up messages reference prior context without prompt injection |
| SC4 | Phase 1 deployment stability | Feature flag `USE_HERMES_AGENT=true` runs 5 consecutive business days without regression |
| SC5 | No plaintext secrets in any Hermes/Honcho/MCP config | Vault audit confirms all secrets provisioned from Vault paths |
| SC6 | Smoke test passes before feature flag defaults to `true` | End-to-end test: web UI → Kaylee → Hermes → MCP tool → response |
| SC7 | MCP server exposes all 14 tools | `hermes tools list` output confirms 14 registered tools |

---

## 5. User Profiles and Journeys

### 5.1 Primary User: Betsy (Business Owner)

**Profile:** Sole operator of the fourdogs online pet supply store. Primary access via
the Kaylee web UI today; primary working communication via Discord.

**Core journeys:**

**Journey B1 — Order Status Query**
> Betsy opens Kaylee web UI → types "what orders came in this morning?" → Kaylee proxies to
> Hermes → Hermes calls `order_view` MCP tool → tool queries fourdogs_central DB → Hermes
> assembles response with Honcho memory context → Kaylee returns streaming response to UI

**Journey B2 — Sales Performance Check**
> Betsy asks "how did sales look last week compared to the same week last month?" → Hermes
> calls `sales_summary` and `sales_query` MCP tools → response includes trend context
> accumulated from prior queries via Honcho peer memory

**Journey B3 — Customer Lookup**
> Betsy asks about a specific customer → Hermes calls `customer_lookup` → returns customer
> record; Honcho may enrich with remembered preferences from past queries about that customer

### 5.2 Secondary Users: Store Staff

**Profile:** Fourdogs store employees who check order status or inventory for customers on
the phone or in-store. Primary channel: Discord (Phase 2). Currently not using Kaylee.

**Core journeys:**

**Journey S1 — Order Check for Phone Customer (Phase 1 via web UI, Phase 2 via Discord)**
> Staff opens Kaylee web UI → types order reference → Hermes routes to `order_view` MCP tool →
> staff reads order status, tracking info, and line items to customer

**Journey S2 — Inventory Check**
> Staff asks about stock level for a product → Hermes calls `inventory_lookup` →
> returns current stock and velocity context

### 5.3 Developer Journey: Feature Flag Control

> Operator sets `USE_HERMES_AGENT=false` in Kaylee deployment env → Kaylee routes to legacy
> `legacy_chat_handler` unchanged → no Hermes involvement → clean rollback path at any time

---

## 6. Functional Requirements

### 6.1 Infrastructure: Dedicated Hermes+Honcho VM

| ID | Requirement |
|----|-------------|
| FR1.1 | A new Proxmox VM `hermes-fourdogs.trantor.internal` must be provisioned via OpenTofu + Ansible in `terminus.infra` using the existing hermes install pattern |
| FR1.2 | The VM must be logically isolated from the personal `hermes.trantor.internal` install: own hostname, own credentials, own databases, no shared network or DNS scope with personal services |
| FR1.3 | VM provisioning must be documented in a Terraform/Ansible runbook as part of the architecture.md (required before story authoring, Article 3) |
| FR1.4 | The VM must be independently relocatable — configuration and data live on the VM; no external service dependency on personal infrastructure |

### 6.2 Honcho Deployment

| ID | Requirement |
|----|-------------|
| FR2.1 | Honcho must be deployed as two systemd services on the hermes-fourdogs VM: `honcho-fourdogs.service` (FastAPI, port 8000) and `honcho-fourdogs-deriver.service` (background deriver worker) |
| FR2.2 | Honcho must use an isolated `honcho_fourdogs` database on the Patroni cluster (10.0.0.56:5432) with the pgvector extension enabled |
| FR2.3 | `EMBEDDING_VECTOR_DIMENSIONS` must be set and documented before first Alembic migration — this value is schema-locked and cannot be changed after first boot |
| FR2.4 | Honcho LLM configuration must point to OmniRoute (terminus-inference-gateway) via OpenAI-compatible transport — not direct llamacpp |
| FR2.5 | Auth must be disabled (`AUTH_USE_AUTH=false`) for the internal-only deployment |
| FR2.6 | Honcho Workspace `fourdogs` and Peer `kaylee` must be bootstrapped via a one-time setup script before Kaylee deployment |
| FR2.7 | All Honcho credentials (DB password, OmniRoute API key) must be provisioned from Vault paths — not stored in `.env` or config files in source control |

### 6.3 Hermes Agent Deployment and Profile

| ID | Requirement |
|----|-------------|
| FR3.1 | Hermes Agent must be deployed as a systemd service `hermes-fourdogs.service` on the hermes-fourdogs VM, exposing the OpenAI-compatible API on port 8642 |
| FR3.2 | A `kaylee` agent profile must be created with: `SOUL.md` persona for the fourdogs assistant, OmniRoute provider config, Honcho endpoint, and fourdogs MCP server registration |
| FR3.3 | The `kaylee` profile `SOUL.md` must define Kaylee's persona and restrict the agent scope to fourdogs domain queries — no personal or off-topic responses |
| FR3.4 | MCP server registration in the profile `config.yaml` must point to the fourdogs MCP server: `mcp_servers.fourdogs-tools.url` |
| FR3.5 | All Hermes credentials (OmniRoute API key, Honcho connection, any session tokens) must be provisioned from Vault — not stored in profile config files in source control |
| FR3.6 | Hermes session continuity must be enabled via `X-Hermes-Session-Id` header routing — the hermes_session_id from Kaylee's Postgres must flow through to Honcho session tracking |

### 6.4 Fourdogs MCP Server

| ID | Requirement |
|----|-------------|
| FR4.1 | A fourdogs MCP server must be implemented, wrapping the 14 existing Kaylee tool functions and exposing them via the Model Context Protocol |
| FR4.2 | The MCP server must expose all 14 tools: `transaction_query`, `sales_summary`, `sales_query`, `order_view`, `order_update`, `order_lifecycle`, `order_create`, `inventory_lookup`, `inventory_velocity`, `customer_lookup`, `top_products`, `list_vendors`, `get_current_date` — and must delete `dispatcher.py` |
| FR4.3 | Phase 1 implementation priority: Orders tools (`order_view`, `order_update`, `order_lifecycle`, `order_create`) and Sales tools (`sales_summary`, `sales_query`, `transaction_query`) fully functional before any other tools |
| FR4.4 | All MCP tool functions must have existing test coverage maintained — the MCP adapter layer does not replace tool unit tests |
| FR4.5 | MCP server credentials (DB connection strings, EtailPet API key when added in Phase 2) must be provisioned from Vault — not hardcoded or in `.env` files in source control |
| FR4.6 | Deployment location and ownership must be decided before infrastructure sprint planning (see Open Questions — H1 from adversarial review) |

### 6.5 Kaylee Thin Proxy Refactor

| ID | Requirement |
|----|-------------|
| FR5.1 | A `USE_HERMES_AGENT` feature flag must be added to Kaylee's settings; when `false`, the existing legacy chat handler runs unchanged |
| FR5.2 | When `USE_HERMES_AGENT=true`, all chat requests must route to a new `hermes_chat_handler` that calls Hermes via `POST /v1/chat/completions` at the configured `HERMES_BASE_URL` |
| FR5.3 | The Kaylee Hermes adapter must set the `X-Hermes-Session-Id` header to the Kaylee Postgres session UUID on every request, enabling Hermes-to-Honcho session continuity |
| FR5.4 | `chat_sessions` table must be migrated to add a `hermes_session_id VARCHAR(64)` column (nullable); existing rows are unaffected |
| FR5.5 | Hermes session ID lazy provisioning: on first message in a session, call Hermes with the Kaylee session UUID as session ID; on subsequent messages, reuse the stored value |
| FR5.6 | The legacy `tools/dispatcher.py` and all inline tool calling code must remain functional and pass existing tests while `USE_HERMES_AGENT=false` — legacy path must not be degraded during the transition period |
| FR5.7 | The following environment variables must be added to Kaylee deployment config: `HERMES_BASE_URL`, `HERMES_AGENT_PROFILE`, `USE_HERMES_AGENT` |
| FR5.8 | Kaylee's updated backend must emit FD-2 compliant structured JSON logs for all Hermes adapter calls (startup_ready, per-request, and error events) |

### 6.6 Architecture Documentation

| ID | Requirement |
|----|-------------|
| FR6.1 | An `architecture.md` document must be authored as a required output of the TechPlan phase — this is a hard gate (Article 3) before any implementation stories can be authored |
| FR6.2 | `architecture.md` must cover: VM topology, Hermes+Honcho+Kaylee+OmniRoute integration data flows, MCP server deployment placement decision, Vault credential delivery path for non-k8s VM services, and OmniRoute connectivity from the hermes-fourdogs VM |
| FR6.3 | `architecture.md` must include explicit data flow statements identifying all external data transmission paths (Article 4 compliance) |

### 6.7 End-to-End Smoke Test

| ID | Requirement |
|----|-------------|
| FR7.1 | An end-to-end smoke test must be defined and must pass before `USE_HERMES_AGENT=true` is set as the default in any environment |
| FR7.2 | Smoke test must cover: web UI → Kaylee → Hermes → MCP tool call → tool response → Hermes response → Kaylee → web UI round-trip |
| FR7.3 | Smoke test must verify Honcho session continuity: second message in a session must reference context from the first message |

---

## 7. Non-Functional Requirements

### 7.1 Reliability

| ID | Requirement |
|----|-------------|
| NFR1.1 | HTTP 400 failure rate on tool-call requests must be ≤1% over any 5-consecutive-business-day period once `USE_HERMES_AGENT=true` |
| NFR1.2 | Kaylee must have a clean fallback path: setting `USE_HERMES_AGENT=false` must restore full legacy behavior within one deployment cycle, with no data loss |
| NFR1.3 | Hermes and Honcho service restarts must not corrupt session state; `honcho_fourdogs` DB must be the authoritative session store |

### 7.2 Performance

| ID | Requirement |
|----|-------------|
| NFR2.1 | Response latency for Hermes-routed queries must not exceed the 95th-percentile latency of the current direct-llamacpp path by more than 20% |
| NFR2.2 | Honcho deriver processing must be asynchronous and must not block inline chat response time |
| NFR2.3 | MCP tool calls must complete within the existing tool timeout budget defined in the Kaylee backend |

### 7.3 Security

| ID | Requirement |
|----|-------------|
| NFR3.1 | No plaintext secrets in any Hermes or Honcho config file, `.env` file, or environment variable in source control — all credentials must be provisioned from Vault paths |
| NFR3.2 | The fourdogs Hermes install must have zero shared credentials with the personal `hermes.trantor.internal` install |
| NFR3.3 | MCP server database connections must use role-scoped Patroni credentials (not superuser) |
| NFR3.4 | `DISCORD_BOT_TOKEN` (Phase 2) must be provisioned from Vault — not stored in any config file (Article 9 compliance) |
| NFR3.5 | Hermes API on port 8642 must be accessible only from the internal network (`trantor.internal`) — not exposed externally |
| NFR3.6 | Article 4 compliance: all Hermes and Honcho LLM inference routes through OmniRoute; no business-data-bearing requests may leave the `trantor.internal` network without OmniRoute governance |

### 7.4 Observability (FD-2)

| ID | Requirement |
|----|-------------|
| NFR4.1 | All new services (`hermes-fourdogs.service`, `honcho-fourdogs.service`, `honcho-fourdogs-deriver.service`, fourdogs MCP server) must emit structured JSON logs to stdout with FD-2 required fields: `timestamp` (UTC ISO 8601), `level` (DEBUG/INFO/WARN/ERROR), `service`, `message` |
| NFR4.2 | Each service must emit a `startup_ready` log line at startup |
| NFR4.3 | Each service must emit `health_degraded` and `critical_failure` events when applicable |
| NFR4.4 | Kaylee's updated Hermes adapter module must log: per-request (session_id, hermes_session_id, status), errors, and retries |
| NFR4.5 | Loki visibility check must confirm at least one startup log line queryable within 5 minutes of deployment start for each new service |

### 7.5 Business Isolation

| ID | Requirement |
|----|-------------|
| NFR5.1 | The fourdogs Hermes install must be independently relocatable as a single VM unit — no dependency on external personal infrastructure services |
| NFR5.2 | All fourdogs databases (`honcho_fourdogs`, `hermes_fourdogs` if applicable) must be explicitly named with the `fourdogs` namespace and must not be shared with personal agent services |
| NFR5.3 | Business separation must be documented in `architecture.md` as an explicit design constraint, not just a deployment choice |

### 7.6 Maintainability

| ID | Requirement |
|----|-------------|
| NFR6.1 | Hermes and Honcho must be deployed from tagged releases (not personal forks) — any required patches must be upstreamed or documented with explicit patch rationale |
| NFR6.2 | `SOUL.md` persona hot-reload must work — persona changes must not require Hermes service restart |
| NFR6.3 | MCP tool additions must be additive — adding a new tool to the MCP server must not require Kaylee backend changes |
| NFR6.4 | MCP server API contract must be documented in `architecture.md` so additional profiles (Phase 2+) can register it without Kaylee involvement |

---

## 8. Constraints and Dependencies

### 8.1 Hard Dependencies

| Dependency | Owner | Notes |
|-----------|-------|-------|
| OmniRoute (terminus-inference-gateway) | terminus | Must be production-stable and reachable from hermes-fourdogs VM before Hermes+Honcho deployment |
| Patroni cluster (10.0.0.56:5432) | terminus/infra | Required for `honcho_fourdogs` DB provisioning |
| terminus.infra OpenTofu + Ansible | terminus/infra | VM provisioning follows existing Hermes install pattern |
| Vault KV store | infra | All credential delivery for non-k8s VM services — Vault Agent or API pull pattern TBD |

### 8.2 Architecture Constraints

| Constraint | Rationale |
|-----------|-----------|
| Hermes+Honcho on dedicated VM (not k3s) | Business isolation — independently relocatable, no k3s namespace entanglement |
| No direct llamacpp for business functions | Grammar-slot bug makes direct inference unreliable for tool-heavy requests |
| OmniRoute as sole LLM gateway | Governed routing, failover, single credential surface |
| Vault for all secrets | Article 9 compliance; no plaintext in source control or config files |
| `architecture.md` before stories | Article 3 hard gate |

### 8.3 Constitution Compliance

| Article | Gate | Status | Notes |
|---------|------|--------|-------|
| Article 3 — Architecture Documentation | Hard | Open (TechPlan) | `architecture.md` required before stories |
| Article 4 — No Data Exfiltration | Hard | Compliant by design | All inference through OmniRoute on internal network |
| Article 7 — TDD Red-Green | Hard | Dev phase | Story-level requirement |
| Article 8 — BDD Acceptance Criteria | Hard | Dev phase | Story-level requirement |
| Article 9 — Security First | Hard | Must verify | Vault delivery for non-k8s VM is open (H2) |
| FD-1 — Doc Sync on develop merges | Hard | Dev phase | `app-arch.md` update required on develop merge |
| FD-2 — Loki Structured Logging | Hard | NFR4 above | All new services must satisfy |

---

## 9. Open Questions

These map to the adversarial review findings and must be resolved before or during TechPlan:

| ID | Question | Blocking | Target Phase |
|----|----------|----------|--------------|
| OQ1 | Does OmniRoute expose `/v1/embeddings`? What dimension count? | Honcho first deploy (EMBEDDING_VECTOR_DIMENSIONS schema-locked) | TechPlan |
| OQ2 | What is the Vault credential delivery pattern for non-k8s VMs (hermes-fourdogs)? Vault Agent sidecar or Vault API pull? | Infrastructure sprint planning | TechPlan |
| OQ3 | Where does the MCP server run? VM-local (systemd), k3s (fourdogs namespace), or embedded in Kaylee? | Infrastructure sprint planning | TechPlan |
| OQ4 | Is OmniRoute accessible from the hermes-fourdogs VM on the host network? NodePort? metallb IP? Tailscale? | LLM connectivity for Hermes | TechPlan |
| OQ5 | Which Honcho version/tag to pin? v3.0.7 is known-good; current main has breaking config changes | Deployment config correctness | TechPlan |
| OQ6 | Does Hermes Agent require its own Postgres state DB beyond Honcho? | DB provisioning scope | TechPlan |
| OQ7 | What happens to Betsy's existing sessions at feature flag flip? Cold Honcho session (no prior context) is acceptable — confirm and document | Session migration story AC | Stories |
| OQ8 | Fallback LLM config if OmniRoute is not reachable from hermes-fourdogs VM on day 1 of provisioning | Infrastructure provisioning validation | TechPlan |

---

## 10. Phase Roadmap

### Phase 1 (this feature — in scope)

- Dedicated VM provisioning (`terminus.infra`)
- Honcho + Hermes deploy with isolated DBs and OmniRoute LLM
- Fourdogs MCP server: 14 tools (Orders + Sales prioritized)
- Kaylee thin proxy with `USE_HERMES_AGENT` feature flag
- DB migration: `hermes_session_id` column on `chat_sessions`
- `architecture.md` (TechPlan gate)
- End-to-end smoke test + 5-day stability validation

### Phase 2 (follow-on feature)

- Discord integration: fourdogs store server → kaylee profile
- EtailPet API tools in MCP server
- Additional Hermes agent profiles (inventory-watch, etc.)

### Phase 3 (future)

- Email read/send via emailfetcher API
- `ops-summary` daily business snapshot agent
- Honcho peer model tuning and dialectic optimization

---

## 11. Glossary

| Term | Definition |
|------|-----------|
| Hermes Agent | Nous Research open-source agent gateway. OpenAI-compatible API on port 8642. Owns conversation management, tool dispatch via MCP, multi-channel connectors, Honcho integration. |
| Honcho | Plastic Labs open-source memory platform. FastAPI on port 8000. Builds persistent peer representations via async deriver. Dialectic API enriches responses with accumulated domain knowledge. |
| OmniRoute | terminus-inference-gateway with provider routing and failover. Used by both Hermes and Honcho as the sole LLM backend. |
| MCP server | Fourdogs Model Context Protocol server. Wraps Kaylee's 14 tool functions; Hermes discovers and calls them via the MCP protocol. |
| SOUL.md | Hermes Agent persona file for a profile. Hot-reloaded on each message — no restart required for persona changes. |
| Thin proxy | Kaylee's post-refactor role: receive message from web UI, POST to Hermes, stream response back. No AI logic in Kaylee backend. |
| Strangler Fig | Migration pattern: feature flag (`USE_HERMES_AGENT`) routes traffic to new Hermes path or legacy path; legacy code removed after validation. |
| hermes_session_id | New Postgres column on `chat_sessions`. Stores the Kaylee session UUID used as `X-Hermes-Session-Id` header for Honcho session continuity. |
| Business isolation | Architectural constraint: `hermes-fourdogs.trantor.internal` has no shared state, credentials, or databases with `hermes.trantor.internal` (personal install). |
