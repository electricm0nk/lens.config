# ExpressPlan Adversarial Review — fourdogs-kaylee-data-access

**Phase:** expressplan  
**Source:** phase-complete  
**Verdict:** pass-with-warnings  
**Date:** 2026-05-21  
**Reviewer:** Lens Adversarial Review Gate + Party-Mode Challenge

---

## Artifact Set Reviewed

| Artifact | Path | Status |
|---|---|---|
| `business-plan.md` | docs/fourdogs/kaylee/fourdogs-kaylee-data-access/business-plan.md | ✅ Present |
| `tech-plan.md` | docs/fourdogs/kaylee/fourdogs-kaylee-data-access/tech-plan.md | ✅ Present |
| `sprint-plan.md` | docs/fourdogs/kaylee/fourdogs-kaylee-data-access/sprint-plan.md | ✅ Present |

---

## Verdict: pass-with-warnings

The expressplan artifacts are coherent, implementation-ready, and correctly scoped. No
critical blockers remain after pre-review triage. Three high-severity findings require
resolution during implementation before stories are marked done. FinalizePlan may proceed
after applying accepted review findings to the planning documents.

---

## Critical Findings

*(None — CRIT-1 downgraded to HIGH after reviewing DA-5 acceptance criteria.)*

---

## High Severity Findings

**HIGH-1: `created_by` field in `order_create` will be NULL at runtime**

The tech plan specifies `created_by = session_metadata.get("session_id")`. However,
`session_metadata` is the `metadata` JSON blob stored in the `sessions` table —
`session_id` is a separate column, not a key inside this blob. The `orders.created_by`
column is NOT NULL, so the INSERT will fail.

**Resolution:** DA-5 implementation must wire `session_id` explicitly. Options:
(a) Extend `ToolEntry` with `needs_session_id=True` and pass it from the chat route
where `session_id` is already in scope; or (b) have the Central UI store
`session_id` in the metadata blob at session creation. Confirm with Central UI before
DA-5 implementation begins. Add as a pre-implementation checklist item in DA-5.

---

**HIGH-2: `dateparser` dependency not declared; date NLP is underspecified**

`sales_query` requires date parsing ("last month", "March 12th", "last week") but
`dateparser` is not in `pyproject.toml`. The sprint plan defers this without a
concrete resolution path.

**Resolution:** DA-2 must include:
- Add `dateparser` to `pyproject.toml` (or implement a minimal regex parser)
- Test at minimum: "last month", "March 12th", "last week", "yesterday"
- Document fallback behavior when date is unparseable → default to last 30 days
- The tool docstring must state the supported patterns

---

**HIGH-3: Existing tool tests must be updated when privilege error stubs are retired**

DA-1 removes the `InsufficientPrivilegeError` stub strings from 5 existing tools.
Current tests may assert the stub string appears on the privilege-error code path.
After grants are applied and stubs removed, those tests will incorrectly pass (stubs
absent → no assertion failure, but behavior changed silently).

**Resolution:** DA-1 must include: review and update test suites for `sales_summary`,
`top_products`, `customer_lookup`, `inventory_velocity`, `order_view` to reflect the
post-grant behavior on the privilege-error code path. If tests were written for the
stub, update them.

---

## Medium Severity Findings

**MED-1: No runbook (Article 11 — org constitution hard gate)**

The org constitution requires a `runbook.md` for materially changed infrastructure
components. Applying Postgres grants is a material operational change. No runbook
story exists.

**Resolution:** Add story DA-0 (or merge into DA-1): create
`docs/fourdogs/kaylee/fourdogs-kaylee-data-access/runbook.md` documenting:
- Patroni access path (e.g., `kubectl exec -it postgresql-0 -n fourdogs`)
- Grant verification commands (`\dp` or `SELECT grantee, privilege_type FROM information_schema.role_table_grants WHERE grantee = 'kaylee_agent'`)
- Grant application SQL
- Verification that existing tools return data (not privilege error)
- Last Verified timestamp

---

**MED-2: Truncation not signaled to operator**

All query tools use `LIMIT 100` but do not signal when the result is truncated.
An operator who asks "show all sales last month" will silently get 100 rows.

**Resolution:** Update tech-plan and add AC to each query story (DA-2, DA-3, DA-4):
When `len(rows) == LIMIT`, append to output: "Showing first 100 results. Narrow
the filter (date range, customer, or product) to see fewer results."

---

**MED-3: Monetary field formatting not specified for `transaction_query`**

Transactions carry `net_sale`, `total`, `profit`, `margin` (decimal). No formatting
standard is specified.

**Resolution:** Add to tech-plan formatting spec: monetary fields → `${value:,.2f}`,
percentage fields (margin) → `{value:.1f}%`. Match existing `sales_summary` pattern.

---

## Low Severity Findings

**LOW-1: Category queries not explicitly supported in `sales_query`**

The `sales` table has a `category` column. Users may ask "what pet food sold
best?" — a category filter. The tech plan does not enumerate category as a
supported query dimension.

**Resolution:** Add category-filter support to `sales_query` or explicitly document
it as out-of-scope for this sprint (deferred to a follow-on feature).

---

**LOW-2: `order_archive` has no idempotency test for already-archived state**

DA-6 covers `order_submit` idempotency ("already submitted" guard) but not
`order_archive`.

**Resolution:** Add AC to DA-6: "Given order already archived, When `order_archive`
triggered, Then return 'Order is already archived.'"

---

**LOW-3: Multi-tool context window not addressed**

If 3 tools fire on one message (sales + inventory + transactions), 300 rows of
data may be injected into the LLM context. No token budget analysis exists.

**Resolution:** Document in tech-plan as a known risk: recommend operators use
specific queries over broad queries. Consider adding a future story for per-tool
result size caps if context overflow becomes an operational issue.

---

## Party-Mode Blind-Spot Challenge

**Suzie Q (Store Owner / Primary Operator):**
> You said Kaylee can look up "what did Suzie buy last month" — but what if I ask
> "show me all customers who bought raw food last week"? That's a product-category
> query with no customer filter. Does `sales_query` handle "no customer, filter by
> category"? I don't see a category trigger in the tech plan. LOW-1 captures this
> but it needs a decision, not just a note.

**Bob (SM):**
> DA-1 says "remove stub error messages after grants confirmed" — but is there a
> rollback plan if the grants cause a regression? We're going from zero queries to
> potentially many per chat message on a large sales table. Is there a EXPLAIN ANALYZE
> step in DA-1 for the new query patterns? I don't see any mention of the index
> coverage on `sales.customer_name` or `sales.product_name`.

**Winston (Architect):**
> The dispatcher fires ALL triggered tools on every message. If I type "show me
> sales and what's on hand for Blue Buffalo", both `sales_query` AND
> `inventory_lookup` trigger. 200 rows injected into one prompt. What is the
> terminus-inference-gateway context limit? Has anyone measured the token cost of
> a 100-row sales dump? This is a real context-overflow risk in production.

---

### Blind-Spot Challenge Questions for Operator Review

1. **Category queries:** Should `sales_query` support category-filter queries ("what
   pet food sold best")? If yes, add to DA-2. If no, document as explicitly deferred.
2. **Index coverage:** Confirm there is a `customer_name` index on `sales` before
   implementing DA-2. If missing, create it as part of the grant verification story.
   An `ILIKE '%name%'` with no index is a full table scan.
3. **`created_by` wiring:** Confirm with Central UI team: does the `POST /sessions`
   body store any user identifier inside the `metadata` blob? If not, DA-5 needs a
   dispatcher change to pass `session_id` as a separate kwarg.
4. **Multi-tool context:** What is the token limit of the model being served through
   terminus-inference-gateway? Is there a risk that 2–3 tool results exhaust the
   context window on a long session?
5. **`sales` row count in prod:** How many rows are in the `sales` table? This drives
   the LIMIT and query performance decisions.

---

## Accepted Review Findings Integration Guidance for FinalizePlan

Before FinalizePlan generates downstream bundle artifacts, the following changes
must be applied to the planning documents:

1. **business-plan.md:** Add `created_by` wiring as an explicit risk under the
   dependencies section.
2. **tech-plan.md:** Add: (a) `dateparser` dependency declaration; (b) truncation
   signal requirement for all query tools; (c) monetary formatting spec for
   `transaction_query`; (d) `created_by` sourcing clarification for `order_create`;
   (e) multi-tool context window as a documented risk.
3. **sprint-plan.md:** Add: (a) DA-0 runbook story; (b) `dateparser` task to DA-2;
   (c) test-update task to DA-1; (d) truncation AC to DA-2, DA-3, DA-4;
   (e) `order_archive` idempotency AC to DA-6; (f) `created_by` pre-implementation
   check to DA-5.
