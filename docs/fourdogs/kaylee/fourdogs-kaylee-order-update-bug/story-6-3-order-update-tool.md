# Story 6.3 — `order_update` Tool — Set final_qty with Guards (TDD)

**Epic:** 6 — Order management tools  
**Track:** hotfix-express  
**Feature:** fourdogs-kaylee-order-update-bug  
**Depends on:** Story 6.1

---

## Context

This is the core fix for the bug. When Betsy asks Kaylee to adjust quantities, the tool must actually write to `order_items.final_qty` in the fourdogs_central DB, and inject a result so the LLM reports what actually happened — not what it imagined.

## Acceptance Criteria

**Given** "add 3 cases of Blue Buffalo large breed" with a draft order containing that item at qty 2  
**When** `order_update` triggers  
**Then** `order_items.final_qty` is updated to 5 in the DB  
**And** the injected context reads: "Updated: Blue Buffalo Large Breed 30lb — qty set to 5."

**Given** "set Blue Buffalo to 10"  
**When** `order_update` triggers  
**Then** `final_qty` is set to 10 (absolute, not delta)

**Given** "remove the Blue Buffalo"  
**When** `order_update` triggers  
**Then** `final_qty` is set to 0

**Given** "decrease by 2" with current qty 3  
**When** `order_update` triggers  
**Then** `final_qty` is set to 1 (not negative — floors at 0)

**Given** no item in the order matches the fragment  
**When** `order_update` triggers  
**Then** injected context: "No item matching '{fragment}' found on this order."  
**And** NO DB write occurs

**Given** two or more items match the fragment  
**When** `order_update` triggers  
**Then** injected context lists the matches: "Which one did you mean? Blue Buffalo LB 30lb, Blue Buffalo LB 15lb"  
**And** NO DB write occurs

**Given** the order is already submitted (`session_metadata["submitted"] == True`)  
**When** any mutation keyword triggers `order_update`  
**Then** injected context: "This order is already submitted and can't be modified."  
**And** NO DB write occurs

**Given** a message with mutation keywords but NO `order_id` in session_metadata  
**When** `dispatch_tools` runs  
**Then** `order_update` does NOT trigger (trigger_fn returns False)

**Given** a message with no mutation keywords ("how are sales?")  
**When** `dispatch_tools` runs  
**Then** `order_update` does NOT trigger

---

## TDD Plan (Article 7)

**Red phase — write and commit these failing tests first:**

`tests/tools/test_order_update.py`

```python
# --- Trigger tests ---
# Test 1: trigger returns True for "add 3 cases of Blue Buffalo" + order_id present
# Test 2: trigger returns True for "set to 5" + order_id present
# Test 3: trigger returns True for "remove the Blue Buffalo" + order_id present
# Test 4: trigger returns True for "increase by 2" + order_id present
# Test 5: trigger returns True for "decrease by 1" + order_id present
# Test 6: trigger returns False for "how are sales looking" (no mutation keyword)
# Test 7: trigger returns False for "add 3 cases" when no order_id in session_metadata

# --- Qty parsing tests (pure function, no DB) ---
# Test 8: parse_qty_intent("add 3 cases", current_qty=2) → ("delta", +3)
# Test 9: parse_qty_intent("set to 10", current_qty=2) → ("absolute", 10)
# Test 10: parse_qty_intent("remove", current_qty=5) → ("absolute", 0)
# Test 11: parse_qty_intent("decrease by 2", current_qty=1) → ("delta", -2) → floor 0
# Test 12: parse_qty_intent("increase by 3", current_qty=4) → ("delta", +3)
# Test 13: parse_qty_intent("5", current_qty=0) → ("absolute", 5) (bare number = set)

# --- Item matching tests (mock DB) ---
# Test 14: update_order_item() with 1 match → DB UPDATE called, confirmation injected
# Test 15: update_order_item() with 0 matches → no DB write, "No item matching..." injected
# Test 16: update_order_item() with 2+ matches → no DB write, ambiguity message injected
# Test 17: update_order_item() with submitted=True → no DB write, "already submitted" injected
# Test 18: update_order_item() qty floors at 0 (decrease by more than current)

# --- Integration test ---
# Test 19: full dispatch_tools call → order_update fires, DB mock called, context injected
```

**Green phase — implement:**

`src/kaylee_agent/tools/order_update.py`:

```python
_MUTATION_KEYWORDS = frozenset(["add", "set", "change", "update", "remove", "increase", "decrease", "bump"])

def _trigger(user_message: str, session_metadata: dict | None = None) -> bool:
    if not session_metadata or not session_metadata.get("order_id"):
        return False
    msg = user_message.lower()
    return any(kw in msg for kw in _MUTATION_KEYWORDS)

def _parse_item_fragment(user_message: str) -> str:
    """Extract item name fragment from mutation message."""
    ...

def _parse_qty_intent(user_message: str, current_qty: int) -> int:
    """Return the new absolute final_qty, floored at 0."""
    ...

async def update_order_item(pool: Any, user_message: str, session_metadata: dict | None = None) -> str:
    ...
```

**SQL (item lookup):**
```sql
SELECT id, item_name, final_qty
FROM order_items
WHERE order_id = $1
  AND item_name ILIKE $2
```

**SQL (write):**
```sql
UPDATE order_items
SET final_qty = $1, updated_at = NOW()
WHERE id = $2 AND order_id = $3
RETURNING item_name, final_qty
```

Register with `needs_central_pool=True, needs_session_metadata=True`.

**Note on trigger_fn signature:** `ToolEntry.trigger_fn` currently takes only `user_message: str`. This story requires the trigger to also check `session_metadata`. Two options:
1. **Preferred:** Keep `trigger_fn(user_message)` — pass `session_metadata` via a closure at registration time (capture it when the tool entry is evaluated per-request). Requires dispatch_tools to re-evaluate the registry per call with current session_metadata.  
2. **Alternative:** Extend trigger_fn signature to `trigger_fn(user_message, session_metadata)` — a small addition to `dispatch_tools` dispatch loop.

**Decision:** Option 2 is cleaner and avoids closure lifetime issues. `dispatch_tools` passes `session_metadata` to `trigger_fn` as a second optional argument. Existing triggers ignore it (they only take `user_message`). Use `inspect.signature` to detect if the trigger accepts a second arg, or document that all triggers must accept `**kwargs`. Test this in Story 6.1 tests.

---

## Tasks

- [ ] Write failing tests (`test_order_update.py`) — commit as RED
- [ ] Implement `_parse_item_fragment` and `_parse_qty_intent` pure functions (easiest to test)
- [ ] Implement `update_order_item` async function with guards
- [ ] Implement `_trigger` with session_metadata check
- [ ] Register in `TOOL_REGISTRY`
- [ ] Handle trigger_fn session_metadata signature (coordinate with 6.1 approach)
- [ ] Run all tests — confirm RED → GREEN, zero regression on existing tools
- [ ] Commit as GREEN
