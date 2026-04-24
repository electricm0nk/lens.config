# Story 5.1: Central DB Connection Pool in kaylee-agent

**Feature:** real-data-grounding  
**Epic:** 5 — Kaylee Real Data & Context Grounding  
**Repo:** fourdogs-kaylee-agent  
**Status:** backlog  
**Priority:** high  
**Estimate:** M  
**Blocks:** Stories 5.3, 5.4, 5.5, 5.6  

---

## User Story

As a developer,
I want kaylee-agent to maintain a second asyncpg connection pool connecting to `fourdogs_central`,
so that data tools can run real queries against fourdogs_central without FDW or separate service calls.

---

## Context

The kaylee-agent currently connects only to `fourdogs_kaylee` (its own DB). The four business data tools
(`inventory_velocity`, `top_products`, `sales_summary`, `customer_lookup`) all need to read from
`fourdogs_central`. PostgreSQL does not support cross-database queries in a single connection.

The simplest correct solution is a second pool at startup authenticated as `fourdogs_kaylee_rw` against
the `fourdogs_central` database. The grant that allows this connection is applied in Story 5.2.

**Why not FDW?** Foreign Data Wrapper adds operational overhead (FDW config, remote server mapping, user
mapping) and makes the schema harder to reason about. A second pool is simpler, testable, and mirrors
exactly how other services work in this stack.

**Postgres host:** `10.0.0.56` (Patroni)  
**Existing pool:** connects to `fourdogs_kaylee` DB as `fourdogs_kaylee_rw`  
**New pool:** connects to `fourdogs_central` DB as `fourdogs_kaylee_rw`  
**Vault secret path:** `secret/terminus/fourdogs/kaylee` (dev: `secret/terminus/dev/fourdogs/kaylee`)

---

## Acceptance Criteria

**Given** `CENTRAL_DATABASE_URL` is set and the `fourdogs_kaylee_rw` role has CONNECT on `fourdogs_central`,
**When** the FastAPI lifespan starts,
**Then** `app.state.central_pool` is an asyncpg pool (min=1, max=5) connected to `fourdogs_central`

**Given** `CENTRAL_DATABASE_URL` is missing from environment,
**When** the FastAPI lifespan starts,
**Then** the app fails fast with a clear error naming `CENTRAL_DATABASE_URL` as missing (same pattern as existing `validate_env()`)

**Given** `CENTRAL_DATABASE_URL` is set but the database is unreachable,
**When** the FastAPI lifespan starts,
**Then** the app fails fast with a connection error logged at ERROR level; the app does not start

**Given** the app shuts down,
**When** the lifespan teardown runs,
**Then** `central_pool.close()` is called, matching the existing `pool.close()` teardown

**Given** a `ToolEntry` with `needs_central_pool=True`,
**When** `dispatch_tools` is called,
**Then** `central_pool` is passed as the first argument to the tool function instead of `pool`

**Given** the `dispatch_tools` function,
**When** `central_pool` is `None` and a tool with `needs_central_pool=True` is triggered,
**Then** the tool is skipped (same silent-skip behavior as exception path)

---

## Technical Notes

### Files to change

**`src/kaylee_agent/config.py`**  
Add `CENTRAL_DATABASE_URL` to the required env var list in `validate_env()`.

**`src/kaylee_agent/db.py`**  
Add a `create_central_pool()` function alongside the existing `create_pool()`. Identical signature,
different env var: reads `CENTRAL_DATABASE_URL`. Same pool sizing: min=1, max=5 (lighter than the
primary pool since writes never go here).

**`src/kaylee_agent/app.py`** (lifespan)
```python
# After existing pool init:
app.state.central_pool = await db.create_central_pool()
# In teardown:
if app.state.central_pool is not None:
    await app.state.central_pool.close()
```

**`src/kaylee_agent/tools/dispatcher.py`**  
Extend `ToolEntry` with a new field and update `dispatch_tools` signature:
```python
@dataclass
class ToolEntry:
    name: str
    trigger_fn: Callable[[str], bool]
    tool_fn: Callable[..., Any]
    needs_pool: bool = False           # existing — kaylee DB pool
    needs_central_pool: bool = False   # new — fourdogs_central pool

async def dispatch_tools(
    messages: list[dict[str, Any]],
    user_message: str,
    registry: list[ToolEntry] | None = None,
    pool: Any = None,
    central_pool: Any = None,          # new param
) -> list[dict[str, Any]]:
```

Call logic in the dispatch loop:
```python
if entry.needs_central_pool:
    if central_pool is None:
        continue  # skip silently — pool not ready
    raw = entry.tool_fn(central_pool)
elif entry.needs_pool:
    raw = entry.tool_fn(pool)
else:
    raw = entry.tool_fn()
```

**`src/kaylee_agent/routes/chat.py`** — pass `central_pool` to `dispatch_tools`:
```python
central_pool = request.app.state.central_pool
messages = await dispatch_tools(messages, body.message, pool=pool, central_pool=central_pool)
```

### Vault / ESO / Helm changes

**Vault (done by Story 5.2 as part of infra grant):**  
Add `CENTRAL_DATABASE_URL` to `secret/terminus/fourdogs/kaylee` and `secret/terminus/dev/fourdogs/kaylee`.  
Value format: `postgresql://fourdogs_kaylee_rw:<password>@10.0.0.56:5432/fourdogs_central`

**ESO ExternalSecret** (`deploy/helm/kaylee-agent/templates/externalsecret.yaml`):  
Add `CENTRAL_DATABASE_URL` to the `.spec.data` block alongside existing `DATABASE_URL`.

**Helm values** (`deploy/helm/kaylee-agent/values.yaml`):  
Add `CENTRAL_DATABASE_URL: ""` placeholder under `env:` (sourced from ESO, not hardcoded).

### Tests

- `tests/test_dispatcher.py`: Add test that `needs_central_pool=True` tool receives `central_pool`; add test that tool is skipped when `central_pool=None`
- `tests/test_db.py` or new `tests/test_central_pool.py`: unit-test `create_central_pool()` reads `CENTRAL_DATABASE_URL`
- No integration tests required for this story — the grant is in Story 5.2

---

## Definition of Done

- [ ] `CENTRAL_DATABASE_URL` validated at startup; app fails fast if missing
- [ ] `app.state.central_pool` initialized in lifespan; torn down on shutdown
- [ ] `ToolEntry.needs_central_pool` field added; dispatcher routes correctly
- [ ] `dispatch_tools` accepts `central_pool` param; skips tool silently if `None`
- [ ] ESO ExternalSecret updated to include `CENTRAL_DATABASE_URL`
- [ ] Vault secret updated (dev + prod) — coordinate with Story 5.2
- [ ] Unit tests pass; CI green
- [ ] Deployed to `fourdogs-kaylee-dev`; `CENTRAL_DATABASE_URL` secret resolves; app starts clean
