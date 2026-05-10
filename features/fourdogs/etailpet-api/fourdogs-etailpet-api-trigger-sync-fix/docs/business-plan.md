---
feature: fourdogs-etailpet-api-trigger-sync-fix
doc_type: business-plan
status: draft
goal: >
  Add a force-trigger HTTP endpoint to the fourdogs-central API server so that
  etailpet data ingestion (inventory, sales, and transactions) can be invoked
  on demand without waiting for the next scheduled cron tick.
key_decisions:
  - Endpoint lives in the fourdogs-central API server alongside the existing velocity-risk/refresh pattern
  - All three data sources (inventory, sales, transactions) are triggerable via a single parameterized endpoint
  - No new binary or deployment — the API server already has access to etailpet.NewClient
  - Callers authenticate via the internal bearer token pattern (same as velocity-risk/refresh — token injected at startup)
  - The endpoint is intentionally idempotent — calling it while a cycle is already running is safe (etailpet API deduplication assumed; concurrent safety confirmed pre-implementation)
open_questions: []
depends_on: []
blocks:
  - fourdogs-central-orders-manual-refresh
updated_at: "2026-05-10"
---

# Business Plan — fourdogs-etailpet-api-trigger-sync-fix

## Executive Summary

The etailpet trigger pods (inventory, sales, transactions) run on a fixed cron schedule.
When a pod is unhealthy during its scheduled window, data becomes stale and there is no
mechanism to recover without waiting for the next scheduled run — which may be hours away.

This feature adds a `POST /internal/etailpet/trigger` endpoint to the central API server.
The endpoint invokes the same fetch cycle the trigger pods execute on their schedule,
allowing an operator or an internal caller (e.g., the orders screen refresh button in
central-ui) to force a data update on demand.

---

## Business Context

The Four Dogs ordering workflow depends on fresh etailpet data (inventory levels, sales
history, transaction records). When the trigger pods miss a scheduled run due to pod
restarts, node issues, or transient ArgoCD drift, the data displayed to operators is
stale and there is no self-service way to recover.

The pattern already exists in this codebase: `POST /internal/velocity-risk/refresh`
does exactly the same thing for the risk scoring pipeline. This feature applies the
same approach to the etailpet ingestion pipeline.

---

## Stakeholders

| Role | Name | Interest |
|---|---|---|
| Operator (primary user) | Four Dogs store staff | Orders screen shows current inventory and sales data |
| Platform on-call | Engineering | Recovery from missed trigger runs without a pod bounce |

---

## Success Criteria

1. `POST /internal/etailpet/trigger?source=inventory` successfully runs an inventory fetch cycle and returns `200 OK` with a structured JSON response.
2. `POST /internal/etailpet/trigger?source=sales` and `?source=transactions` work identically for their respective sources.
3. Calling `?source=all` or omitting `source` triggers all three sources sequentially.
4. The `data/freshness` endpoint reflects an updated timestamp after a successful trigger call.
5. The endpoint uses bearer token auth (injected via config at startup, same as velocity-risk/refresh). It is accessible via the existing internal ingress or port-forward path used by other /internal/ routes.

---

## Out of Scope

- Changes to the trigger pod deployments or Helm charts
- Fixing any specific ArgoCD drift — that is an operational action
- Changes to the trigger schedule or cron configuration
- UI changes — the orders screen button wire-up is owned by `fourdogs-central-orders-manual-refresh`
