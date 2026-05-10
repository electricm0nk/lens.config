# Story 6.2 — `order_view` Tool — Read Current Order Items (TDD)

**Epic:** 6 — Order management tools  
**Track:** hotfix-express  
**Feature:** fourdogs-kaylee-order-update-bug  
**Depends on:** Story 6.1

---

## Context

Betsy frequently asks "what's on the order?" before making adjustments. Currently Kaylee has no tool to read the live `order_items` state — she can only see the count from the `[ORDER CONTEXT]` block. This tool closes that gap and also gives the LLM ground truth to reference when confirming updates.

## Acceptance Criteria

**Given** a user message containing "show me the order" or "what's in the order" or "view the order"  
**And** `session_metadata` contains a valid `order_id`  
**When** `dispatch_tools` runs  
**Then** `[TOOL CONTEXT: order_view]` is injected with a formatted table of `item_name`, `final_qty`, `ghost_qty`, `must_have`, `current_stock_qty`

**Given** a user message containing order view keywords  
**But** `session_metadata` has no `order_id`  
**When** `dispatch_tools` runs  
**Then** the tool returns empty string (no injection)

**Given** the order has zero items  
**When** `order_view` fires  
**Then** the injected context says "No items on this order yet."

**Given** a non-view message ("add 3 cases of Blue Buffalo")  
**When** `dispatch_tools` runs  
**Then** `order_view` does NOT trigger

---

## TDD Plan (Article 7)

**Red phase — write and commit these failing tests first:**

`tests/tools/test_order_view.py`

```python
# Test 1: trigger returns True for "show me the order"
# Test 2: trigger returns True for "what's in the order"
# Test 3: trigger returns True for "current order"
# Test 4: trigger returns False for "add 3 cases"
# Test 5: trigger returns False for "how are sales looking"
# Test 6: view_order() with mock pool + order_id returns formatted table
# Test 7: view_order() with no order_id in session_metadata returns ""
# Test 8: view_order() with empty order_items returns "No items on this order yet."
# Test 9: full dispatch_tools integration — view tool fires, context injected
```

Use `asyncpg` mock pattern from existing tool tests (`unittest.mock.AsyncMock`).

**Green phase — implement:**

`src/kaylee_agent/tools/order_view.py`:
```python
_TRIGGER_KEYWORDS = frozenset([
    "show.*order", "what.*in.*order", "current order", "order list",
    "view.*order", "whats.*on.*order", "what.*on.*order",
])

def _trigger(user_message: str) -> bool:
    msg = user_message.lower()
    return any(re.search(kw, msg) for kw in _TRIGGER_KEYWORDS)

async def view_order(pool: Any, session_metadata: dict | None = None) -> str:
    ...  # query order_items, format table
```

Register in `TOOL_REGISTRY` with `needs_central_pool=True, needs_session_metadata=True`.

**SQL:**
```sql
SELECT item_name, final_qty, ghost_qty, current_stock_qty, velocity_tier, must_have
FROM order_items
WHERE order_id = $1
ORDER BY must_have DESC, item_name ASC
```

---

## Output Format

```
[TOOL CONTEXT: order_view]
Current order items (draft):

| Item | Qty | Suggested | Stock | Velocity | Must Have |
|------|-----|-----------|-------|----------|-----------|
| Blue Buffalo Large Breed 30lb | 5 | 4 | 2 | high | yes |
| Orijen Cat Food 5lb | 0 | 2 | 1 | medium | no |
```

---

## Tasks

- [ ] Write failing tests (`test_order_view.py`) — commit as RED
- [ ] Implement `src/kaylee_agent/tools/order_view.py`
- [ ] Register in `TOOL_REGISTRY` with `needs_central_pool=True, needs_session_metadata=True`
- [ ] Run all tests — confirm RED → GREEN
- [ ] Commit as GREEN
