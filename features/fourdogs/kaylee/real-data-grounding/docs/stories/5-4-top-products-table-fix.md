# Story 5.4: Fix top_products — correct table reference + central_pool

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
I want `top_products` to query the correct `items` table with the correct column aliases from
`fourdogs_central`,
so that Kaylee can tell Betsy which items are actually selling well rather than returning nothing.

---

## Context

`top_products.py` has two bugs beyond using the wrong pool:

1. **Wrong table name:** queries `fourdogs_central.products` — this is invalid PostgreSQL syntax (no
   cross-DB dot notation) AND the table is named `items`, not `products`
2. **Wrong column aliases:** references `p.product_id` and `p.name` (non-existent) — the `items` table
   uses short aliases `i.i` (id) and `i.n` (name)
3. **Wrong pool:** `needs_pool=True` — should be `needs_central_pool=True`

The fix requires rewriting the query SQL and updating the pool reference. No schema migration needed —
the correct data is already in `fourdogs_central.items` and `fourdogs_central.order_items`.

**`fourdogs_central.items` schema (relevant columns):**

| Short alias | Column | Type |
|---|---|---|
| `i` | `id` | TEXT (PK) |
| `n` | `name` | TEXT |
| `b` | `brand` | TEXT |
| `q` | `quantity`/`qty` | INTEGER |
| `vc` | `velocity_class` | TEXT (HIGH/MED/LOW) |

**`fourdogs_central.order_items` schema (relevant columns):**

| Column | Type | Note |
|---|---|---|
| `item_id` | TEXT | FK to items.i |
| `item_name` | TEXT | |
| `final_qty` | INTEGER | quantity actually ordered |
| `velocity_tier` | TEXT | |
| `must_have` | BOOLEAN | |

---

## Acceptance Criteria

**Given** `central_pool` is available and the grant is applied,
**When** Kaylee receives a message triggering `top_products`,
**Then** the query executes without error and returns ranked product data from real `items` and
`order_items` rows

**Given** the query returns results,
**When** the context is injected,
**Then** the result contains `item_id`, `name`, `brand`, `order_count` or `total_ordered` fields,
consistent with what the system prompt can interpret

**Given** `central_pool` is `None`,
**When** the trigger fires,
**Then** the tool is skipped silently

---

## Technical Notes

### File to rewrite

**`src/kaylee_agent/tools/top_products.py`**

**Current broken SQL (paraphrased):**
```sql
SELECT p.product_id, p.name, SUM(oi.quantity) as total_ordered
FROM fourdogs_central.products p                     -- WRONG: dot notation invalid, wrong table name
JOIN fourdogs_central.order_items oi ON oi.product_id = p.product_id  -- WRONG column
GROUP BY p.product_id, p.name
ORDER BY total_ordered DESC
LIMIT 10
```

**Corrected SQL:**
```sql
SELECT
    i.i         AS item_id,
    i.n         AS name,
    i.b         AS brand,
    i.vc        AS velocity_class,
    COUNT(oi.item_id)   AS appearance_count,
    SUM(oi.final_qty)   AS total_ordered
FROM items i
LEFT JOIN order_items oi ON oi.item_id = i.i
GROUP BY i.i, i.n, i.b, i.vc
ORDER BY total_ordered DESC NULLS LAST
LIMIT 10
```

Note: No schema prefix needed — `central_pool` connects directly to `fourdogs_central` where these
tables are in the `public` schema.

**Updated `ToolEntry` registration:**
```python
TOP_PRODUCTS_ENTRY = ToolEntry(
    name="top_products",
    trigger_fn=lambda msg: any(w in msg.lower() for w in TRIGGERS),
    tool_fn=get_top_products,
    needs_central_pool=True,   # changed from needs_pool=True
)
```

Function signature stays `get_top_products(pool)` — dispatcher passes `central_pool`.

### Edge case

If `order_items` has no rows yet for a given item, `SUM(oi.final_qty)` returns `NULL`. The
`ORDER BY ... NULLS LAST` clause ensures unordered items sort to the bottom. Return result as-is —
Kaylee can interpret sparse data correctly.

---

## Definition of Done

- [ ] SQL rewritten to use `items` (not `products`), correct column aliases, no schema prefix
- [ ] `TOP_PRODUCTS_ENTRY` changed to `needs_central_pool=True`
- [ ] Unit test: `test_top_products_uses_central_pool` added
- [ ] Integration test in dev: trigger "what's selling well" message; confirm real item names in response
- [ ] CI green
