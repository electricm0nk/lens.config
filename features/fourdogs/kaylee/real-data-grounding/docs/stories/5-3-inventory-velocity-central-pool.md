# Story 5.3: Fix inventory_velocity — real cross-DB query via central_pool

**Feature:** real-data-grounding  
**Epic:** 5 — Kaylee Real Data & Context Grounding  
**Repo:** fourdogs-kaylee-agent  
**Status:** backlog  
**Priority:** high  
**Estimate:** S  
**Depends on:** Story 5.1 (central pool), Story 5.2 (Postgres grant)

---

## User Story

As the AI agent,
I want `inventory_velocity` to query real `items` and `order_items` data from `fourdogs_central`,
so that when Betsy asks "what's running low?" or "what are my high-risk items?", Kaylee answers with
real inventory numbers instead of silently returning nothing.

---

## Context

`inventory_velocity.py` currently queries `items` and `order_items` against the `fourdogs_kaylee` pool
(its own DB). These tables do not exist in `fourdogs_kaylee` — they live in `fourdogs_central`. The
query fails with `relation "items" does not exist` and is silently skipped by the dispatcher (by design
after the dispatcher bug fix).

The column aliases in the query ARE correct for the `fourdogs_central.items` schema. Only the pool
needs to change. The `ToolEntry` registration must change from `needs_pool=True` to
`needs_central_pool=True`.

**Current `items` column aliases (from DB schema):**

| Alias | Column | Meaning |
|---|---|---|
| `i` | `id` | item_id (PK) |
| `n` | `name` | product name |
| `b` | `brand` | brand |
| `q` | `quantity` / `qty` | current stock |
| `v` | `velocity` | weekly velocity |
| `va` | `velocity_all` | overall velocity |
| `vc` | `velocity_class` | HIGH/MED/LOW |
| `dos_days` | `dos_days` | days of stock |
| `risk_score` | `risk_score` | 0.0–1.0 |

**`order_items` relevant columns:** `item_id TEXT`, `item_name`, `velocity_tier`, `must_have`, `final_qty`

---

## Acceptance Criteria

**Given** `central_pool` is available (Story 5.1 deployed, Story 5.2 grant applied),
**When** Kaylee receives a message containing the `inventory_velocity` trigger word,
**Then** `inventory_velocity` executes its SQL against `fourdogs_central` via `central_pool` and returns
real row data (not empty, not an exception trace)

**Given** the query returns items,
**When** `dispatch_tools` injects the result,
**Then** the injected context contains `name`, `velocity_class`, `dos_days`, and `risk_score` fields

**Given** `central_pool` is `None` (Story 5.1 not yet deployed),
**When** the dispatcher evaluates `inventory_velocity`,
**Then** the tool is skipped silently — no error text injected into context

**Given** the grant has not been applied (Story 5.2 not done),
**When** the tool runs,
**Then** asyncpg raises a `PermissionError`; the dispatcher catches it and skips silently

---

## Technical Notes

### File to change

**`src/kaylee_agent/tools/inventory_velocity.py`**

Change only the tool registration:

```python
# Before:
INVENTORY_VELOCITY_ENTRY = ToolEntry(
    name="inventory_velocity",
    trigger_fn=lambda msg: any(w in msg.lower() for w in TRIGGERS),
    tool_fn=get_inventory_velocity,
    needs_pool=True,       # ← wrong pool
)

# After:
INVENTORY_VELOCITY_ENTRY = ToolEntry(
    name="inventory_velocity",
    trigger_fn=lambda msg: any(w in msg.lower() for w in TRIGGERS),
    tool_fn=get_inventory_velocity,
    needs_central_pool=True,  # ← correct pool (fourdogs_central)
)
```

The function signature `get_inventory_velocity(pool)` does not change — it still accepts a pool
argument. The dispatcher now passes `central_pool` instead of `pool`.

No SQL query changes needed. The existing query is already correct for `fourdogs_central.items` and
`fourdogs_central.order_items` — it just needs the right connection.

### Verification

```bash
# SSH to control plane, then:
kubectl -n fourdogs-kaylee-dev exec deploy/kaylee-agent -- \
  python -c "
import asyncio
import asyncpg
import os
async def test():
    pool = await asyncpg.create_pool(os.environ['CENTRAL_DATABASE_URL'])
    rows = await pool.fetch('SELECT i, n, vc, dos_days, risk_score FROM items ORDER BY risk_score DESC LIMIT 5')
    print(rows)
    await pool.close()
asyncio.run(test())
"
```

---

## Definition of Done

- [ ] `INVENTORY_VELOCITY_ENTRY` changed from `needs_pool=True` to `needs_central_pool=True`
- [ ] Unit test: `test_inventory_velocity_uses_central_pool` added (confirm called with central_pool)
- [ ] Integration test in dev: trigger inventory question to Kaylee; confirm real data in response
- [ ] CI green
