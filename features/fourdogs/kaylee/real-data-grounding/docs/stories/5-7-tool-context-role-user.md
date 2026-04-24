# Story 5.7: Tool context role fix — user-context injection instead of role:tool

**Feature:** real-data-grounding  
**Epic:** 5 — Kaylee Real Data & Context Grounding  
**Repo:** fourdogs-kaylee-agent  
**Status:** backlog  
**Priority:** medium  
**Estimate:** XS  
**Depends on:** none (independent)

---

## User Story

As a developer,
I want tool results injected as `role: "user"` context blocks with a bracketed header rather than
`role: "tool"` messages,
so that the OpenAI API never receives a `tool` role message without the required `tool_call_id`
field, which would cause a 400 error once tools start returning results reliably.

---

## Context

The dispatcher currently injects tool results as:
```python
{"role": "tool", "content": content, "tool_name": entry.name}
```

The OpenAI Chat Completions API requires `tool_call_id` on any `role: "tool"` message. Since kaylee
uses a soft trigger-dispatch pattern (not the OpenAI function-calling API), there is no `tool_call_id`
in the flow. Sending `role: "tool"` without `tool_call_id` is technically non-compliant and may cause
400 errors from OpenAI.

The inference gateway (`terminus-inference-gateway`) passes messages through raw — it does not
transform `role: "tool"` to `role: "user"`. So this fix must live in the dispatcher.

**Recommended format** (pragmatic, works with all models including non-OpenAI):
```
role: "user"
content: "[TOOL CONTEXT: inventory_velocity]\n{json_data}"
```

This is readable in conversation history, clearly labeled, and cannot cause a 400 for missing
`tool_call_id`. Kaylee's system prompt already instructs her to interpret bracketed context blocks.

---

## Acceptance Criteria

**Given** a tool executes successfully,
**When** the dispatcher appends the result,
**Then** the injected message has `"role": "user"` (not `"tool"`)

**Given** a tool result is injected,
**When** the message content is inspected,
**Then** it begins with `[TOOL CONTEXT: {tool_name}]\n` followed by the JSON-serialized result

**Given** the updated messages array is passed to the inference gateway,
**When** the gateway forwards to OpenAI,
**Then** no `tool_call_id` field is needed and no 400 error is produced

**Given** the existing unit tests for the dispatcher,
**When** they run after this change,
**Then** all 9 tests still pass (update any that assert on role value)

---

## Technical Notes

### File to change

**`src/kaylee_agent/tools/dispatcher.py`**

Only the injection line changes:

```python
# Before:
result.append({"role": "tool", "content": content, "tool_name": entry.name})

# After:
result.append({
    "role": "user",
    "content": f"[TOOL CONTEXT: {entry.name}]\n{content}",
})
```

The `tool_name` key is no longer needed in the dict (it was extra metadata not used by the gateway).

`content` is already JSON-serialized in the existing dispatcher — no change needed there.

### Inference gateway note

`terminus-inference-gateway` Go code has `openaiMessage struct { Role string; Content string }`.
The `role: "user"` message passes through cleanly. No gateway changes required.

### System prompt note

The kaylee system prompt at `/etc/kaylee/system-prompt.txt` should already reference `[TOOL CONTEXT]`
blocks. If not, a follow-up ConfigMap update may be needed. Check after deploying this story.

---

## Definition of Done

- [ ] `result.append({"role": "user", "content": f"[TOOL CONTEXT: {entry.name}]\n{content}"})` replaces old line
- [ ] `tool_name` key removed from injected dict
- [ ] All 9 existing dispatcher tests still pass (update role assertions if any)
- [ ] CI green
- [ ] Deployed to dev; Kaylee responds normally when tools inject context
