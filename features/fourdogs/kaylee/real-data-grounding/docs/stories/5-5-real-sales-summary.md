# Story 5.5: Real sales_summary — query fourdogs_central.sales

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
I want `sales_summary` to return real revenue totals and order counts from `fourdogs_central.sales`,
so that Kaylee gives Betsy an accurate business snapshot rather than the hardcoded stub of $142,500
in 87 orders.

---

## Context

`sales_summary.py` currently returns a hardcoded dict marked `"source": "stub"`. The real data lives
in `fourdogs_central.sales`. The tool needs to run an aggregation query via `central_pool`.

**`fourdogs_central.sales` schema (relevant columns):**

| Column | Type | Note |
|---|---|---|
| `net_total` | NUMERIC | revenue per line item |
| `order_id` | TEXT | can be DISTINCT-counted for order count |
| `sale_date` | DATE or TIMESTAMP | filter current month / date range |
| `quantity` | INTEGER | units sold |
| `product_name` | TEXT | |
| `brand` | TEXT | |
| `customer_name` | TEXT | |
| `customer_id` | TEXT | |

**Default scope:** current calendar month (from `date_trunc('month', NOW())` to now). The tool
accepts no parameters from the user message — scope is always "this month" unless the trigger
message includes a year (future story).

---

## Acceptance Criteria

**Given** `central_pool` is available and the grant is applied,
**When** Kaylee receives a message triggering `sales_summary`,
**Then** the query runs against `fourdogs_central.sales` and returns real aggregated revenue

**Given** the current month has sales rows,
**When** the summary runs,
**Then** the result contains `total_revenue` (NUMERIC), `order_count` (INT), `avg_order_value`
(NUMERIC), and `month` (YYYY-MM string) — and does NOT contain `"source": "stub"`

**Given** the current month has no sales rows yet (e.g., first day of month),
**When** the summary runs,
**Then** the result returns `{"total_revenue": 0.0, "order_count": 0, "avg_order_value": 0.0,
"month": "YYYY-MM", "note": "no sales recorded this month"}`

**Given** `central_pool` is `None`,
**When** the trigger fires,
**Then** the tool is skipped silently

---

## Technical Notes

### File to rewrite

**`src/kaylee_agent/tools/sales_summary.py`**

**Current stub (full replacement):**
```python
# STUB — replace entirely:
return {"total_revenue": 142500.00, "orders": 87, "avg_order_value": 1638.00, "source": "stub"}
```

**Real implementation:**
```python
import asyncpg
from datetime import datetime

async def get_sales_summary(pool: asyncpg.Pool) -> dict:
    sql = """
        SELECT
            COALESCE(SUM(net_total), 0)::float           AS total_revenue,
            COUNT(DISTINCT order_id)                      AS order_count,
            COALESCE(AVG(net_total), 0)::float            AS avg_order_value
        FROM sales
        WHERE sale_date >= date_trunc('month', NOW())
    """
    row = await pool.fetchrow(sql)
    month_str = datetime.now().strftime("%Y-%m")
    if row["order_count"] == 0:
        return {
            "total_revenue": 0.0,
            "order_count": 0,
            "avg_order_value": 0.0,
            "month": month_str,
            "note": "no sales recorded this month",
        }
    return {
        "total_revenue": round(float(row["total_revenue"]), 2),
        "order_count": int(row["order_count"]),
        "avg_order_value": round(float(row["avg_order_value"]), 2),
        "month": month_str,
    }
```

**Updated `ToolEntry` registration:**
```python
SALES_SUMMARY_ENTRY = ToolEntry(
    name="sales_summary",
    trigger_fn=lambda msg: any(w in msg.lower() for w in TRIGGERS),
    tool_fn=get_sales_summary,
    needs_central_pool=True,
)
```

### Note on avg_order_value

The `AVG(net_total)` here averages over all sales line items, not unique orders. If we have multiple
line items per order (e.g., multi-item POS transactions), this will be lower than a true "average
order total". That's acceptable for now — a follow-up story can calculate per-order subtotals if needed.
Document this in a code comment.

---

## Definition of Done

- [ ] Stub replaced with real asyncpg query against `fourdogs_central.sales`
- [ ] Zero-result case handled cleanly (no NoneType errors, returns meaningful note)
- [ ] `"source": "stub"` key removed from result
- [ ] `SALES_SUMMARY_ENTRY` uses `needs_central_pool=True`
- [ ] Unit test: `test_sales_summary_queries_central_pool` added
- [ ] Integration test in dev: trigger "how are sales this month" message; confirm non-stub response
- [ ] CI green
