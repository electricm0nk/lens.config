# Sprint Plan — fourdogs-kaylee-data-access

**Feature:** fourdogs-kaylee-data-access  
**Service:** fourdogs/kaylee  
**Track:** express  
**Author:** Bob/SM  
**Date:** 2026-05-21

---

## Sprint Objective

Give Kaylee full read access to sales (per-customer, per-SKU, date-range), transactions
(full table), and inventory (EtailPet stock snapshot) — and extend order access to the
full create/submit/archive lifecycle. All new behavior is covered by TDD red-green tests
with BDD acceptance criteria.

---

## Implementation Slices

### Story DA-0: Runbook — Grant Verification Procedures

**Priority:** First. Article 11 (org constitution) requires a runbook for materially
changed infrastructure. Applying Postgres grants is a material operational change.

**Goal:** Create `docs/fourdogs/kaylee/fourdogs-kaylee-data-access/runbook.md` with
standby-ready operational procedures for the grant verification and application work.

**Tasks:**
1. Create `runbook.md` with:
   - Patroni access path: `kubectl exec -it postgresql-0 -n fourdogs -- psql -U postgres fourdogs_central`
   - Grant verification commands: `\dp sales transactions inventory items order_items orders`
   - Grant application SQL (matching tech-plan DB Grant Requirements)
   - Verification commands: confirm each tool that had `InsufficientPrivilegeError` now returns data
   - Last Verified timestamp field for post-apply sign-off
2. Commit to feature docs path

**Acceptance Criteria:**
- Given `runbook.md` exists at `docs/fourdogs/kaylee/fourdogs-kaylee-data-access/runbook.md`
- When any operator needs to re-apply grants after a Patroni failover
- Then the runbook provides complete self-contained instructions without consulting another source

**Dependencies:** None

---

### Story DA-1: DB Grant Verification and Application

**Priority:** Must-do first. Blocks all other data-tool stories.

**Goal:** Verify and apply all missing cross-DB grants for `kaylee_agent` role on
`fourdogs_central`.

**Tasks:**
1. Connect to `fourdogs_central` Patroni primary at `10.0.0.56:5432`
2. Run `\dp sales transactions inventory items order_items orders` to check current grants
3. Apply any missing grants:
   ```sql
   GRANT SELECT ON sales TO kaylee_agent;
   GRANT SELECT ON transactions TO kaylee_agent;
   GRANT SELECT ON inventory TO kaylee_agent;
   GRANT SELECT ON items TO kaylee_agent;
   GRANT SELECT ON order_items TO kaylee_agent;
   GRANT INSERT, UPDATE ON orders TO kaylee_agent;
   ```
4. Verify each existing tool that previously returned `InsufficientPrivilegeError` now works
5. Remove the stub error messages from `sales_summary`, `top_products`, `customer_lookup`,
   `inventory_velocity` after grants confirmed
6. Review and update test suites for `sales_summary`, `top_products`, `customer_lookup`,
   `inventory_velocity`, `order_view` to reflect post-grant behavior on the privilege-error
   code path (tests written against stubs must be updated to match the new live-query behavior)

**Acceptance Criteria:**
- Given the kaylee_agent role exists and grants are applied
- When any tool with `needs_central_pool=True` acquires a connection
- Then `InsufficientPrivilegeError` is not raised for any of the above tables

**Dependencies:** DA-0 (runbook exists), then none for infrastructure

---

### Story DA-2: `sales_query` Tool

**Goal:** New tool enabling per-customer, per-product, and date-range queries against
`fourdogs_central.sales`. Replaces the narrow current-month-only limitation of
`sales_summary`.

**Tasks:**
1. Write failing tests (`test_tool_sales_query.py`) for:
   - Customer name query → returns matching rows
   - Date range query → returns rows in range
   - UPC/product name query → returns matching rows
   - Category filter query → returns matching rows
   - Empty result → friendly message
   - No filter → defaults to last 30 days
   - `InsufficientPrivilegeError` → graceful error string
   - Natural-language date inputs: "last month", "March 12th", "last week", "yesterday"
   - Unparseable date → falls back to last 30 days
   - Result count == 100 → truncation notice appended
2. Implement `tools/sales_query.py`:
   - Add `dateparser` to `pyproject.toml` before implementation
   - Parameterized SELECT with optional customer, product/UPC, category, date filters
   - LIMIT 100; when `len(rows) == 100`, append truncation notice
   - Returns formatted text block per row
   - Appends `ToolEntry` to `TOOL_REGISTRY`
3. Import in `tools/__init__.py` (or wherever tools are imported for registration)
4. All tests pass

**BDD Acceptance Criteria:**
- **AC1:** Given "what did Suzie buy last month", When dispatched, Then results from `sales` for customer_name ILIKE '%Suzie%' within last calendar month
- **AC2:** Given "show me Fussie Cat Tuna sales", When dispatched, Then results filtered by product_name ILIKE '%Fussie Cat Tuna%'
- **AC3:** Given a date like "March 12th", When dispatched, Then results filtered by sale_date = '2026-03-12'
- **AC4:** Given no recognizable filter, When dispatched, Then returns last 30 days, max 100 rows
- **AC5:** Given "what pet food sold best", When dispatched, Then results filtered by category ILIKE '%pet food%'
- **AC6:** Given result count equals 100, When dispatched, Then response appends truncation notice

**Dependencies:** DA-1 (grants)

---

### Story DA-3: `transaction_query` Tool

**Goal:** New tool giving Kaylee full read access to `fourdogs_central.transactions`.

**Tasks:**
1. Write failing tests (`test_tool_transaction_query.py`) for:
   - Date range query → returns matching transactions
   - order_id lookup → returns that transaction
   - Empty result → friendly message
   - `InsufficientPrivilegeError` → graceful error string
2. Implement `tools/transaction_query.py`:
   - Parameterized SELECT with optional date range and order_id filter
   - LIMIT 100
   - Full row data: order_id, transaction_date, shop_name, net_sale, total, profit,
     margin, customer_name, employee_name
   - Appends `ToolEntry` to `TOOL_REGISTRY`
3. All tests pass

**BDD Acceptance Criteria:**
- **AC1:** Given "show me transactions last week", When dispatched, Then rows from `transactions` for last 7 days
- **AC2:** Given a specific order_id, When dispatched, Then that exact transaction row is returned
- **AC3:** Given no filter, When dispatched, Then last 30 days, max 100 rows
- **AC4:** Given result count equals 100, When dispatched, Then response appends truncation notice

**Dependencies:** DA-1 (grants)

---

### Story DA-4: `inventory_lookup` Tool

**Goal:** New tool giving Kaylee read access to `fourdogs_central.inventory` (EtailPet
stock snapshot: qty_on_hand, pricing, vendor, reorder_status). Distinct from
`inventory_velocity` which reads the `items` catalog.

**Tasks:**
1. Write failing tests (`test_tool_inventory_lookup.py`) for:
   - Product name fragment → matching rows
   - UPC exact match → matching row
   - Archived products excluded
   - Empty result → friendly message
   - `InsufficientPrivilegeError` → graceful error string
2. Implement `tools/inventory_lookup.py`:
   - Parameterized SELECT from `inventory WHERE archived = FALSE`
   - Returns qty_on_hand, price, vendor_cost, default_vendor, reorder_status, category_name
   - LIMIT 100
   - Appends `ToolEntry` to `TOOL_REGISTRY`
3. Partition trigger keywords to avoid collision with `inventory_velocity`
4. All tests pass

**BDD Acceptance Criteria:**
- **AC1:** Given "what's on hand for Blue Buffalo Chicken", When dispatched, Then inventory rows matching 'Blue Buffalo Chicken'
- **AC2:** Given a UPC, When dispatched, Then exact match from inventory
- **AC3:** Given archived=TRUE products, When queried, Then excluded from results
- **AC4:** Given result count equals 100, When dispatched, Then response appends truncation notice

**Dependencies:** DA-1 (grants)

---

### Story DA-8: `list_vendors` Tool

**Goal:** Return a numbered list of unique vendor names from `items.default_vendor`.
This is a prerequisite for `order_create` when the operator has not specified a vendor.
Order-creation flow: operator says "create an order" → Kaylee calls `list_vendors` →
presents numbered list → operator selects → Kaylee calls `order_create`.

**Tasks:**
1. Write failing tests (`test_tool_list_vendors.py`) for:
   - Returns numbered list of distinct vendors
   - Empty result (no items) → friendly message
   - Does NOT trigger on "what's on hand" or "show inventory"
2. Implement `tools/list_vendors.py`:
   - `SELECT DISTINCT default_vendor FROM items WHERE default_vendor IS NOT NULL ORDER BY default_vendor`
   - Returns formatted numbered list
   - Appends `ToolEntry` to `TOOL_REGISTRY` with trigger keywords: "list vendors", "what vendors", "which vendors", "available vendors"
3. All tests pass

**BDD Acceptance Criteria:**
- **AC1:** Given "list vendors" or "what vendors do we have", When dispatched, Then returns numbered list of `items.default_vendor` values
- **AC2:** Given no items in table, When dispatched, Then returns "No vendors found in catalog."
- **AC3:** Trigger does NOT fire for "what's on hand" or "show inventory"

**Dependencies:** DA-1 (grants)

---

### Story DA-5: `order_create` Tool

**Goal:** Allow Kaylee to create a new purchase order for the session's vendor.

**Tasks:**
1. Write failing tests (`test_tool_order_create.py`) for:
   - Creates order row when vendor_adapter_id in session_metadata
   - Returns order_id confirmation
   - Returns error when vendor_adapter_id missing from session_metadata
   - Does not trigger on generic message without "new order" / "create order"
2. **Pre-implementation gate (required before any code):** Agree on vendor-selection
   dispatch mechanism from ADR-6 options (a/b/c). Document the chosen mechanism in
   a code comment or PR description. Confirm that the agreed mechanism is tested by
   at least one integration-style test covering the multi-turn vendor selection flow.
3. Implement `tools/order_create.py`:
   - Reads `vendor_adapter_id` from `session_metadata` (required — error if missing)
   - Reads `created_by` via `needs_session_id=True` ToolEntry flag (dispatcher passes `session_id`
     explicitly; **do not use `session_metadata.get("session_id")` — `session_id` is not
     inside the metadata blob**)
   - Pre-implementation check: confirm dispatcher supports `needs_session_id` or add support
   - Parameterized INSERT INTO `orders`
   - Returns confirmation string with new `order_id`
   - Appends `ToolEntry` to `TOOL_REGISTRY`
4. All tests pass

**BDD Acceptance Criteria:**
- **AC1:** Given "start a new order", session_metadata has vendor_adapter_id, When dispatched, Then new orders row created and order_id returned
- **AC2:** Given session_metadata lacks vendor_adapter_id, When dispatched, Then descriptive error returned, no DB write
- **AC3:** Trigger does NOT fire for "update order" or "show order"

**Dependencies:** DA-1 (grants), DA-8 (list_vendors available as prerequisite)

---

### Story DA-6: `order_submit` and `order_archive` Tools

**Goal:** Allow Kaylee to submit (lock) and archive (close) an existing order.

**Tasks:**
1. Write failing tests for `order_submit`:
   - Sets `submitted = TRUE` when `order_id` in session_metadata
   - Returns confirmation
   - No-ops when `submitted` already TRUE
   - Returns error when `order_id` missing
2. Write failing tests for `order_archive`:
   - Sets `archived = TRUE` when `order_id` in session_metadata
   - Returns confirmation
   - Trigger does not fire without "archive" / "close order" keywords
3. Implement `tools/order_submit.py` and `tools/order_archive.py`
4. All tests pass

**BDD Acceptance Criteria:**
- **AC1:** Given "submit this order", order_id in session_metadata, When dispatched, Then `submitted = TRUE` in DB
- **AC2:** Given "archive the order", order_id in session_metadata, When dispatched, Then `archived = TRUE` in DB
- **AC3:** Already-submitted guard: given order already submitted, attempting `order_submit` again returns "Order already submitted"
- **AC4:** Already-archived guard: given order already archived, attempting `order_archive` again returns "Order is already archived."
- **AC5:** Trigger exclusion: given message "place an order for Nature's Recipe", When evaluated, Then `order_submit` does NOT fire ("place order" trigger must not match order-creation intent; add negative trigger test)

**Dependencies:** DA-1, DA-5

---

### Story DA-7: Architecture Doc Update and FD-1 Compliance

**Goal:** Update `docs/fourdogs/kaylee/architecture.md` to reflect new tool modules
and satisfy FD-1 (doc sync requirement on develop merges).

**Tasks:**
1. Add new tools section to architecture.md documenting all DA-2..DA-6 tools
2. Add table of DB grants and verification procedure
3. Add tool trigger keyword registry summary
4. Verify `docs/fourdogs/app-arch.md` and `docs/fourdogs/index.md` — add
   No-Doc-Sync-Required or update if applicable

**Dependencies:** DA-2 through DA-6 complete

---

## Sprint Dependency Map

```
DA-0 (runbook)
  → DA-1 (grants)
      → DA-2 (sales_query)
      → DA-3 (transaction_query)
      → DA-4 (inventory_lookup)
      → DA-8 (list_vendors)
      → DA-5 (order_create)  [also depends on DA-8]
          → DA-6 (order_submit + order_archive)
              → DA-7 (arch doc)
```

DA-2, DA-3, DA-4, DA-8 are all independently executable after DA-1.

---

## Unresolved Risks for FinalizePlan

| Risk | Owner | Mitigation |
|---|---|---|
| `vendor_adapter_id` may not be present in all sessions (set by Central UI at session creation) | Dev | DA-5 must guard against missing vendor_adapter_id; surface clear error |
| `session_metadata.created_by` field: check if Central UI sets this or if it needs to be wired | Dev | Check Central UI `POST /sessions` body before DA-5 implementation |
| Date parsing NLP for `sales_query`: "March 12th" → ISO date — needs a lightweight parser | Dev | Use `dateparser` or a simple regex for common patterns; document limitations |
| Grant application may require Patroni primary access via `kubectl exec` | Ops | Confirm access path before DA-1; document in runbook |

---

## Completion Criteria

- All 6 functional stories (DA-2 through DA-6) pass TDD red-green tests
- All BDD acceptance criteria have passing tests (Article 8)
- DB grants applied and confirmed (DA-1)
- `InsufficientPrivilegeError` stub messages retired
- Architecture doc updated (DA-7)
- FD-1 compliance confirmed in DA-7 PR
