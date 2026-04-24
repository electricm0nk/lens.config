# Story 5.2: Postgres Grant — fourdogs_kaylee_rw on fourdogs_central

**Feature:** real-data-grounding  
**Epic:** 5 — Kaylee Real Data & Context Grounding  
**Repo:** terminus.infra  
**Status:** backlog  
**Priority:** high  
**Estimate:** S  
**Blocks:** Stories 5.3, 5.4, 5.5, 5.6  
**Supersedes:** fourdogs-kaylee-agent Story 1.7 (partially; extends scope to include `order_items` and `sales`)

---

## User Story

As a DBA,
I want `fourdogs_kaylee_rw` granted CONNECT on `fourdogs_central` and SELECT on the tables queried by
kaylee's data tools,
so that kaylee-agent can connect to `fourdogs_central` using its existing credential and read the data
it needs without a permission error.

---

## Context

`fourdogs_kaylee_rw` is the Postgres role used by kaylee-agent (both dev and prod). It currently has
full access to `fourdogs_kaylee` but no access to `fourdogs_central`. Story 1.7 was written before the
scope was fully known — it only mentioned `items` and `products`. This story supersedes it with the
correct table list derived from the actual tool queries.

**Postgres host:** Patroni at `10.0.0.56` (SSH: `ubuntu@10.0.0.56` or via control plane `ubuntu@10.0.0.251`)  
**Source DB:** `fourdogs_kaylee`  
**Target DB:** `fourdogs_central`  
**Role to grant:** `fourdogs_kaylee_rw`

**Tables required by tools:**

| Table | Used by | Query type |
|---|---|---|
| `items` | `inventory_velocity`, `top_products` | SELECT (risk_score, dos_days, velocity_class, qty) |
| `order_items` | `inventory_velocity`, `top_products` | SELECT (item_id, final_qty, velocity_tier, must_have) |
| `sales` | `sales_summary`, `customer_lookup` | SELECT (net_total, sale_date, customer_name, customer_id, quantity, product_name) |

---

## Acceptance Criteria

**Given** the grant script is executed as a Postgres superuser on `fourdogs_central`,
**When** `fourdogs_kaylee_rw` connects to `fourdogs_central` and runs `SELECT * FROM items LIMIT 1`,
**Then** the query returns results without a `permission denied` error

**Given** the grant is applied,
**When** `fourdogs_kaylee_rw` runs `SELECT * FROM order_items LIMIT 1` and `SELECT * FROM sales LIMIT 1`,
**Then** both queries succeed without permission errors

**Given** the grant is applied,
**When** `fourdogs_kaylee_rw` attempts `INSERT INTO items ...` or `DELETE FROM items ...`,
**Then** the operation is rejected with `permission denied` — SELECT-only is enforced

**Given** the grant script is run a second time,
**When** it completes,
**Then** it exits cleanly with no errors (idempotent using `GRANT ... IF NOT EXISTS` or equivalent)

**Given** `CENTRAL_DATABASE_URL` is added to the Vault kaylee secrets,
**When** kaylee-agent starts,
**Then** the central pool connects without error

---

## Technical Notes

### Grant script

Execute as `postgres` superuser while connected to `fourdogs_central`:

```sql
-- Allow fourdogs_kaylee_rw to connect to the fourdogs_central database
GRANT CONNECT ON DATABASE fourdogs_central TO fourdogs_kaylee_rw;

-- Grant schema visibility
GRANT USAGE ON SCHEMA public TO fourdogs_kaylee_rw;

-- Grant SELECT on tables used by kaylee tools
GRANT SELECT ON items TO fourdogs_kaylee_rw;
GRANT SELECT ON order_items TO fourdogs_kaylee_rw;
GRANT SELECT ON sales TO fourdogs_kaylee_rw;
```

To execute:
```bash
# From control plane or Vault host:
ssh ubuntu@10.0.0.251 "kubectl -n fourdogs-central-dev exec deploy/fourdogs-central -- \
  psql \$DATABASE_URL -c 'GRANT CONNECT ON DATABASE fourdogs_central TO fourdogs_kaylee_rw;'"

# Or directly via psql on Patroni:
psql -h 10.0.0.56 -U postgres -d fourdogs_central -c "
GRANT CONNECT ON DATABASE fourdogs_central TO fourdogs_kaylee_rw;
GRANT USAGE ON SCHEMA public TO fourdogs_kaylee_rw;
GRANT SELECT ON items TO fourdogs_kaylee_rw;
GRANT SELECT ON order_items TO fourdogs_kaylee_rw;
GRANT SELECT ON sales TO fourdogs_kaylee_rw;
"
```

### Vault secret update

Add `CENTRAL_DATABASE_URL` to both dev and prod Vault paths:

```bash
# Dev
vault kv patch secret/terminus/dev/fourdogs/kaylee \
  CENTRAL_DATABASE_URL="postgresql://fourdogs_kaylee_rw:<password>@10.0.0.56:5432/fourdogs_central"

# Prod
vault kv patch secret/terminus/fourdogs/kaylee \
  CENTRAL_DATABASE_URL="postgresql://fourdogs_kaylee_rw:<password>@10.0.0.56:5432/fourdogs_central"
```

The `fourdogs_kaylee_rw` password is the same credential already used in `DATABASE_URL` — it's the role's
password, not DB-specific.

### Runbook file

Commit a runbook at `docs/fourdogs/kaylee/cross-db-grant-runbook.md` documenting:
- What was granted, when, by whom
- How to verify the grant is active
- How to extend if new tables are needed

---

## Definition of Done

- [ ] Grant script executed on Patroni; `fourdogs_kaylee_rw` can `SELECT` from `items`, `order_items`, `sales` in `fourdogs_central`
- [ ] SELECT-only verified (INSERT rejected)
- [ ] `CENTRAL_DATABASE_URL` added to Vault dev and prod kaylee secrets
- [ ] Runbook committed to `docs/fourdogs/kaylee/cross-db-grant-runbook.md`
- [ ] Story 1.7 in fourdogs-kaylee-agent Stories marked superseded/complete
