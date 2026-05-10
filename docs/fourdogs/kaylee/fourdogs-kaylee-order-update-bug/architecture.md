# Architecture: Order Update Bug Fix

**Feature:** fourdogs-kaylee-order-update-bug  
**Track:** hotfix-express  
**Phase:** techplan  
**Date:** 2026-05-10  
**Author:** Todd Hintzmann

---

## Root Cause

Three compounding gaps produce the no-op order update bug:

### Gap 1 — System prompt promises a capability that doesn't exist

The `ORDERING MODE` section of `/etc/kaylee/system-prompt.txt` instructs the LLM:

> "When Betsy tells you to adjust the order worksheet — add cases, remove items, change quantities — **do it and confirm briefly**."

The LLM is a compliant instruction-follower. It confidently says "Done — added 3 cases of Blue Buffalo!" because that is what it was told to do. There is no mechanism to tell it that the write never happened.

### Gap 2 — No order mutation tools exist

`src/kaylee_agent/tools/` contains six read-only tools:
- `customer_lookup`, `dispatcher`, `get_current_date`, `inventory_velocity`, `sales_summary`, `top_products`

There are no write-capable tools. No `order_view`, no `order_update`, no `order_submit`. The `TOOL_REGISTRY` is empty of any order action.

### Gap 3 — `dispatch_tools` cannot pass `order_id` to tools

`dispatch_tools(messages, user_message, pool, central_pool)` has no mechanism to forward `session_metadata` to tools. Even if a write tool existed, it would have no way to know which order to modify. The `order_id` is assembled into the messages list as an `[ORDER CONTEXT]` block but is not passed as a structured argument.

### Result

The LLM receives an [ORDER CONTEXT] block (order_id, vendor, items count, budget) via `assemble_messages`, hallucinates a successful mutation, and the user sees a confident confirmation that nothing actually executed.

---

## Fix Architecture

Four focused changes, all within `fourdogs-kaylee-agent`. No changes to `fourdogs-central` or the EtailPet API are in scope.

### Component 1 — Extend `ToolEntry` and `dispatch_tools` for session_metadata

**File:** `src/kaylee_agent/tools/dispatcher.py`

Add `needs_session_metadata: bool = False` to `ToolEntry`. Extend `dispatch_tools` to accept `session_metadata: dict | None = None` and pass it as a keyword argument to any tool with `needs_session_metadata=True`. Fully backward compatible — existing tools are unaffected.

```
ToolEntry(
    name="order_update",
    trigger_fn=...,
    tool_fn=update_order_item,
    needs_central_pool=True,
    needs_session_metadata=True,   # NEW
)
```

`dispatch_tools` calling convention for these tools:
```python
raw = entry.tool_fn(central_pool, session_metadata=session_metadata)
```

`chat.py` passes `session_metadata=session_meta` to `dispatch_tools`.

### Component 2 — `order_view` tool (read)

**File:** `src/kaylee_agent/tools/order_view.py`

**Trigger keywords:** `"show.*order"`, `"what.*in.*order"`, `"current order"`, `"order list"`, `"view.*order"`, `"whats.*on.*order"`

**Behavior:**
1. Extracts `order_id` from `session_metadata`. If absent → returns empty string (tool is a no-op without order context).
2. Queries `order_items` in `fourdogs_central` DB:
   ```sql
   SELECT item_name, final_qty, ghost_qty, current_stock_qty, velocity_tier, must_have
   FROM order_items
   WHERE order_id = $1
   ORDER BY must_have DESC, item_name ASC
   ```
3. Returns a formatted markdown table of current order state.

**DB access:** `central_pool` (fourdogs_central postgres, read-only query)

### Component 3 — `order_update` tool (write)

**File:** `src/kaylee_agent/tools/order_update.py`

**Trigger keywords:** `"add"`, `"set"`, `"change"`, `"update"`, `"remove"`, `"increase"`, `"decrease"`, `"bump"` — but **only** when an `order_id` is present in `session_metadata`. If no `order_id`, trigger returns False.

**Behavior:**
1. Guard: if `session_metadata.get("submitted")` is True → inject context: "Order is already submitted. Cannot modify a submitted order." No DB write.
2. Parse user message for:
   - **Item fragment:** noun phrase after action verb (e.g. "add 3 cases of Blue Buffalo" → "Blue Buffalo")
   - **Quantity intent:** regex patterns for `N cases`, `N bags`, `N`, `set to N`, `increase by N`, `decrease by N`, `remove` (→ 0)
3. Fuzzy-match item by `ILIKE '%{fragment}%'` on `order_items.item_name` for the given `order_id`. Return all matches.
   - **0 matches:** inject "No item matching '{fragment}' found in this order."
   - **2+ matches:** inject list of matches — "Which one did you mean? [item A, item B]" No write.
   - **1 match:** proceed to write.
4. Compute new `final_qty`:
   - `set to N` or bare `N` → `N`
   - `add N` / `increase by N` → `current final_qty + N`
   - `remove` / `set to 0` → `0`
   - `decrease by N` → `max(0, current final_qty - N)`
5. Guard: `final_qty >= 0` always.
6. Execute:
   ```sql
   UPDATE order_items SET final_qty = $1, updated_at = NOW()
   WHERE id = $2 AND order_id = $3
   RETURNING item_name, final_qty
   ```
7. Inject result: "Updated: {item_name} — qty set to {final_qty}."

**DB access:** `central_pool` with a write (`UPDATE`). The `fourdogs_kaylee_rw` role already used by other tools must have `UPDATE` privilege on `order_items` (see Story 6.1 prerequisite check).

### Component 4 — System prompt update

**Configmap:** `kaylee-agent-config` in `fourdogs-kaylee` and `fourdogs-kaylee-dev`

Add to the `ORDERING MODE` section:

```
IMPORTANT: You can only modify draft orders (not submitted). When you adjust quantities,
confirm what was actually saved — not what you intended. If the tool returns an error or
ambiguity, relay it exactly so Betsy can clarify.
```

This constrains the LLM to report tool results faithfully rather than hallucinating success.

---

## Data Flow (After Fix)

```
User: "add 3 cases of Blue Buffalo large breed"
  │
  ├─ assemble_messages()  → [system] + [ORDER CONTEXT order_id=abc] + [history] + [user message]
  │
  ├─ dispatch_tools(messages, user_message, central_pool=..., session_metadata={order_id:"abc", submitted:False})
  │     │
  │     ├─ order_view.trigger("add 3 cases...")  → False (not a view request)
  │     │
  │     └─ order_update.trigger("add 3 cases...") → True (has "add" + order_id present)
  │           │
  │           ├─ parse: fragment="Blue Buffalo large breed", delta=+3
  │           ├─ ILIKE match → 1 result: id=xyz, item_name="Blue Buffalo Large Breed 30lb", final_qty=2
  │           ├─ new_qty = 2 + 3 = 5
  │           ├─ UPDATE order_items SET final_qty=5 WHERE id=xyz AND order_id=abc
  │           └─ inject: "[TOOL CONTEXT: order_update]\nUpdated: Blue Buffalo Large Breed 30lb — qty set to 5."
  │
  └─ gateway POST → LLM sees actual tool result → responds accurately
```

---

## Privilege Check

The `fourdogs_kaylee_rw` role (used by `central_pool`) must have `UPDATE` on `fourdogs_central.order_items`. Story 6.1 includes a verification step; a one-time DBA grant is tracked as a story task (not a migration, matching the pattern from Story 1.7).

---

## Scope Boundary

| In scope | Out of scope |
|---|---|
| `order_view` read tool | EtailPet purchase-order API |
| `order_update` write tool | Creating new orders |
| `dispatch_tools` session_metadata extension | Submitting orders |
| System prompt tightening | Order archiving / deletion |
| TDD tests for all new tools | fourdogs-central changes |

---

## Stories

| ID | Title | Depends On |
|---|---|---|
| 6.1 | Extend dispatcher for session_metadata + verify UPDATE grant | — |
| 6.2 | `order_view` tool — read current order items (TDD) | 6.1 |
| 6.3 | `order_update` tool — set final_qty with guards (TDD) | 6.1 |
| 6.4 | System prompt update + helm configmap patch | 6.3 |
