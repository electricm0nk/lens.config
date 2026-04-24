---
feature: real-data-grounding
doc_type: epics
status: dev-ready
goal: >
  Eliminate the three root-cause gaps in Kaylee's central-app behaviour: hallucinated stub data,
  broken cross-DB tool access, and missing order context. Stories span kaylee-agent, fourdogs-central,
  and terminus.infra. No new user-facing API surface is introduced — this is purely correctness work.
key_decisions:
  - Central DB pool (Story 5.1) must land and be deployed before any cross-DB tool story (5.3, 5.4, 5.5, 5.6)
  - Infra grant (Story 5.2) must land before 5.3–5.6 can be verified end-to-end
  - Tool role change (Story 5.7) is independent and can ship in parallel with any story
  - Order context injection (Story 5.8) requires changes in both kaylee-agent and fourdogs-central
open_questions: []
depends_on:
  - fourdogs-kaylee-agent Story 1.7 (cross-DB grant) superseded by 5.2 in this feature
blocks: []
updated_at: "2026-04-23T00:00:00Z"
---

# Epics: real-data-grounding

## Epic Map

| Epic | Title | Repo(s) | Stories | Outcome |
|---|---|---|---|---|
| EPIC-5 | Kaylee Real Data & Context Grounding | kaylee-agent, fourdogs-central, terminus.infra | 5.1 – 5.8 | All data tools query real data; Kaylee knows the order she is helping with |

---

## EPIC-5: Kaylee Real Data & Context Grounding

**Goal:** Kaylee answers questions about real store data from the actual fourdogs_central database and
knows which order she is assisting with when accessed via the central app. Stub tools are replaced with
live queries. Tool message injection is format-safe. The persona failures caused by error-noise injection
(fixed in the dispatcher bug fix) are the last known personality regression.

**Repos affected:**
- `fourdogs-kaylee-agent` — tool rewrites, central DB pool, message assembly update, tool role fix
- `fourdogs-central` — order context injection into kaylee session creation
- `terminus.infra` — Postgres grant for `fourdogs_kaylee_rw` on fourdogs_central tables

**Dependency order:**
```
5.1 (central pool)
5.2 (infra grant)
    ↓ both complete before:
5.3 inventory_velocity fix
5.4 top_products fix
5.5 real sales_summary
5.6 real customer_lookup

5.7 tool role fix       ← independent, can ship any time
5.8 order context       ← independent, can ship any time
```

### Story 5.1: Central DB Connection Pool in kaylee-agent

**Repo:** fourdogs-kaylee-agent  
**Status:** backlog  
**Priority:** high (blocks 5.3–5.6)

As a developer, I want kaylee-agent to maintain a second asyncpg connection pool for `fourdogs_central`,
so that data tools can query real fourdogs_central tables without FDW or cross-DB hacks.

### Story 5.2: Postgres Grant — fourdogs_kaylee_rw on fourdogs_central tables

**Repo:** terminus.infra  
**Status:** backlog  
**Priority:** high (blocks 5.3–5.6)

As a DBA, I want `fourdogs_kaylee_rw` granted CONNECT + SELECT on the required fourdogs_central tables,
so that kaylee's tools can query real data without permission errors.

### Story 5.3: Fix inventory_velocity — real cross-DB query

**Repo:** fourdogs-kaylee-agent  
**Status:** backlog  
**Priority:** high  
**Depends on:** 5.1, 5.2

As an AI agent, I want `inventory_velocity` to query real `items` and `order_items` data from fourdogs_central
using the central pool, so that Kaylee gives accurate inventory risk information.

### Story 5.4: Fix top_products — correct table reference + real cross-DB query

**Repo:** fourdogs-kaylee-agent  
**Status:** backlog  
**Priority:** high  
**Depends on:** 5.1, 5.2

As an AI agent, I want `top_products` to query real `items` and `order_items` data with the correct
column aliases, so that Kaylee correctly ranks what is actually selling.

### Story 5.5: Real sales_summary — query fourdogs_central.sales

**Repo:** fourdogs-kaylee-agent  
**Status:** backlog  
**Priority:** medium  
**Depends on:** 5.1, 5.2

As an AI agent, I want `sales_summary` to return real revenue and order counts from the `sales` table,
so that Betsy gets an accurate business pulse instead of the stub's hardcoded $142,500.

### Story 5.6: Real customer_lookup — query fourdogs_central.sales

**Repo:** fourdogs-kaylee-agent  
**Status:** backlog  
**Priority:** medium  
**Depends on:** 5.1, 5.2

As an AI agent, I want `customer_lookup` to find real customer records in the `sales` table by name,
so that Betsy can look up actual purchase history for a customer.

### Story 5.7: Tool context role fix — user-context injection instead of role:tool

**Repo:** fourdogs-kaylee-agent  
**Status:** backlog  
**Priority:** medium  
**Depends on:** none

As a developer, I want tool results injected as `role: "user"` context blocks with a bracketed
header, so that OpenAI never receives a `role: "tool"` message without a required `tool_call_id`,
preventing potential 400 errors as tool success rates improve.

### Story 5.8: Order context injection from fourdogs-central

**Repo:** fourdogs-central (PostKayleeMessage handler) + fourdogs-kaylee-agent (message_assembly)  
**Status:** backlog  
**Priority:** medium  
**Depends on:** none

As the AI agent, I want to receive order context (vendor name, order date, item count, budget) in
the session metadata when fourdogs-central creates my session, so that I can help Betsy without her
having to re-explain which order she is working on.
