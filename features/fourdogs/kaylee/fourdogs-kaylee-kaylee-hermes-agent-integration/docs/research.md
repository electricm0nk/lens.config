# Research: Kaylee → Hermes Agent Integration

**Feature:** fourdogs-kaylee-kaylee-hermes-agent-integration
**Phase:** preplan
**Date:** 2026-05-30

---

## 1. Stack Overview — Hermes Agent + Honcho

The personal install at `hermes.trantor.internal` runs **two distinct products**:

| Product | Vendor | Role | Port |
|---|---|---|---|
| **Hermes Agent** | Nous Research (MIT) | Agent shell, OpenAI-compatible API, channel connectors (Discord plugin etc), tool dispatch, persona | 8642 |
| **Honcho** | Plastic Labs (MIT, v3+) | Memory platform, dialectic reasoning, peer representation, FastAPI REST | 8000 |

**Architecture decision:** Both components are required for fourdogs. Honcho-only
would give Kaylee memory but no agent framework — Kaylee's code would still own
tool dispatch, conversation management, persona, and channel routing. The full
Hermes+Honcho stack offloads all of that: Kaylee becomes a pure thin proxy.

Sources: live inspection of `hermes.trantor.internal`; `github.com/plastic-labs/honcho`;
`github.com/NousResearch/hermes-agent`; `https://hermes-agent.nousresearch.com/docs`;
`https://honcho.dev/docs`.

---

## 2. Hermes Agent — Architecture and Capabilities

Hermes Agent is the **agent gateway layer**. It owns:

- **OpenAI-compatible API** (port 8642) — the surface Kaylee calls (`POST /v1/chat/completions`)
- **Agent profile management** — profiles are full sub-home directories (`profiles/<name>/`),
  each with own `config.yaml`, `sessions/`, `skills/`, `state.db`, and `workspace/`
- **Tool dispatch** — Kaylee's 14 business tools become registered Hermes skills.
  Hermes decides when to call them, collects results, assembles context.
- **Channel connectors** — native Discord integration and others. The fourdogs
  Discord server wires directly to the `kaylee` agent profile; no Kaylee backend
  involvement needed for Discord messages.
- **Honcho integration** — Hermes calls Honcho's dialectic API internally on every
  turn, enriching responses with accumulated peer knowledge.

### 2.1 Agent Profiles

A profile is a **full sub-home directory** at `profiles/<name>/`. Confirmed
structure from personal install:

```
profiles/
  gunny/
    config.yaml        ← model/provider overrides (e.g. OmniRoute)
    sessions/
    skills/
    state.db
    workspace/
    SOUL.md            ← persona definition, hot-reloaded
    channel_directory.json
    auth.json
```

`config.yaml` for the fourdogs `kaylee` profile will follow the `gunny` pattern
(OmniRoute provider at `inference-gateway.trantor.internal`).
`SOUL.md` defines personality — loaded fresh each message, no restart needed.

The fourdogs Hermes install will host at minimum the `kaylee` profile, with
capacity to add additional profiles (inventory-watch, email-processor, ops-summary)
without any Kaylee code changes.

### 2.2 Discord Connector

Hermes Discord support is a **plugin** at `plugins/platforms/discord/` (not a
core gateway adapter). It uses `discord.py` and exposes a `DISCORD_BOT_TOKEN`
configured connector.

**Confirmed from personal install:** `channel_directory.json` already maps Discord
channels to agent profiles (`vanderspeigle`, `gunny`, `shepherd` channels in the
"Trantor" Discord guild). Discord IS a first-class supported channel — just
installed as a plugin rather than built into the gateway core.

Wire the fourdogs store Discord server → `kaylee` profile:
- Staff messages in configured Discord channels route through Hermes
- Hermes dispatches tools, queries Honcho memory, responds in-channel
- No Kaylee web UI required for Discord interactions
- Same agent profile, same memory, same tool access — channel is transparent

Required: `DISCORD_BOT_TOKEN` provisioned from Vault (not in config files).
Configuration lives in the profile's `channel_directory.json` and plugin config.
Kaylee backend is not involved in Discord interactions at all.

### 2.3 Multi-Agent Capability

A single fourdogs Hermes install can host multiple profiles sharing one Honcho
backend. Planned agent profiles:

| Profile | Purpose | Priority |
|---|---|---|
| `kaylee` | Primary assistant — orders, sales, customers, inventory | Phase 1 |
| `inventory-watch` | Proactive low-stock and velocity alerts | Phase 2 |
| `email-processor` | Triage and draft responses to customer emails | Phase 3 |
| `ops-summary` | Daily business snapshot for Betsy | Phase 3 |

Each profile has its own tool whitelist and system prompt. Honcho's peer model
means accumulated knowledge about fourdogs domain is shared across all profiles.

---

## 3. Honcho — Memory Platform Architecture

Honcho is the **memory layer** for the full agent fleet. It provides:

- **Continuous learning:** Every message passes through the deriver — a background
  worker that extracts observations about peers and builds representations over time.
- **Dialectic API:** On-demand reasoning endpoint. Hermes calls this on every turn;
  given a query + session context, Honcho searches accumulated peer knowledge and
  synthesizes a contextually-aware response that Hermes folds back into its reply.
- **Dreaming:** Async consolidation of peer representations during idle periods.
- **Session/peer/workspace model:** Conversations are sessions. Peers are named
  entities (users, the kaylee agent, other agents). All agents share one workspace
  (`fourdogs`), each with their own peer identity.

### 3.1 Personal Install Facts (confirmed from live hermes.trantor.internal)

- Honcho v3.0.7 at `/opt/honcho/repo/`
- Two systemd services: `honcho.service` (FastAPI, port 8000) + `honcho-deriver.service`
- Personal DB: `honcho_hermes` on Patroni cluster
- Config: `HONCHO_AUTH_ENABLED=false`, pgvector extension, llamacpp direct for LLM

---

## 4. Honcho API Contract (confirmed from live OpenAPI on personal install)

Base URL: `http://<host>:8000`

### 4.1 Dialectic Chat (Hermes calls this; Kaylee does not call it directly)

```
POST /v3/workspaces/{workspace_id}/peers/{peer_id}/chat
```

**Request body (`DialecticOptions`):**
```json
{
  "query": "string (required, max 10000 chars)",
  "session_id": "string (optional)",
  "target": "string (optional)",
  "stream": false,
  "reasoning_level": "minimal | low | medium | high | max"
}
```

**Response:** `{ "content": "string" }`

`stream: true` returns SSE (`text/event-stream`).

### 4.2 Add Messages to Session

```
POST /v3/workspaces/{workspace_id}/sessions/{session_id}/messages
```

**Request body (`MessageBatchCreate`):**
```json
{
  "messages": [
    {
      "content": "string (required, max 25000 chars)",
      "peer_id": "string (required)",
      "metadata": {},
      "created_at": "datetime (optional)"
    }
  ]
}
```

Max 100 messages per batch.

### 4.3 Other Endpoints

Full CRUD for: workspaces, sessions, peers, messages, conclusions, summaries.
Queue status endpoint for deriver processing state.
Auth: `HTTPBearer` (disabled with `AUTH_USE_AUTH=false` for internal deployments).

---

## 5. LLM Backend — OmniRoute (Not Direct Qwen3)

**Decision:** All Hermes and Honcho LLM calls route through
**terminus-inference-gateway with OmniRoute provider routing**.

**Rationale:** Direct llamacpp/Qwen3 on worker-07 has a confirmed grammar-slot
exhaustion bug that causes ~30% failure rate with tool-call grammars (12+ tools).
Business-critical functions cannot tolerate this. OmniRoute provides:

- Stable primary routing (cloud or local vLLM)
- Automatic failover when local provider is unhealthy
- Consistent model API surface regardless of backend
- A single credential and endpoint for both Hermes and Honcho config

**Gateway endpoint for Hermes/Honcho configuration:**
```
http://terminus-inference-gateway.<namespace>.svc.cluster.local/v1
```

Or internal hostname if Hermes VM needs direct access (not in k3s):
```
http://inference-gateway.trantor.internal:<port>/v1
```

**Model selection note:** OmniRoute can target any configured provider/model.
For Hermes tool dispatch and Honcho deriver/dialectic, a capable reasoning model
with tool calling support is required. OmniRoute's routing policy will determine
whether this hits local vLLM, llamacpp, or a cloud provider — transparently.

**Honcho LLM config example (OmniRoute via openai transport):**

```bash
LLM_OPENAI_API_KEY=<vault-provisioned-gateway-key>

# Deriver
DERIVER_ENABLED=true
DERIVER_WORKERS=2
DERIVER_MODEL_CONFIG__TRANSPORT=openai
DERIVER_MODEL_CONFIG__MODEL=<omniroute-model-id>
DERIVER_MODEL_CONFIG__OVERRIDES__BASE_URL=http://inference-gateway.trantor.internal:<port>/v1

# Dialectic levels (low is default for most turns)
DIALECTIC_LEVELS__low__MODEL_CONFIG__TRANSPORT=openai
DIALECTIC_LEVELS__low__MODEL_CONFIG__MODEL=<omniroute-model-id>
DIALECTIC_LEVELS__low__MODEL_CONFIG__OVERRIDES__BASE_URL=http://inference-gateway.trantor.internal:<port>/v1

# Summary and dream: same pattern
# Embeddings: see section 7 (open question)
```

---

## 6. Honcho v3 Configuration (current docs — differs from personal install)

Priority: env vars > `.env` > `config.toml` > built-in defaults.

**Breaking config changes from v3.0.7 (personal install) to current v3:**

| Old key (personal install v3.0.7) | New key (current v3 docs) |
|---|---|
| `HONCHO_AUTH_ENABLED=false` | `AUTH_USE_AUTH=false` |
| `DB__CONNECTION_URI=...` | `DB_CONNECTION_URI=...` |
| `DERIVER_FLUSH_ENABLED=true` | `DERIVER_ENABLED=true` |

**Critical:** Honcho v3 current **refuses to start without LLM configuration**.
Must configure at minimum a working transport + model before first startup.

**Core config:**
```bash
AUTH_USE_AUTH=false
LOG_LEVEL=INFO
EMBED_MESSAGES=true
NAMESPACE=honcho-fourdogs
DB_CONNECTION_URI=postgresql+psycopg://honcho_fourdogs_user:...@10.0.0.56:5432/honcho_fourdogs
VECTOR_STORE_TYPE=pgvector
EMBEDDING_VECTOR_DIMENSIONS=1536        # schema-pinned at first migration — do NOT change after
CACHE_ENABLED=true
CACHE_URL=redis://localhost:6379/0
```

---

## 7. Fourdogs Hermes+Honcho Deployment Plan

### 7.1 Infrastructure

- **Hosting:** New dedicated Proxmox VM on `trantor.internal`
  - Suggested hostname: `hermes-fourdogs.trantor.internal`
  - Pattern: follows existing `hermes.trantor.internal` personal install
  - Provisioning via OpenTofu + Ansible in `terminus.infra`
  - VM is relocatable — physical location on trantor is fine; business isolation
    is logical (own credentials, own DBs, own config)

- **Services on VM:**
  - `hermes-fourdogs.service` — Hermes Agent, OpenAI-compatible API port 8642
  - `honcho-fourdogs.service` — Honcho FastAPI, port 8000
  - `honcho-fourdogs-deriver.service` — Honcho background deriver worker
  - `redis-fourdogs` — Redis for Honcho caching (local on VM or shared)

### 7.2 Databases (Patroni cluster, 10.0.0.56:5432)

```sql
-- Honcho memory DB
CREATE DATABASE honcho_fourdogs;
CREATE USER honcho_fourdogs_user WITH ENCRYPTED PASSWORD '...';
GRANT ALL PRIVILEGES ON DATABASE honcho_fourdogs TO honcho_fourdogs_user;
\c honcho_fourdogs
CREATE EXTENSION IF NOT EXISTS vector;

-- Hermes agent state DB (if Hermes requires one — verify from personal install)
CREATE DATABASE hermes_fourdogs;
CREATE USER hermes_fourdogs_user WITH ENCRYPTED PASSWORD '...';
GRANT ALL PRIVILEGES ON DATABASE hermes_fourdogs TO hermes_fourdogs_user;
```

Vault paths: `secret/fourdogs/dev/postgres/honcho`, `secret/fourdogs/dev/postgres/hermes`

### 7.3 Workspace and Peer Bootstrap

One-time setup before Kaylee can call Hermes. Script creates:
- Workspace: `fourdogs`
- Peers: `kaylee` (assistant), `user-betsy` (or user-per-id model TBD)
- Verifies dialectic endpoint responds before Kaylee deploy

### 7.4 Install Steps (Honcho, following personal install pattern)

```bash
git clone https://github.com/plastic-labs/honcho.git /opt/honcho-fourdogs/repo
cd /opt/honcho-fourdogs/repo
# Pin to known working tag or use latest main (open question — see Section 9)
uv sync
cp .env.template .env
# Edit .env with fourdogs config (DB, LLM via OmniRoute, auth disabled)
uv run alembic upgrade head
# Start systemd services
```

---

## 8. Kaylee Integration Design (Thin Proxy)

### 8.1 New Architecture

```
User (web UI)  →  Kaylee backend  →  POST /v1/chat/completions (Hermes API, port 8642)
                                         ↓
Discord (store server)  →  Hermes discord plugin  →  Same Hermes agent profile
                                         ↓
                               Hermes: tool dispatch + persona
                                         ↓
                               Honcho: dialectic + memory enrichment
                                         ↓
                               OmniRoute: LLM inference (stable, routed)
```

Kaylee backend calls Hermes. Hermes handles all intelligence, tool calls, memory,
and channel routing. Kaylee's session management in Postgres survives (UI needs it)
but carries no AI logic.

### 8.2 Kaylee → Hermes API Payload

**Confirmed from personal install source (`gateway/platforms/api_server.py`):**
Hermes exposes a standard **OpenAI-compatible API** on port 8642.

```
POST http://hermes-fourdogs.trantor.internal:8642/v1/chat/completions
Content-Type: application/json
X-Hermes-Session-Id: <kaylee-session-uuid>   ← enables session continuity

{
  "model": "kaylee",
  "messages": [
    {"role": "user", "content": "<user message text>"}
  ],
  "stream": true
}
```

Response: SSE (`text/event-stream`) when `stream: true`, or standard OpenAI
chat completion JSON when `stream: false`.

Session continuity: pass `X-Hermes-Session-Id` header on every turn. Hermes
propagates this through to Honcho session tracking automatically.

Alternate stateful endpoint: `POST /v1/responses` with `previous_response_id`
(OpenAI Responses API format).

### 8.3 Session Mapping

Kaylee stores a Hermes session ID in its Postgres session table:

```sql
ALTER TABLE chat_sessions
  ADD COLUMN hermes_session_id VARCHAR(64);
```

Lazy provisioning: on first message, call Hermes with the Kaylee session UUID as
`session_id`. Hermes propagates this through to Honcho session tracking.

### 8.4 Feature Flag Strategy (Strangler Fig)

```python
if settings.USE_HERMES_AGENT:
    return await hermes_chat_handler(request, session, user_message)
else:
    return await legacy_chat_handler(request, session, user_message)
```

`.env` addition:
```bash
HERMES_BASE_URL=http://hermes-fourdogs.trantor.internal:8642
HERMES_AGENT_PROFILE=kaylee
USE_HERMES_AGENT=true
```

### 8.5 Kaylee Tools → MCP Server (Not "Skills")

**Important terminology distinction confirmed from hermes-agent source:**

- **Skills** = Markdown documents (`SKILL.md` in `skills/` dir). Behavioral/instructional
  prompts loaded as context. They shape how the agent reasons — NOT callable functions.
- **Tools** = Python functions registered via `registry.register()` or exposed via MCP.
  These are what the LLM actually invokes during tool-call turns.

**Integration pattern for fourdogs custom tools: MCP server.**

Hermes supports MCP (Model Context Protocol) via `mcp_tool.py`. External tools are
exposed via an MCP server process; Hermes discovers and registers them at startup.
Config in profile `config.yaml`:

```yaml
mcp_servers:
  fourdogs-tools:
    url: "http://localhost:8700/mcp"   # or stdio transport
    timeout: 30
```

**Fourdogs MCP server:** A lightweight FastAPI/starlette MCP server that wraps
Kaylee's 14 existing tool functions and exposes them as MCP tools. Lives in the
Kaylee repo or as a standalone `fourdogs-mcp-server` service on the hermes-fourdogs VM.

| Current Kaylee tool | MCP tool name | Domain |
|---|---|---|
| `transaction_query.py` | `transaction_query` | Sales |
| `sales_summary.py` | `sales_summary` | Sales |
| `sales_query.py` | `sales_query` | Sales |
| `order_view.py` | `order_view` | Orders |
| `order_update.py` | `order_update` | Orders |
| `order_lifecycle.py` | `order_lifecycle` | Orders |
| `order_create.py` | `order_create` | Orders |
| `inventory_lookup.py` | `inventory_lookup` | Inventory |
| `inventory_velocity.py` | `inventory_velocity` | Inventory |
| `customer_lookup.py` | `customer_lookup` | Customers |
| `top_products.py` | `top_products` | Products |
| `list_vendors.py` | `list_vendors` | Vendors |
| `get_current_date.py` | `get_current_date` | Utility |
| `dispatcher.py` | **deleted** | Replaced by Hermes routing |

All 14 have existing test coverage. MCP wrapping is a thin adapter, not a rewrite.
EtailPet API tools and email tools are added to the MCP server (not to Kaylee backend).

**Behavioral skills (SKILL.md)** for domain knowledge (fourdogs products, EtailPet
workflow, Betsy's preferences) are separate `SKILL.md` docs installed into the
kaylee profile's `skills/` directory via `hermes skills install`.

---

## 9. Open Questions

1. ~~**Hermes ACP API contract:**~~ **RESOLVED** — Port 8642, OpenAI-compatible
   `/v1/chat/completions`, session via `X-Hermes-Session-Id` header. Confirmed
   from `gateway/platforms/api_server.py` on personal install.

2. ~~**Hermes skill/tool registration mechanism:**~~ **RESOLVED** — Two distinct systems:
   (a) **Skills** = `SKILL.md` markdown docs in `skills/<category>/<name>/SKILL.md`,
   installed via hub. Behavioral prompts, not callable functions.
   (b) **Tools** = Python functions registered via `registry.register()` in `tools/`,
   or exposed via external **MCP servers** declared in profile `config.yaml` under
   `mcp_servers`. Fourdogs business tools will use MCP server pattern (see §8.5).

3. ~~**Discord connector config:**~~ **RESOLVED** — Plugin at `plugins/platforms/discord/`
   using `discord.py`. Bot-token-based (`DISCORD_BOT_TOKEN`). Channels mapped in
   profile's `channel_directory.json`. Confirmed working on personal install.

4. **Honcho version pin:** Personal install is v3.0.7. Current main has breaking
   config changes. Decision: deploy from latest main or pin to a tested tag?

5. **Embeddings provider:** Does OmniRoute expose an embeddings endpoint
   (`text-embedding-3-small`, 1536 dims)? If not, a separate provider is needed
   for Honcho's vector store. `EMBEDDING_VECTOR_DIMENSIONS=1536` is schema-pinned
   at first migration — cannot change afterward.

6. **Hermes state database:** Does Hermes Agent require its own Postgres DB for
   agent state, or does it delegate all persistence to Honcho? Verify from personal
   install service config and source.

7. **Redis:** VM-local Redis or reuse fourdogs k3s Redis if one exists? Redis is
   optional but recommended for Honcho production caching.

8. **OmniRoute gateway accessibility from VM:** Hermes VM is on the host network
   at `hermes-fourdogs.trantor.internal`. The inference-gateway runs in k3s. Confirm
   the gateway is reachable from the VM (NodePort, Tailscale, or metallb IP).

---

## 10. Constitution Compliance Checkpoints

Hard gates applicable to this feature's artifacts and implementation:

- **Article 3 (architecture.md required):** Full architecture document covering
  Hermes+Honcho deployment, Kaylee proxy integration, Discord wiring, and
  OmniRoute LLM routing must be produced before any implementation stories.
- **Article 4 (no confidential data exfiltration):** Hermes and Honcho stay on
  `trantor.internal`. OmniRoute is the governed LLM gateway. Discord connector
  sends messages to Hermes; no raw business data leaves the internal network.
  Discord bot token stored in Vault — not in config files.
- **Article 7 (TDD red-green):** Story implementation follows red-green-refactor.
  Test coverage required for Hermes proxy handler, session mapping, and each
  ported Hermes skill.
- **Article 8 (BDD GWT scenarios):** Each story requires Given/When/Then acceptance
  scenarios before implementation begins.
- **Article 9 (security — credentials via Vault):** Hermes DB credentials, Honcho
  DB credentials, OmniRoute gateway API key, Discord bot token, EtailPet API key —
  all provisioned from Vault via ESO. No plaintext credentials in `.env` or configs
  committed to any repo.
- **FD-1 (doc sync):** Merges to develop that change routing or tool integration
  must update `docs/fourdogs/app-arch.md`.
- **FD-2 (Loki logging):** The new `hermes-fourdogs` VM hosts three new deployable
  services. All must emit structured JSON logs; Promtail config on the VM needs
  updating. Kaylee's updated thin-proxy backend must continue FD-2 structured
  logging.
