---
feature: sales
doc_type: ux-design
status: draft
goal: "Document operator and developer experience for the etailpet-sales-trigger worker — no end-user UI surface exists."
must_reference: prd
updated_at: "2026-04-27T00:00:00Z"
---

# UX Design: Sales Trigger — Operator Experience

## Scope

This document covers the operator and developer experience for `cmd/etailpet-sales-trigger`. The
service has no end-user UI. The relevant "users" are:

| User | Context |
|---|---|
| **Operator** (Todd) | Deploys, configures, monitors, and troubleshoots the trigger via k8s, ArgoCD, and logs |
| **Developer** | Runs locally with `--dry-run`; inspects logs; writes unit tests against the mock transport |
| **Downstream pipeline** (emailfetcher) | Passively consumes the xlsx report delivered to the retailer inbox |

See [prd.md](prd.md) for requirements.

---

## Operator Workflows

### 1. Normal daily operation

**Expectation:** Worker fires automatically on pod startup, then every `TRIGGER_INTERVAL_HOURS` (default: 24h). No operator action required.

**How to verify:** Tail logs from the pod.

```bash
kubectl logs -n fourdogs-etailpet-sales-trigger deploy/etailpet-sales-trigger -f
```

**Healthy log pattern:**

```json
{"time":"2026-04-27T06:00:00Z","level":"INFO","msg":"trigger_starting","interval_hours":24,"lookback_days":7}
{"time":"2026-04-27T06:00:01Z","level":"INFO","msg":"trigger_invoked","url":"https://www.etailpet.com/...","date_from":"2026-04-20","date_to":"2026-04-27"}
{"time":"2026-04-27T06:00:02Z","level":"INFO","msg":"trigger_success","status_code":200,"latency_ms":342}
```

---

### 2. Disabling the trigger without redeploy

Set `TRIGGER_ENABLED=false` in the Helm values (or via ArgoCD parameter override). The process stays
running (liveness probe passes) but emits `trigger_disabled` on each tick and does not call the
eTailPet API.

**When to use:** During eTailPet maintenance windows or when the emailfetcher is offline.

---

### 3. Staging validation (one-shot)

To validate in staging without waiting 24 hours, set `TRIGGER_INTERVAL_HOURS=1`. Do **not** set
intervals below `1h` — undocumented rate limit on the `www` API; maximum one live trigger per day
during steady-state operation.

**Validation steps:**

1. Deploy to staging namespace with `TRIGGER_INTERVAL_HOURS=1`
2. Wait for first `trigger_success` log (≤ 5 minutes)
3. Confirm retailer inbox receives a `sales-line` email within 10 minutes
4. Confirm emailfetcher routing rule matches the email subject
5. Confirm xlsx attachment is present and the first row contains expected column headers

---

### 4. Investigating a trigger failure

**`trigger_failure` log event:** Transient error during token fetch or trigger GET. Retry is
in progress. No action required unless it escalates to `trigger_exhausted`.

**`trigger_exhausted` log event:** All 3 retry attempts failed. Operator action required.

Runbook steps:
1. Check pod logs for the error chain: `trigger_backoff` events will precede `trigger_exhausted`
2. Verify the `fourdogs-etailpet-api-sales-secrets` k8s Secret exists and is populated
3. Verify the Vault path `secret/terminus/fourdogs/etailpet-api` contains `CLIENT_ID` and `CLIENT_SECRET`
4. Attempt a manual `--dry-run` pod (see below) to confirm env vars are injected
5. If credentials are valid, re-enable the trigger; it will retry on the next `TRIGGER_INTERVAL_HOURS` tick

---

### 5. Manual dry-run pod

Run a one-shot pod with `--dry-run` to verify env vars and connectivity without calling eTailPet:

```bash
kubectl run sales-trigger-dryrun \
  --image=<image> \
  --rm -it \
  --env=TRIGGER_ENABLED=true \
  --env=CLIENT_ID=<from-secret> \
  --env=CLIENT_SECRET=<from-secret> \
  --env=TENANT_SCHEMA=fourpaws \
  -- /etailpet-sales-trigger --dry-run
```

**Expected output:**

```json
{"level":"INFO","msg":"trigger_dry_run","url":"https://www.etailpet.com/...","date_from":"...","date_to":"..."}
```

---

## Developer Workflows

### Local development

```bash
# Build
go build ./cmd/etailpet-sales-trigger/...

# Dry run (no API calls)
CLIENT_ID=test CLIENT_SECRET=test TENANT_SCHEMA=fourpaws \
  ./etailpet-sales-trigger --dry-run

# Unit tests
go test ./internal/etailpet/... ./cmd/etailpet-sales-trigger/...
```

### Running tests with mock transport

Unit tests in `internal/etailpet/legacy_client_test.go` inject a mock `http.RoundTripper`. No
credentials or live network access required. See story `sales-1-03`.

---

## Configuration Reference

All configuration is via environment variables injected by ESO.

| Variable | Required | Default | Description |
|---|---|---|---|
| `CLIENT_ID` | Yes | — | eTailPet OAuth2 client ID (from Vault) |
| `CLIENT_SECRET` | Yes | — | eTailPet OAuth2 client secret (from Vault) |
| `TENANT_SCHEMA` | Yes | — | eTailPet tenant schema path segment (e.g., `fourpaws`) |
| `TRIGGER_ENABLED` | No | `true` | `false` disables trigger without stopping process |
| `TRIGGER_INTERVAL_HOURS` | No | `24` | Hours between trigger cycles |
| `TRIGGER_LOOKBACK_DAYS` | No | `7` | Rolling lookback window for report date range |
| `LOG_LEVEL` | No | `info` | `debug` enables verbose request tracing (no secrets) |

---

## Health and Observability

**Liveness probe:** HTTP GET `/healthz` — returns `200 OK` always (process is alive)

**Readiness probe:** HTTP GET `/readyz` — returns `200 OK` when at least one successful trigger has
completed; `503` before first success to prevent premature traffic routing

**Metrics (future):** No Prometheus scraping in v1. Log-based alerting via Loki for `trigger_exhausted`.

---

## Downstream UX: emailfetcher

The emailfetcher receives the triggered report as an email from eTailPet. No emailfetcher code
changes are in scope for the `sales` feature, but the operator should verify:

- Sales-line email arrives from eTailPet with the expected subject line (confirmed in spike `sales-0-01`)
- emailfetcher routing rule matches that subject and routes to the correct processing queue
- xlsx attachment column headers match the schema confirmed in the spike

This is a validation step, not a deliverable of the `sales` feature.
