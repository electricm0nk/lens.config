---
feature: sales
doc_type: architecture
status: draft
goal: "Design and deploy a Go worker that triggers the eTailPet www.etailpet.com legacy email-export API daily to deliver per-product sales-line data to the Four Dogs emailfetcher pipeline."
key_decisions:
  - New binary cmd/etailpet-sales-trigger in fourdogs-central; clean separation from pos-domain transactions worker
  - New LegacyClient in internal/etailpet/ for www query-param OAuth2; distinct from existing PosClient
  - time.Ticker daily scheduler; fire-and-forget trigger model; no delivery confirmation polling
  - ESO ExternalSecret in namespace fourdogs-etailpet-sales-trigger; same Vault path as transactions (confirmed shared credentials)
  - Separate Helm chart and ArgoCD Application; same fourdogs project conventions
  - Spike-first gate on sales-0-01 before any Sprint 1 story starts
depends_on:
  - fourdogs-etailpet-api-transactions
blocks: []
open_questions:
  - TO_EMAIL param behavior on www trigger — default retailer address vs. explicit param (spike deliverable)
  - Email subject line format from eTailPet (spike deliverable; required for emailfetcher routing)
  - Xlsx column headers — first row of sales-line report (spike deliverable)
  - eTailPet server-side deduplication on overlapping 7-day windows (accepted risk per businessplan review)
  - Staging/production inbox isolation — shared eTailPet credentials mean staging triggers hit production inbox
updated_at: "2026-04-27T00:00:00Z"
---

# Architecture: fourdogs-etailpet-api-sales

> See [prd.md](prd.md) for requirements, success metrics, and feature scope.
> See [tech-plan.md](tech-plan.md) for detailed ADRs and environment variable reference.

## Context

The `transactions` feature delivers daily order-level summaries from `pos.etailpet.com`. That API
returns no per-product line items. SKU-level sales velocity — needed by Kaylee's recommendation
engine — is only available via the `www.etailpet.com` legacy email-trigger path with
`report_type=sales-line`. This architecture defines how the trigger worker is built and deployed.

The two eTailPet domains are separate auth scopes: `pos` uses JSON-body OAuth2;
`www` uses query-param OAuth2. The same Vault CLIENT_ID/CLIENT_SECRET works for both
(confirmed 2026-04-22 auth probe).

---

## System Context Diagram

```
┌─────────────────────────────────────────────────────────────┐
│  fourdogs-central (k8s, fourdogs-etailpet-sales-trigger ns) │
│                                                             │
│  ┌─────────────────────────────┐                           │
│  │ cmd/etailpet-sales-trigger  │                           │
│  │                             │                           │
│  │  time.Ticker (24h default)  │                           │
│  │        │                   │                           │
│  │  LegacyClient               │─── POST www.etailpet.com ──→ [eTailPet]
│  │   1. www token request      │    /oauth2/token/         │       │
│  │   2. trigger GET            │─── GET  www.etailpet.com ──→     │
│  │                             │    /api/v1/transaction-   │       │
│  │  ESO-injected secrets       │    report/?report_type=   │       │
│  │  CLIENT_ID, CLIENT_SECRET   │    sales-line …           │       ↓
│  └─────────────────────────────┘                           │  [eTailPet email server]
│                                                             │       │
│  ┌────────────────────────────┐                            │       │ xlsx email
│  │ Vault (ClusterSecretStore) │                            │       ↓
│  │ secret/terminus/fourdogs/  │                            │  [Four Dogs retailer inbox]
│  │ etailpet-api               │                            │       │
│  └────────────────────────────┘                            │       ↓
│                                                             │  [emailfetcher]  ← out of scope
└─────────────────────────────────────────────────────────────┘
```

---

## Component Design

### `internal/etailpet/LegacyClient`

New type alongside the existing `PosClient`. Responsibility boundary: www OAuth2 token
acquisition and `transaction-report` trigger GET only.

```go
type LegacyClient struct {
    BaseURL    string        // https://www.etailpet.com
    Schema     string        // tenant schema path segment (e.g. fourdogspetsupplies)
    ClientID   string
    ClientSecret string
    HTTPClient *http.Client  // injected; Timeout: 30s
    Logger     *slog.Logger
}

func (c *LegacyClient) TriggerSalesReport(ctx context.Context, params SalesTriggerParams) error
// SalesTriggerParams: StartDate, EndDate (YYYY-MM-DD), ReportType, FileFormat
```

**Token flow** (query-param OAuth2, distinct from PosClient's JSON-body flow):

```
POST https://www.etailpet.com/{schema}/oauth2/token/
  ?grant_type=client_credentials
  &client_secret={CLIENT_SECRET}
  &client_id={CLIENT_ID}
→ {"access_token": "...", "token_type": "Bearer", "expires_in": 36000}
```

Token is obtained fresh on each trigger cycle. Not cached across restarts.

**Trigger GET:**

```
GET https://www.etailpet.com/{schema}/api/v1/transaction-report/
  ?report_type=sales-line
  &file_format=xlsx
  &start_date=YYYY-MM-DD
  &end_date=YYYY-MM-DD
Authorization: Bearer {access_token}
→ 200 OK  (email queued by eTailPet server-side)
```

Date format: `YYYY-MM-DD`. This differs from the `pos` domain (`MM/DD/YYYY`). Date adapter
is internal to `LegacyClient`.

**Retry policy:** 3 attempts; exponential backoff 5s/10s/20s (capped at 60s); 5xx and 429
are retryable; 4xx (except 429) are not.

---

### `cmd/etailpet-sales-trigger`

```
main.go
  └── Config struct (env vars via os.Getenv + required-check at startup)
  └── LegacyClient init (inject http.Client{Timeout: 30s})
  └── probeServer := &ProbeServer{port: ":8080"}     ← goroutine; see Probe Server below
  └── go probeServer.ListenAndServe()
  └── time.Ticker loop  (single goroutine; see Concurrent Safety below)
        ├── TRIGGER_ENABLED=false → log trigger_disabled, skip
        ├── --dry-run flag → log trigger_dry_run, skip
        └── LegacyClient.TriggerSalesReport(ctx, params)
              ├── success → probeServer.SetReady(true); log trigger_success
              └── exhausted → log trigger_exhausted; continue to next tick
  └── signal.NotifyContext(ctx, syscall.SIGTERM, syscall.SIGINT)
        └── graceful shutdown → log trigger_shutdown
```

Startup validation: if any required env var (`ETAILPET_CLIENT_ID`, `ETAILPET_CLIENT_SECRET`,
`ETAILPET_SCHEMA_NAME`) is empty, exit 1 before first tick.

**Probe HTTP server:**

`ProbeServer` is a minimal `net/http.Server` running on port `:8080` in a separate goroutine.

| Endpoint | Behaviour |
|---|---|
| `GET /healthz` | Always `200 OK` — confirms the process is alive |
| `GET /readyz` | `503 Service Unavailable` until the first `trigger_success`; `200 OK` after |

Readiness state is held in an `atomic.Bool` (`ready`), set to `true` on the first successful
trigger. The probe goroutine is started before the ticker loop and is never restarted.
The `Service` ClusterIP in the Helm chart exposes port `8080` to the probe endpoints.

**Concurrent trigger safety:**

The ticker loop runs in a single goroutine. Each tick is processed to completion (including all
retry backoff) before the next tick is consumed from the channel. Because `time.Ticker` buffers
one tick, a slow trigger cycle may cause one tick to queue but never more than one. The trigger
loop uses a blocking `select` on `ticker.C` — no concurrent `TriggerSalesReport` calls can occur.
No mutex is required; the single-goroutine model is the concurrency control.

---

### Helm Chart: `fourdogs-etailpet-sales-trigger`

New chart under `deploy/helm/fourdogs-etailpet-sales-trigger/` in `fourdogs-central`.

| Resource | Notes |
|---|---|
| `Deployment` | Single replica; `replicas: 1` |
| `ServiceAccount` | Namespace-scoped |
| `ExternalSecret` | `fourdogs-etailpet-api-sales-secrets`; references `ClusterSecretStore`; Vault path `secret/terminus/fourdogs/etailpet-api` |
| `Service` (internal) | ClusterIP; port `8080` for liveness/readiness probe HTTP endpoint |
| Liveness probe | `GET /healthz :8080` — always 200; confirms process alive |
| Readiness probe | `GET /readyz :8080` — 200 after first successful trigger; 503 before |
| Sync-wave | ArgoCD `sync-wave: "1"` on the Deployment; ExternalSecret at `sync-wave: "0"` to ensure Secret is populated before pod starts |

The chart **does not share** a namespace, ServiceAccount, or ExternalSecret with the
transactions worker. Independence is required (see ADR-1 in tech-plan.md).

---

### ArgoCD Application

New `Application` manifest in the `fourdogs` ArgoCD project:

```yaml
spec:
  source:
    repoURL: https://github.com/electricm0nk/fourdogs-central.git
    targetRevision: main
    path: deploy/helm/fourdogs-etailpet-sales-trigger
  destination:
    server: https://kubernetes.default.svc
    namespace: fourdogs-etailpet-sales-trigger
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## Data Flow

```
[time.Ticker fires]
        ↓
[LegacyClient: POST www.etailpet.com oauth2/token/]  → access_token
        ↓
[LegacyClient: GET www.etailpet.com transaction-report?report_type=sales-line]
        ↓
[eTailPet internal: queue email with xlsx attachment]
        ↓
[Four Dogs retailer inbox receives email]
        ↓
[emailfetcher processes attachment]  ← out of scope for this feature
```

**No data is persisted by the trigger worker.** It is stateless between ticks.

---

## Security

| Concern | Control |
|---|---|
| Credentials at rest | Vault `secret/terminus/fourdogs/etailpet-api`; ESO injects as k8s Secret |
| Credentials in logs | `LegacyClient` never logs CLIENT_ID, CLIENT_SECRET, or Authorization headers |
| Transport | TLS enforced on all outbound calls to `www.etailpet.com` |
| RBAC | Namespace-scoped ServiceAccount; no cluster-level permissions |
| Secret rotation | Vault path rotation propagates via ESO refresh interval without redeploy |

---

## Observability

All log events use `log/slog` JSON output. See `ux-design.md` for the full operator runbook.

| Event | Fields |
|---|---|
| `trigger_starting` | `interval_hours`, `lookback_days` |
| `trigger_invoked` | `url`, `date_from`, `date_to` |
| `trigger_success` | `status_code`, `latency_ms` |
| `trigger_failure` | `status_code`, `error`, `retry_count` |
| `trigger_backoff` | `attempt`, `delay_seconds` |
| `trigger_exhausted` | `error` |
| `trigger_shutdown` | — |
| `trigger_disabled` | — |
| `trigger_dry_run` | `url`, `date_from`, `date_to` |

No metrics scraping in v1. Loki log-based alerting on `trigger_exhausted` is the
operator's primary alert surface.

---

## Deployment Sequence

1. `sales-0-01` spike: live confirm www auth + email delivery
2. Sprint 1: `LegacyClient` impl + unit tests + CI wiring
3. Sprint 2: Helm chart + ESO ExternalSecret + ArgoCD Application
4. Sprint 2: Staging validation at `TRIGGER_INTERVAL_HOURS=1`; confirm email routing
5. Sprint 2: Production deploy at `TRIGGER_INTERVAL_HOURS=24`

---

## Scope Boundaries

| In scope | Out of scope |
|---|---|
| www.etailpet.com token + trigger | pos.etailpet.com (transactions feature) |
| cmd/etailpet-sales-trigger binary | Parsing or storing xlsx attachment |
| LegacyClient in internal/etailpet/ | emailfetcher changes |
| Helm chart + ArgoCD Application | Kaylee SKU-velocity tools |
| ESO ExternalSecret | DB schema or migrations |
| Structured logging + probes | Email delivery monitoring (deferred, accepted risk H2) |

---

## Constraints and Accepted Risks

- **Fire-and-forget gap (H2):** HTTP 200 from www does not confirm email delivery. Email
  non-delivery is undetected in v1. Operator performs periodic inbox spot-checks. (Accepted,
  businessplan adversarial review 2026-04-27.)
- **No fallback if spike fails (H1):** If `sales-0-01` fails, feature is paused; eTailPet
  support inquiry raised before any Sprint 1 work. (Accepted, businessplan adversarial review
  2026-04-27.)
- **Shared inbox in staging:** No staging-isolated eTailPet schema available; staging triggers
  reach the production retailer inbox. Rate discipline (max 1 live trigger/day) applies in
  both environments.
- **eTailPet deduplication unknown:** 7-day rolling lookback may produce overlapping windows.
  Downstream emailfetcher must be resilient to duplicate report emails.
