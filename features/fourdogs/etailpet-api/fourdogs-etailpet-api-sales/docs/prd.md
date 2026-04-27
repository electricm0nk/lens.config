---
feature: sales
doc_type: prd
status: draft
goal: "Automate eTailPet sales-line item export using the www.etailpet.com legacy trigger API, delivering per-product sales detail to the emailfetcher pipeline to unlock SKU-level velocity signals for Kaylee and ordering."
key_decisions:
  - www.etailpet.com email-trigger path is the only available source for per-product line items; pos.etailpet.com direct JSON does not carry them
  - Fire-and-forget delivery model; eTailPet emails the xlsx report to the Four Dogs retailer inbox
  - Scope is trigger-only; parsing, persistence, and Kaylee tool enablement are downstream concerns
  - Same Vault credentials (CLIENT_ID/CLIENT_SECRET) reused from the transactions feature
depends_on:
  - fourdogs-etailpet-api-transactions
blocks: []
open_questions:
  - Confirm www legacy auth accepts existing Vault credentials on the report route (spike gates implementation)
  - Confirm sales-line email subject line for emailfetcher routing
  - Confirm xlsx column headers from first row of the report
updated_at: "2026-04-27T00:00:00Z"
---

# PRD: Sales — eTailPet Sales-Line Export Trigger

## Overview

The `sales` feature automates the delivery of eTailPet per-product sales data into the Four Dogs
data pipeline. It builds a scheduled worker that triggers eTailPet's legacy `www.etailpet.com`
email-export API to produce a `sales-line` xlsx report once daily. The report is emailed to the
Four Dogs retailer inbox, where the existing emailfetcher service receives and processes it.

### Why This Feature Exists

The `transactions` feature delivers daily order-level financial summaries (gross total, net, margin
per order) via `pos.etailpet.com`. Those summaries contain no per-product line items. Without
product-level sales history, Kaylee's recommendation engine is constrained to customer-total
spending patterns. SKU-level velocity data unlocks:

- Product-level reorder recommendations
- Seasonal and trend signals per SKU
- Margin analysis by product line
- Identification of slow-moving vs. fast-moving inventory

eTailPet's legacy API (`www.etailpet.com`) exposes a `report_type=sales-line` export that contains
this per-product detail. This is the only known path to line-item data without a direct JSON API.

---

## User Problem

**Four Dogs staff and the Kaylee agent lack product-level sales visibility.** Ordering decisions,
reorder recommendations, and inventory prioritization all depend on knowing which SKUs sold, in
what quantities, and at what price points. Today this information is either obtained manually from
the eTailPet portal or not obtained at all.

---

## Goals and Non-Goals

### Goals

- Automate daily delivery of eTailPet sales-line export to the emailfetcher inbox
- Make trigger outcomes observable and alertable via structured logs
- Lay the data foundation for Kaylee SKU-velocity tooling and ordering workflow enhancements
- Follow existing platform conventions (fourdogs-central repo, Helm + ArgoCD, ESO/Vault)

### Non-Goals

- Parsing or processing the xlsx/csv attachment (downstream emailfetcher and future feature)
- Persisting sales-line records to the Patroni DB (gated on post-spike schema confirmation)
- Kaylee tool changes or UI updates
- Changes to the existing `transactions` worker
- Catalog operations (`stores-catalog-export`, `catalog-update`)

---

## Functional Requirements

### FR-1: Scheduled sales-line export trigger

| ID | Requirement | Priority |
|---|---|---|
| FR-1.1 | Worker triggers `GET www.etailpet.com/{schema}/api/v1/transaction-report/?report_type=sales-line` on a daily schedule | Must |
| FR-1.2 | Trigger fires immediately at pod startup, then on a configurable `TRIGGER_INTERVAL_HOURS` schedule | Must |
| FR-1.3 | `TRIGGER_ENABLED=false` suppresses trigger without stopping the process | Must |
| FR-1.4 | `--dry-run` flag logs parameters without making API calls | Must |
| FR-1.5 | Date window defaults to a 7-day rolling lookback; configurable via `TRIGGER_LOOKBACK_DAYS` | Must |

### FR-2: www Legacy OAuth2 authentication

| ID | Requirement | Priority |
|---|---|---|
| FR-2.1 | Worker obtains a fresh Bearer token from `POST www.etailpet.com/{schema}/oauth2/token/` at the start of each trigger cycle | Must |
| FR-2.2 | Token request uses query-param form: `?grant_type=client_credentials&client_secret=...&client_id=...` | Must |
| FR-2.3 | Credentials injected from k8s Secret (ESO-managed); never hardcoded or logged | Must |
| FR-2.4 | Token obtained fresh per cycle; not cached across restarts | Must |

### FR-3: Retry and resilience

| ID | Requirement | Priority |
|---|---|---|
| FR-3.1 | Transient failures (5xx, 429) retried with exponential backoff: 3 attempts, initial 5s, max 60s | Must |
| FR-3.2 | Non-retryable 4xx errors (except 429) returned immediately without retry | Must |
| FR-3.3 | All 3 retry attempts exhausted → `trigger_exhausted` log event; operator intervention required | Must |
| FR-3.4 | `http.Client{Timeout: 30s}` applied to both token request and trigger GET | Must |

### FR-4: Observability

| ID | Requirement | Priority |
|---|---|---|
| FR-4.1 | Structured JSON logs via `log/slog` for all trigger lifecycle events | Must |
| FR-4.2 | Log events: `trigger_starting`, `trigger_invoked`, `trigger_success`, `trigger_failure`, `trigger_backoff`, `trigger_exhausted`, `trigger_shutdown`, `trigger_disabled`, `trigger_dry_run` | Must |
| FR-4.3 | `trigger_success` log includes `status_code` and `latency_ms` | Must |
| FR-4.4 | No credentials, tokens, or authorization headers logged | Must |

---

## Non-Functional Requirements

| NFR | Requirement |
|---|---|
| **Reliability** | Worker fires on schedule with < 1% missed runs over 30 days |
| **Security** | All credentials via ESO/Vault; no secrets in manifests, config files, or logs |
| **Deployment** | GitOps-managed via Helm + ArgoCD; same conventions as `fourdogs-etailpet-trigger` |
| **Independence** | No shared state, resources, or rate-limit contention with the `transactions` worker |
| **Operability** | Operator can enable/disable trigger via `TRIGGER_ENABLED` env var without redeploy; runbook written |
| **Compatibility** | Go binary in `fourdogs-central` repo; no new repositories |

---

## Success Metrics

| Metric | Target |
|---|---|
| Daily trigger success rate | ≥ 99% over 30 days |
| Missed scheduled triggers | < 1% |
| emailfetcher inbox delivery | Sales-line xlsx received within 10 minutes of trigger |
| Operator confidence | Runbook covers all known failure modes; `trigger_exhausted` is the only event requiring manual intervention |

---

## Stakeholders

| Stakeholder | Role | Interest |
|---|---|---|
| Todd Hintzmann | Lead / operator | Automated daily SKU-level sales data; observable trigger outcomes |
| Betsy | End user (ordering) | Per-product velocity and reorder signals surfaced in the ordering workflow |
| Kaylee agent | Downstream consumer | SKU-level sales history required for product-velocity recommendation tools |
| fourdogs-central emailfetcher | Downstream processor | Receives triggered report email; processes xlsx attachment |
| transactions feature | Predecessor / sibling | Vault credentials, `internal/etailpet/` client package, and worker patterns reused |

---

## Constraints

- No eTailPet sandbox environment — live production API only; at most one trigger call per day during validation
- `pos.etailpet.com` and `www.etailpet.com` are separate auth domains; separate token flows required
- The `pos` direct API confirmed has no sales-line or per-product report route (all sibling routes returned `404` in the 2026-04-22 API surface probe)
- Implementation is gated on Sprint 0 spike (`sales-0-01`) confirming `www` token validity on the report route

---

## Implementation Scope

See [tech-plan.md](tech-plan.md) for architecture, ADRs, and environment variables.
See [epics.md](epics.md) and [sprint-plan.md](sprint-plan.md) for delivery phasing.

**Spike gate:** All implementation is blocked on `sales-0-01` confirming:
1. `www` legacy token accepts existing Vault credentials on the report route
2. `GET /api/v1/transaction-report/?report_type=sales-line` returns `200 OK`
3. Sales-line email delivered to retailer inbox; subject line and attachment column headers documented
