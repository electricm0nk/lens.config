# Story 5.6: Real customer_lookup — query fourdogs_central.sales

**Feature:** real-data-grounding  
**Epic:** 5 — Kaylee Real Data & Context Grounding  
**Repo:** fourdogs-kaylee-agent  
**Status:** backlog  
**Priority:** medium  
**Estimate:** S  
**Depends on:** Story 5.1 (central pool), Story 5.2 (Postgres grant)

---

## User Story

As the AI agent,
I want `customer_lookup` to search real customer records in `fourdogs_central.sales`,
so that Kaylee can tell Betsy accurate purchase history for a customer she asks about, rather than
returning a fake "Stub Customer" with hardcoded account data.

---

## Context

`customer_lookup.py` returns a hardcoded stub dict. There is no `customers` table in `fourdogs_central`.
All customer data lives denormalized in `fourdogs_central.sales` via `customer_name` (TEXT) and
`customer_id` (TEXT) columns.

The lookup should support:
- Search by name (fuzzy `ILIKE` on `customer_name`)
- Return: last purchase date, purchase count (distinct order_ids), total spend

Since there's no `customers` table, the tool derives customer identity from the `sales` table. This
is intentionally minimal — a proper customer registry can be added later if needed.

**`fourdogs_central.sales` schema (relevant columns):**

| Column | Type | Note |
|---|---|---|
| `customer_name` | TEXT | matches via ILIKE |
| `customer_id` | TEXT | identifier if known |
| `net_total` | NUMERIC | spend per line item |
| `order_id` | TEXT | count DISTINCT for purchase count |
| `sale_date` | DATE or TIMESTAMP | for last_purchase |

---

## Acceptance Criteria

**Given** a customer name that exists in `sales`,
**When** `customer_lookup` is triggered with a matching message,
**Then** the result contains `customer_name`, `total_spend`, `purchase_count`, and `last_purchase` drawn from real data

**Given** a customer name with no matching rows in `sales`,
**When** `customer_lookup` runs,
**Then** the result contains `{"found": false, "query": "<name>", "note": "no purchase history found"}`

**Given** `customer_id` is available in a sales row alongside `customer_name`,
**When** multiple customer_id values appear for the same customer name,
**Then** the most recently appearing `customer_id` is included in the result

**Given** the message does not contain a recognizable customer name,
**When** the trigger fires,
**Then** the tool returns a result indicating `"no customer name found in query"` rather than
throwing an exception

**Given** `central_pool` is `None`,
**When** the trigger fires,
**Then** the tool is skipped silently

---

## Technical Notes

### File to rewrite

**`src/kaylee_agent/tools/customer_lookup.py`**

**Name extraction:**  
The trigger detection needs to extract a customer name from the user message. A simple heuristic:
capture the text after "customer" / "for" / "look up" / "lookup" in the message. Return early if
no candidate name found (min 2 chars after token).

**Real implementation:**
```python
import asyncpg
import re

def _extract_customer_name(msg: str) -> str | None:
    """Extract a customer name candidate from the user message."""
    # Patterns: "look up <name>", "for customer <name>", "customer named <name>", "about <name>"
    patterns = [
        r"look\s+up\s+(.+)",
        r"customer\s+(?:named\s+)?(.+)",
        r"for\s+(.+)",
        r"about\s+(.+)",
    ]
    msg_lower = msg.lower()
    for pattern in patterns:
        m = re.search(pattern, msg_lower)
        if m:
            candidate = m.group(1).strip().rstrip("?.,!")
            if len(candidate) >= 2:
                return candidate
    return None


async def get_customer_lookup(pool: asyncpg.Pool, user_message: str = "") -> dict:
    name_query = _extract_customer_name(user_message)
    if not name_query:
        return {"found": False, "note": "no customer name found in query"}

    sql = """
        SELECT
            customer_name,
            customer_id,
            COUNT(DISTINCT order_id)                    AS purchase_count,
            COALESCE(SUM(net_total), 0)::float          AS total_spend,
            MAX(sale_date)                              AS last_purchase
        FROM sales
        WHERE customer_name ILIKE $1
        GROUP BY customer_name, customer_id
        ORDER BY last_purchase DESC
        LIMIT 1
    """
    row = await pool.fetchrow(sql, f"%{name_query}%")
    if row is None:
        return {"found": False, "query": name_query, "note": "no purchase history found"}

    return {
        "found": True,
        "customer_name": row["customer_name"],
        "customer_id": row["customer_id"],
        "purchase_count": int(row["purchase_count"]),
        "total_spend": round(float(row["total_spend"]), 2),
        "last_purchase": str(row["last_purchase"]),
    }
```

**ToolEntry registration:**
```python
CUSTOMER_LOOKUP_ENTRY = ToolEntry(
    name="customer_lookup",
    trigger_fn=lambda msg: any(w in msg.lower() for w in TRIGGERS),
    tool_fn=get_customer_lookup,      # signature: (pool, user_message)
    needs_central_pool=True,
    passes_user_message=True,         # new flag — dispatcher passes the user message string
)
```

**Dispatcher change needed for `passes_user_message`:**
The dispatcher must pass `user_message` as a kwarg when `entry.passes_user_message=True`:
```python
if entry.passes_user_message:
    raw = await entry.tool_fn(selected_pool, user_message=user_message)
else:
    raw = await entry.tool_fn(selected_pool)
```

Add `passes_user_message: bool = False` to `ToolEntry` dataclass.

### Limitation note

ILIKE on `customer_name` is a best-effort match. If `fourdogs_central.sales` stores full names
like "Betsy Reynolds" and the query is "Reynolds", it will still find a match. For short or common
names there may be false positives. This is acceptable MVP behaviour.

---

## Definition of Done

- [ ] Stub replaced with real asyncpg query against `fourdogs_central.sales`
- [ ] `_extract_customer_name()` helper implemented and tested
- [ ] Not-found case handled cleanly (no exception, returns `{"found": false, ...}`)
- [ ] `passes_user_message: bool = False` added to `ToolEntry`; dispatcher updated
- [ ] `CUSTOMER_LOOKUP_ENTRY` uses `needs_central_pool=True` and `passes_user_message=True`
- [ ] Unit tests: name extraction, found case, not-found case
- [ ] Integration test in dev: "look up customer [real name]"; confirm real data in response
- [ ] CI green
