# Business Plan — fourdogs-kaylee-data-access

**Feature:** fourdogs-kaylee-data-access  
**Service:** fourdogs/kaylee  
**Track:** express  
**Author:** John/PM  
**Date:** 2026-05-21

---

## Problem Statement

The Kaylee AI agent is in active use by the Four Dogs team for order management and
inventory assistance. Its data tools are partially implemented but limited in scope:

- `sales_summary` returns only current-month aggregate totals — it cannot answer
  questions about specific dates, specific customers, or specific products
- `transactions` table has no tool at all — Kaylee cannot answer anything about
  POS-level transaction data
- `inventory` table (the EtailPet stock snapshot: qty_on_hand, pricing, vendor,
  reorder status) has no tool — `inventory_velocity` uses the `items` catalog, not
  the real inventory snapshot
- Orders are limited to line-item quantity updates — Kaylee cannot create new
  orders, submit them to a vendor, or archive completed orders
- Cross-DB role grants for the `transactions` and `inventory` tables may not be
  applied yet — an explicit verification and grant application story is needed

This leaves Kaylee unable to answer the kinds of questions the team needs most:

> "Suzie came in and bought 12 cans of Fussie Cat Tuna on March 12th — she does
> this every three weeks. Can you look her up?"
>
> "What did we sell in March vs February?"
>
> "What's the current on-hand for Blue Buffalo Chicken?"
>
> "Can you start a new order for Nature's Recipe?"

---

## Target Users

**Primary:** Todd (store owner/operator) and floor-walk staff using Kaylee during
daily ordering, inventory checks, and customer lookups.

---

## Business Goals

1. Give Kaylee full read access to the `sales` table — including per-SKU,
   per-customer, and date-range breakdowns, not just monthly aggregates.
2. Give Kaylee full read access to the `transactions` table — no rollups; the
   model can see the complete POS order picture.
3. Give Kaylee full read access to the `inventory` table — the EtailPet stock
   snapshot with qty_on_hand, pricing, vendor, and reorder status.
4. Wire read access to both `items` (Four Dogs catalog) and `inventory` (EtailPet
   snapshot) so Kaylee can cross-reference catalog and real stock levels.
5. Give Kaylee full create/update/submit/archive capability for `orders`.
6. Provide interactive vendor selection when initiating orders — when an operator
   says "create an order" without specifying a vendor, Kaylee calls `list_vendors`
   and presents a numbered list of available vendors from the `items` catalog so
   the operator can pick by name rather than remembering the exact vendor string.
7. Verify and apply all cross-DB role grants needed for goals 1–6.
8. Retire the "stub return" error strings from existing tools when grants are
   confirmed applied.

---

## Scope

### In Scope

- New tool: `sales_query` — date-range sales, per-customer, per-SKU queries against
  `fourdogs_central.sales`. Replaces `sales_summary` (which narrows to current month
  only). The new tool must support open-ended natural-language queries: by customer
  name, by product name/UPC, by date or date range, by employee, by category.
- New tool: `transaction_query` — unrestricted read from `fourdogs_central.transactions`.
  Return full row data for any requested date range or order_id lookup.
- New tool: `inventory_lookup` — reads `fourdogs_central.inventory` for stock levels,
  pricing, vendor, and reorder status. Distinct from `inventory_velocity` (which
  reads `items` + `order_items`).
- Enhanced `items` access — confirm `items` catalog read is working and expose UPC,
  vendor, price fields alongside catalog data.
- Extend `order_update` to cover full order lifecycle:
  - `order_create`: create a new `orders` row for the session vendor
  - `order_submit`: set `submitted = TRUE`
  - `order_archive`: set `archived = TRUE`
- New tool: `list_vendors` — returns a numbered list of unique vendors from
  `items.default_vendor`. Called by Kaylee before `order_create` when no vendor is
  specified. Order-creation only; not triggered by inventory or catalog queries.
- DB grant verification and application for all tables touched by new tools.

### Out of Scope

- No changes to the `fourdogs_central` schema or migrations (all tables already exist)
- No changes to the terminus inference gateway
- No changes to the Kaylee Chat UI
- `customer` table does not exist — customer data lives in `sales.customer_name`;
  the `customer_lookup` tool already reads this correctly

---

## Dependencies

- `fourdogs_central` Patroni PostgreSQL at `10.0.0.56:5432` — accessible via
  `app.state.central_pool` (asyncpg)
- `kaylee_agent` DB role must have SELECT on: `sales`, `transactions`, `inventory`,
  `items`, `order_items`
- `kaylee_agent` DB role must have INSERT/UPDATE on: `orders`
- Vault secret `kaylee/fourdogs_central_url` supplies the connection string
- No external infrastructure changes required

---

## Risks

| Risk | Likelihood | Mitigation |
|---|---|---|
| DB grants missing for transactions/inventory | Medium | Sprint includes explicit grant verification + apply stories |
| LLM may not parse open-ended date queries reliably | Low | Tool accepts explicit ISO dates + natural range keywords; document in tool docstring |
| Sales table is large — unbounded queries | Medium | Tool enforces LIMIT 100 per call (ADR-3); date window default applied when no filter provided |
| order_create needs session vendor context | Medium | session_metadata already carries vendor_adapter_id from session setup |
| `orders.created_by` NOT NULL — `session_id` is not inside the metadata blob | High | Extend ToolEntry with `needs_session_id=True`; dispatcher passes `session_id` explicitly (one dispatcher change in DA-5) |

---

## Success Criteria

1. Kaylee can answer "What did Suzie buy last month?" by querying `sales` with customer name + date range.
2. Kaylee can answer "What's on hand for Blue Buffalo Chicken?" by querying `inventory`.
3. Kaylee can answer "Show me transactions from last Tuesday" by querying `transactions`.
4. Kaylee can create a new order, add items to it, submit it, and archive it — all through conversation.
5. When operator says "create an order" without naming a vendor, Kaylee responds with a numbered vendor list and waits for selection before inserting the order row.
6. All tools catch and surface `InsufficientPrivilegeError` gracefully — no raw DB exceptions reach the model.
7. Cross-DB grants verified applied; tool stub error strings retired.
8. All new tools registered in `TOOL_REGISTRY` and trigger correctly from natural-language messages.
9. All new behaviors covered by passing TDD red-green tests with BDD acceptance criteria.
