# Story 5.8: Order context injection from fourdogs-central

**Feature:** real-data-grounding  
**Epic:** 5 — Kaylee Real Data & Context Grounding  
**Repos:** fourdogs-central + fourdogs-kaylee-agent  
**Status:** backlog  
**Priority:** medium  
**Estimate:** M  
**Depends on:** none (independent of 5.1–5.7)

---

## User Story

As the AI agent,
I want to receive order metadata (vendor name, order date, item count, budget) when fourdogs-central
creates my session,
so that I can help Betsy with the specific order she is viewing without her having to re-explain
which order she is working on.

---

## Context

When a user opens the Kaylee chat in fourdogs-central, `PostKayleeMessage()` calls
`createKayleeSession()` with an empty JSON body `{}`. Kaylee has no idea which order is being
discussed unless the user explains it in chat.

The fix has two sides:
1. **fourdogs-central** — `PostKayleeMessage` fetches the order from DB and sends order metadata
   in the POST /sessions body when creating the session
2. **fourdogs-kaylee-agent** — `message_assembly.py` reads session metadata and prepends an order
   context block after the system message

The kaylee-agent side already accepts session metadata in `POST /sessions` (stored as JSONB). The
gap is (a) central not sending it, and (b) the message assembler not reading it.

### Current fourdogs-central handler (paraphrased)

```go
// kaylee.go
func (h *KayleeHandler) PostKayleeMessage(w http.ResponseWriter, r *http.Request) {
    // ... reads session_id and message from request
    if sessionID == "" {
        sessionID = h.createKayleeSession()  // sends {} body
    }
    // ... proxies message to kaylee-agent
}

func (h *KayleeHandler) createKayleeSession() string {
    resp, _ := http.Post(kayleeURL+"/sessions", "application/json", strings.NewReader("{}"))
    // ... reads session_id from response
}
```

### Order ID source

The order ID is available in the HTTP request: either as a URL parameter (`/orders/{order_id}/kaylee`)
or as a request body field. Confirm the route definition in fourdogs-central before implementing.

---

## Acceptance Criteria

**fourdogs-central:**

**Given** a user opens Kaylee chat while viewing order `abc-123`,
**When** fourdogs-central calls `createKayleeSession`,
**Then** the POST /sessions body includes:
```json
{
  "metadata": {
    "order_id": "abc-123",
    "vendor_name": "PetSmart Wholesale",
    "order_date": "2026-04-15",
    "budget_cents": 500000,
    "submitted": false,
    "item_count": 12
  }
}
```

**Given** the order query fails (order not found),
**When** `createKayleeSession` is called,
**Then** the session is still created with `metadata: {}` — order context failure is non-fatal

**fourdogs-kaylee-agent:**

**Given** a session was created with order metadata,
**When** `assemble_messages` is called,
**Then** the assembled messages array includes a `role: "user"` message immediately after the system
message containing:
```
[ORDER CONTEXT]
Order ID: abc-123
Vendor: PetSmart Wholesale
Order Date: 2026-04-15
Budget: $5,000.00
Status: draft
Items in order: 12
```

**Given** a session has no metadata (empty `{}`),
**When** `assemble_messages` is called,
**Then** no order context block is injected — messages assemble normally

**Given** a session has metadata with missing fields,
**When** order context is formatted,
**Then** missing fields are omitted from the context block rather than displaying "None" or crashing

---

## Technical Notes

### fourdogs-central changes

**`internal/handler/kaylee.go`**

1. Determine order_id from request context (route param or body field — check chi router definition)
2. Add `db.Queries` or query function to `KayleeHandler` if not already present
3. Call `GetOrderByID(ctx, orderID)` to fetch order row
4. Call `countOrderItems(ctx, orderID)` or use the `order_items` count query
5. Build metadata struct and pass to `createKayleeSession`

Suggested struct (internal only, serialized to JSON):
```go
type kayleeSessionMetadata struct {
    OrderID     string `json:"order_id"`
    VendorName  string `json:"vendor_name,omitempty"`
    OrderDate   string `json:"order_date,omitempty"`
    BudgetCents int64  `json:"budget_cents,omitempty"`
    Submitted   bool   `json:"submitted"`
    ItemCount   int    `json:"item_count"`
}
```

`createKayleeSession` updated signature:
```go
func (h *KayleeHandler) createKayleeSession(ctx context.Context, meta *kayleeSessionMetadata) string
```

Build POST body:
```go
body := map[string]interface{}{"metadata": meta}
// or {"metadata": {}} if meta is nil
```

### fourdogs-kaylee-agent changes

**`src/kaylee_agent/message_assembly.py`**

Update `assemble_messages` signature:
```python
def assemble_messages(
    system_prompt: str,
    history: list[dict],
    user_message: str,
    session_metadata: dict | None = None,
) -> list[dict]:
```

If `session_metadata` contains an `order_id`:
```python
if session_metadata and session_metadata.get("order_id"):
    order_context = _format_order_context(session_metadata)
    messages = [
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": order_context},
        *history,
        {"role": "user", "content": user_message},
    ]
else:
    messages = [
        {"role": "system", "content": system_prompt},
        *history,
        {"role": "user", "content": user_message},
    ]
```

Helper:
```python
def _format_order_context(meta: dict) -> str:
    lines = ["[ORDER CONTEXT]"]
    if v := meta.get("order_id"):
        lines.append(f"Order ID: {v}")
    if v := meta.get("vendor_name"):
        lines.append(f"Vendor: {v}")
    if v := meta.get("order_date"):
        lines.append(f"Order Date: {v}")
    if v := meta.get("budget_cents"):
        lines.append(f"Budget: ${v / 100:,.2f}")
    submitted = meta.get("submitted", False)
    lines.append(f"Status: {'submitted' if submitted else 'draft'}")
    if v := meta.get("item_count"):
        lines.append(f"Items in order: {v}")
    return "\n".join(lines)
```

**`src/kaylee_agent/routes/chat.py`**

Fetch session metadata before calling `assemble_messages`:
```python
session_metadata = await persistence.get_session_metadata(pool, body.session_id)
messages = assemble_messages(
    system_prompt,
    history,
    body.message,
    session_metadata=session_metadata,
)
```

**`src/kaylee_agent/persistence.py`**

Add:
```python
async def get_session_metadata(pool: asyncpg.Pool, session_id: str) -> dict | None:
    row = await pool.fetchrow(
        "SELECT metadata FROM sessions WHERE id = $1",
        session_id
    )
    if row is None:
        return None
    return dict(row["metadata"]) if row["metadata"] else {}
```

### fourdogs-central DB queries needed

If not already present in `internal/db/queries/`:
```sql
-- name: GetOrderItemCount :one
SELECT COUNT(*) FROM order_items WHERE order_id = $1;
```

`GetOrderByID` should already exist — check `internal/db/queries/orders.sql`.

---

## Definition of Done

**fourdogs-central:**
- [ ] `KayleeHandler` fetches order and item count when creating kaylee session
- [ ] `createKayleeSession` sends `{"metadata": {...}}` with order context
- [ ] Order fetch failure is non-fatal (falls back to `{}` metadata)
- [ ] Unit/integration test: session creation request includes order metadata
- [ ] CI green

**fourdogs-kaylee-agent:**
- [ ] `assemble_messages` accepts `session_metadata` param
- [ ] Order context block injected as `role: "user"` message when metadata has `order_id`
- [ ] No injection when metadata is empty or nil
- [ ] `persistence.get_session_metadata()` added
- [ ] `chat.py` fetches and passes session metadata
- [ ] Unit tests: with and without metadata; partial metadata fields
- [ ] CI green

**Integration:**
- [ ] End-to-end test: open order in central app, send Kaylee a vague question ("what about this one?"); Kaylee references the correct vendor/order in response
