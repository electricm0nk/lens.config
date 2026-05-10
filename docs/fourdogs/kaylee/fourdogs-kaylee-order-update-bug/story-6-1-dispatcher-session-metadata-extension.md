# Story 6.1 — Extend Dispatcher for session_metadata + Verify UPDATE Grant

**Epic:** 6 — Order management tools  
**Track:** hotfix-express  
**Feature:** fourdogs-kaylee-order-update-bug

---

## Context

`dispatch_tools` has no way to pass the `order_id` (or any session-level context) to tools. This blocks all order-action tools. Additionally, the `fourdogs_kaylee_rw` DB role's write privileges on `order_items` must be verified before Story 6.3 can write safely.

## Acceptance Criteria

**Given** a `ToolEntry` with `needs_session_metadata=True`  
**When** `dispatch_tools` is called with `session_metadata={"order_id": "abc", "submitted": False}`  
**Then** the tool function receives `session_metadata` as a keyword argument

**Given** a `ToolEntry` with `needs_session_metadata=False` (all existing tools)  
**When** `dispatch_tools` is called with or without `session_metadata`  
**Then** existing tools are called exactly as before — no regression

**Given** `session_metadata=None` is passed  
**When** a `needs_session_metadata=True` tool triggers  
**Then** it is skipped silently (same skip-on-None pattern as central_pool)

**Given** the `fourdogs_kaylee_rw` role  
**When** connected to the fourdogs_central DB  
**Then** it can execute `UPDATE order_items SET final_qty = final_qty WHERE id = gen_random_uuid()` without a privilege error (a no-op UPDATE verifies the grant)

---

## TDD Plan (Article 7 — Red-Green Discipline)

**Red phase — write and commit these failing tests first:**

`tests/tools/test_dispatcher_session_metadata.py`

```python
# Test 1: needs_session_metadata tool receives session_metadata kwarg
# Test 2: needs_session_metadata=False tools are unaffected
# Test 3: session_metadata=None causes needs_session_metadata tool to be skipped
# Test 4: tool_fn receives central_pool AND session_metadata when both flags set
```

**Green phase — implement:**

`src/kaylee_agent/tools/dispatcher.py`:
- Add `needs_session_metadata: bool = False` to `ToolEntry` dataclass
- In `dispatch_tools`, add `session_metadata: dict | None = None` parameter
- In the dispatch loop, when `needs_session_metadata=True`: skip if `session_metadata is None`; else pass `session_metadata=session_metadata` as kwarg

`src/kaylee_agent/routes/chat.py`:
- Pass `session_metadata=session_meta` to `dispatch_tools`

---

## Tasks

- [ ] Write failing tests (`test_dispatcher_session_metadata.py`) — commit as RED
- [ ] Add `needs_session_metadata` to `ToolEntry`
- [ ] Update `dispatch_tools` signature and dispatch logic
- [ ] Update `chat.py` call site
- [ ] Verify `fourdogs_kaylee_rw` has UPDATE on `order_items` (kubectl exec into a pod and run a test query against central pool)
- [ ] If grant missing: document the grant command and run it as DBA task
- [ ] Run all existing tests — confirm zero regression
- [ ] Commit as GREEN
