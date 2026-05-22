# Tech Plan — fourdogs-kaylee-data-access

**Feature:** fourdogs-kaylee-data-access  
**Service:** fourdogs/kaylee  
**Track:** express  
**Author:** Winston/Architect  
**Date:** 2026-05-21

---

## Architecture Context

Kaylee is a FastAPI service (`fourdogs-kaylee-agent`) using:
- `asyncpg` connection pools for Postgres access
- `app.state.central_pool` for `fourdogs_central` (Patroni at `10.0.0.56:5432`)
- A `TOOL_REGISTRY` pattern — tools self-register by appending a `ToolEntry` to
  the list on module import; the dispatcher evaluates all registered triggers on
  every chat message
- `session_metadata`: a free-form JSON dict stored per session; carries `order_id`,
  `submitted`, `vendor_adapter_id`, and other fields set at session creation

The existing `dispatcher.py` `ToolEntry` supports:
- `needs_central_pool` — pool passed as first arg
- `needs_session_metadata` — injected as kwarg
- `needs_user_message` — injected as kwarg
- `needs_pool` — kaylee-local DB (not central)

All new tools follow this pattern with no dispatcher changes required.

---

## Architecture Decision Record

### ADR-1: Extend TOOL_REGISTRY pattern, no new dispatch infrastructure

New tools are added as modules under `src/kaylee_agent/tools/`. The dispatcher
requires no changes. Each module appends a `ToolEntry` at import time.

**Rationale:** The existing pattern is proven, testable, and self-documenting.
Adding new dispatch modes (e.g., paginated results) is premature for this sprint.

### ADR-2: `sales_query` replaces `sales_summary` for general-purpose sales access

`sales_summary` is not deprecated — it remains for current-month aggregate
queries. A new `sales_query` tool is added alongside it for flexible date-range
and per-entity queries. Trigger keywords are partitioned to avoid double-firing.

**Rationale:** Existing callers (tests, integration) depend on `sales_summary`
returning the current-month aggregate. Replacing it inline would break those tests.
Adding a parallel `sales_query` tool avoids disrupting existing behavior while
unlocking the full per-customer/per-SKU use cases.

### ADR-3: Tool results are bounded — LIMIT enforced; truncation signaled

All new SELECT queries must include a LIMIT clause (default 100 rows). When
`len(rows) == LIMIT`, the tool must append to its response:
> "Showing first 100 results. Narrow the filter (date range, customer, or product) to see fewer results."

This applies to `sales_query`, `transaction_query`, and `inventory_lookup`. The
`sales` table is large; unbounded reads would inject enormous context into the LLM prompt.

### ADR-4: `order_create`, `order_submit`, `order_archive` are separate tools

Rather than extending `order_update` further, three focused tools are added:
- `order_create`: INSERT into `orders`, requires `vendor_adapter_id` from session metadata
- `order_submit`: UPDATE `orders SET submitted = TRUE`
- `order_archive`: UPDATE `orders SET archived = TRUE`

**Rationale:** Single-responsibility keeps triggers clean, tests minimal,
and failure modes isolated.

### ADR-5: `inventory_lookup` reads the `inventory` table; `inventory_velocity` unchanged

`inventory_velocity` reads `items` + `order_items` (demand scoring) and is not
changed. New `inventory_lookup` reads the `inventory` table (EtailPet snapshot:
qty_on_hand, pricing, vendor, reorder_status). Trigger keywords are partitioned.

### ADR-6: `list_vendors` is a prerequisite tool for order creation

When `order_create` is triggered without a vendor in session_metadata, the system must
present the vendor list before inserting an order. `list_vendors` reads
`DISTINCT default_vendor FROM items ORDER BY default_vendor` and returns a formatted
numbered list. The order-creation flow is:
1. Operator: "create an order"
2. Kaylee: calls `list_vendors` → presents numbered vendor list
3. Operator: selects a vendor
4. Kaylee: calls `order_create` with the selected vendor

**Dispatch mechanism (choose before DA-5 implementation begins):** The keyword-based
`TOOL_REGISTRY` dispatcher fires tools on message keywords, not by LLM instruction.
Three implementation paths are viable — choose one and document the decision in DA-5:
- **(a) Inline delegation:** `order_create` checks `vendor_adapter_id` in session_metadata;
  if absent, calls the `list_vendors` query inline and returns the list, deferring the INSERT.
- **(b) System prompt guidance:** System prompt instructs the model to present the vendor
  list proactively before order creation; user explicitly says "list vendors".
- **(c) Conditional trigger:** `order_create` trigger logic fires only when
  `vendor_adapter_id` is set; otherwise returns "Say 'list vendors' to pick a vendor first."

Path (a) is the recommended default — it keeps the interaction within a single tool call
and avoids multi-turn coordination requirements. The chosen path must be documented in
DA-5 before any implementation begins.

**Rationale:** Prevents the LLM from guessing vendor name strings that may not
match `items.default_vendor` exactly. Keeps `order_create` safe from NULL
`vendor_adapter_id` on ambiguous input.

---

## New Modules and Responsibilities

| Module | Tables | Operation | Trigger keywords |
|---|---|---|---|
| `tools/sales_query.py` | `sales` | SELECT — date range, customer, UPC, category, employee | "sales", "sold", "bought", "purchase", "customer sales", "product sales" (excludes "summary" which stays with `sales_summary`) |
| `tools/transaction_query.py` | `transactions` | SELECT — date range, order_id lookup | "transaction", "receipt", "tender", "payment" |
| `tools/inventory_lookup.py` | `inventory` | SELECT — qty_on_hand, pricing, vendor, reorder | "on hand", "stock level", "price", "vendor", "reorder", "inventory check" |
| `tools/list_vendors.py` | `items` | SELECT DISTINCT default_vendor — vendor selection list for order creation | "list vendors", "what vendors", "which vendors", "available vendors" |
| `tools/order_create.py` | `orders` | INSERT — new order for session vendor | "new order", "start order", "create order" |
| `tools/order_submit.py` | `orders` | UPDATE submitted=TRUE | "submit order", "send order", "place order" |
| `tools/order_archive.py` | `orders` | UPDATE archived=TRUE | "archive order", "close order", "finish order" |

---

## Data Contract

### `sales_query` inputs

The tool accepts a `user_message` string and parses intent:
- `customer_name`: extracted fragment for ILIKE `$1`
- `date_from` / `date_to`: ISO date strings or natural keywords ("last month", "this week", "March")
- `upc_fragment`: product name or UPC ILIKE fragment
- When no filters are extractable: last 30 days, all customers, no product filter

Query signature (parameterized):
```sql
SELECT order_id, sale_date, customer_name, product_name, item_upc,
       quantity, net_total, category, brand, employee_name
FROM sales
WHERE sale_date >= $1
  AND sale_date <= $2
  AND ($3::text IS NULL OR customer_name ILIKE $3)
  AND ($4::text IS NULL OR product_name ILIKE $4 OR item_upc ILIKE $4)
  AND ($5::text IS NULL OR category ILIKE $5)
ORDER BY sale_date DESC, order_id
LIMIT 100
```

Date parsing: `dateparser` library (add to `pyproject.toml`). Supported patterns tested:
"last month", "March 12th", "last week", "yesterday", "this month".
Fallback when unparseable: last 30 days. Tool docstring must enumerate supported patterns.

### `transaction_query` inputs

Accepts `user_message` → extracts date range or order_id.
Default: last 30 days, 100 rows.

```sql
SELECT order_id, transaction_date, shop_name, net_sale, total, profit,
       margin, customer_name, employee_name, cart_source
FROM transactions
WHERE ($1::timestamptz IS NULL OR transaction_date >= $1)
  AND ($2::timestamptz IS NULL OR transaction_date <= $2)
  AND ($3::text IS NULL OR order_id = $3)
ORDER BY transaction_date DESC
LIMIT 100
```

### `inventory_lookup` inputs

Accepts `user_message` → extracts product name fragment or UPC.

```sql
SELECT product_name, item_upc, qty_on_hand, price, vendor_cost,
       default_vendor, reorder_status, category_name, brand_name
FROM inventory
WHERE archived = FALSE
  AND ($1::text IS NULL OR product_name ILIKE $1 OR item_upc = $1)
ORDER BY product_name
LIMIT 100
```

### `list_vendors` query

```sql
SELECT DISTINCT default_vendor
FROM items
WHERE default_vendor IS NOT NULL
ORDER BY default_vendor
```

Returns a numbered list formatted for the LLM:
```
1. Blue Buffalo
2. Hills Science Diet
3. Nature's Recipe
...
```

### `order_create` inputs

Reads `vendor_adapter_id` from `session_metadata`. Requires `created_by` — sourced
via `needs_session_id=True` on the `ToolEntry`; the dispatcher passes `session_id`
as an explicit kwarg to the tool function. **Do not use `session_metadata.get("session_id")`
— `session_id` is a separate column, not a key inside the metadata blob.**

```sql
INSERT INTO orders (vendor_adapter_id, created_by, order_date)
VALUES ($1, $2, CURRENT_DATE)
RETURNING id
```

`order_submit` / `order_archive`: UPDATE by `order_id` from session_metadata.

---

## DB Grant Requirements

New grants required on `fourdogs_central` for the `kaylee_agent` role:

```sql
GRANT SELECT ON sales TO kaylee_agent;          -- may already exist
GRANT SELECT ON transactions TO kaylee_agent;
GRANT SELECT ON inventory TO kaylee_agent;
GRANT SELECT ON items TO kaylee_agent;           -- may already exist
GRANT SELECT ON order_items TO kaylee_agent;     -- may already exist
GRANT INSERT, UPDATE ON orders TO kaylee_agent;  -- for create/submit/archive
GRANT USAGE ON SEQUENCE orders_id_seq TO kaylee_agent;  -- not needed (UUID PK)
```

A sprint story must:
1. Verify current grants with `\dp` in psql
2. Apply any missing grants
3. Confirm all existing `InsufficientPrivilegeError` stub messages are retired
   after grants are applied

---

## Testing Strategy

All new tools follow the existing test pattern (see `tests/test_tool_sales_summary.py`):
- Unit tests use `asyncpg` pool mocks (MagicMock or `AsyncMock`)
- TDD red-green: failing test first, then implementation
- BDD acceptance scenarios mapped to test functions per story

### Article 7 (TDD) compliance plan

Each story PR must demonstrate the red-green commit sequence:
1. Commit: failing test covering new behavior
2. Commit: minimal implementation making test pass
3. Commit: refactor (if any) with tests green

### Article 8 (BDD) acceptance criteria

Each story provides Given/When/Then scenarios; each AC has a corresponding
passing pytest test.

### Monetary and numeric formatting (MED-3)

For `transaction_query` output:
- Monetary fields (`net_sale`, `total`, `profit`, `vendor_cost`): `${value:,.2f}`
- Percentage fields (`margin`): `{value:.1f}%`

Matches the existing `sales_summary` formatting pattern.

### Multi-tool context window — known risk (LOW-3)

The dispatcher fires all matching tools on a single message. A query that triggers
`sales_query` + `inventory_lookup` simultaneously injects up to 200 rows of data
into the LLM context. For long sessions, this can approach the model context limit.

**Accepted risk:** No mitigation in this sprint. Operators are advised to use specific
queries over broad queries. Monitor in production; open a follow-on story for
per-tool result size caps if context overflow becomes an operational issue.

### Article 9 (Security) considerations

- No new credentials introduced; all tools use existing `central_pool` (Vault-backed `CENTRAL_DATABASE_URL`)
- No user-supplied data written to DB except through parameterized queries (`$1`, `$2`, etc.)
- `order_create` uses `vendor_adapter_id` from server-side session metadata, not user message input — prevents injection
- All queries are parameterized; no string interpolation into SQL

### Article 15 (AI Safety) data-handling

- Tools inject structured text into LLM context; no PII is persisted by Kaylee
- Customer names from `sales` appear in tool context only for the duration of the request
- No data leaves the local network; all inference is via `terminus-inference-gateway` on internal network

### FD-2 (Loki Logging)

No new deployable workload is introduced. Existing Kaylee logging already emits JSON to stdout. No new startup events required.

### FD-3 (OpenAPI)

No new HTTP routes are introduced in this feature. All changes are internal tool logic.

---

## Architecture Documentation Note (Article 3)

The existing `docs/fourdogs/kaylee/architecture.md` documents the Kaylee agent
architecture. This feature adds new tools but does not change the service topology
or introduce new deployable workloads. The architecture doc must be updated in the
implementation sprint to reflect:
- New tool modules and their table dependencies
- DB grant requirements and verification procedure
- The `inventory_lookup` tool distinction from `inventory_velocity`

---

## Implementation Boundaries

- All code changes in `TargetProjects/fourdogs/kaylee-agent/fourdogs-kaylee-agent/`
- No changes to `fourdogs_central` application code or migrations
- No changes to Helm chart or Kubernetes resources (existing `central_pool` env vars already wired)
- DB grants are applied via `psql` in a sprint verification story — not automated migrations
