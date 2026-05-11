---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
inputDocuments:
  - docs/terminus/watchdog/terminus-watchdog-agent/prd.md
  - docs/terminus/watchdog/terminus-watchdog-agent/ux-design.md
  - docs/terminus/watchdog/terminus-watchdog-agent/product-brief.md
  - docs/terminus/watchdog/terminus-watchdog-agent/research.md
  - docs/terminus/watchdog/terminus-watchdog-agent/brainstorm.md
  - docs/terminus/watchdog/terminus-watchdog-agent/businessplan-adversarial-review.md
workflowType: architecture
project_name: terminus-watchdog-agent
user_name: Todd Hintzmann
date: '2026-05-11'
---

# Architecture Decision Document — Terminus Watchdog Agent

_This document builds collaboratively through step-by-step discovery. Sections are appended as we work through each architectural decision together._

---

## Project Context Analysis

**Platform:** Terminus homelab — k3s (6 nodes), ArgoCD, Temporal, Vault, Proxmox VMs

**Agent type:** Event-correlation and remediation agent — not a webhook relay. Classification-first design (image-promotion vs. live-drift; zombie vs. stale).

**Functional scope (MVP1):**
- 11 detection patterns across 4 source families (ArgoCD REST API, Temporal gRPC, k8s Watch, Loki HTTP)
- 3 interactive remediation actions: ArgoCD sync, Temporal terminate, k8s pod delete
- Discord delivery to 2 channels; ED25519 interaction signature validation required
- Stateless detection; in-memory suppression + active-alert maps reconstructed on cold-start

**NFRs driving architecture:**
- Detection latency < 6 min (5-min poll + delivery)
- ≥ 99% uptime; single-container; k8s-native deployment to `terminus-watchdog` namespace
- Security-first (org constitution Article 9): signature validation, RBAC scoped to monitored namespaces, action logging, rate limiting
- Runbook required (org constitution Article 11)

**Dominant structural challenge:** Two independent concurrent runtimes sharing in-memory state:
1. Detection loop (polling + k8s Watch streams) — AsyncIO task group
2. Discord interaction HTTP server — handles Approve / Snooze / Investigate button presses

**Three unresolved open questions (from adversarial review H1/H2/H3) that architecture must answer:**
- **H1:** ArgoCD manifest diff API capability — does `/api/v1/applications/{name}/resource-tree` or `/manifests` allow image-tag-only vs. config-change classification?
- **H2:** Public ingress mechanism for Discord interaction endpoint + secret storage for bot token + interaction public key
- **H3:** External self-health observer for missed heartbeat detection

---

## Technology Stack

### Language & Toolchain

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | Python 3.12 | Established by PRD; asyncio-native; all required client SDKs available |
| Package manager | `uv` + `pyproject.toml` | Matches existing Terminus toolchain; fast lockfile, venv, script running |
| Linting / type checking | `ruff` + `mypy` | Standard Terminus Python toolchain |
| Testing | `pytest` + `pytest-asyncio` | TDD-first; async test support required for all coroutine code |

### Runtime & Concurrency Model

**Single asyncio event loop; `discord.py` 2.4+ as main runner.**

`discord.py` runs natively on asyncio. Background tasks (detection loop, heartbeat) are created as asyncio Tasks on `on_ready`. No threads. No subprocess workers.

```
asyncio event loop (main thread)
  ├── discord.py bot (Gateway websocket — persistent connection to Discord)
  │     ├── on_ready()           → creates background tasks
  │     └── on_interaction()     → Approve / Snooze / Investigate handlers
  ├── Task: detection_loop()     → 5-min poll cycle (ArgoCD, Temporal, k8s, Loki)
  │     ├── argocd_poller()
  │     ├── temporal_poller()
  │     ├── k8s_watch_stream()   → persistent Watch for pods / deployments / nodes
  │     └── loki_poller()
  └── Task: heartbeat_task()     → fires every 6 hours, posts to #platform-info
```

**H2 — Resolved:** Discord interaction delivery uses **Gateway mode (websocket)**. Button press interactions (Approve / Snooze / Investigate) arrive on the same Gateway connection as all other bot events. No public ingress required, no Cloudflare Tunnel, no separate HTTP server. ED25519 signature validation is handled transparently by the `discord.py` library.

### API Clients

| System | Client Library | Transport |
|--------|---------------|-----------|
| Discord | `discord.py` 2.4+ | Gateway websocket (persistent) |
| ArgoCD | `httpx` 0.27+ (async) | REST over HTTPS to `argocd.trantor.internal` |
| Temporal | `temporalio` 1.x (official SDK) | gRPC |
| Kubernetes | `kubernetes-asyncio` | k8s Watch API + resource queries |
| Loki | `httpx` 0.27+ (async) | HTTP LogQL query API |

### Secrets Management

Vault + External Secrets Operator (consistent with existing Terminus platform pattern). ESO syncs Vault secrets → k8s Secrets in the `terminus-watchdog` namespace. The bot process reads secrets from environment variables populated by the k8s Secret.

| Secret | Vault Path | k8s Secret Key |
|--------|-----------|----------------|
| Discord bot token | `terminus/watchdog/discord-bot-token` | `DISCORD_BOT_TOKEN` |
| ArgoCD API token | `terminus/watchdog/argocd-token` | `ARGOCD_TOKEN` |
| Temporal mTLS cert | `terminus/watchdog/temporal-cert` | `TEMPORAL_CERT_PEM` |
| Temporal mTLS key | `terminus/watchdog/temporal-key` | `TEMPORAL_KEY_PEM` |

### Deployment

New standalone repo: **`terminus.watchdog`** (independent Python service; own CI/CD; separate from Go-based platform services). Deployed via **Helm chart** to `terminus-watchdog` namespace in the k3s cluster. ArgoCD manages deployment from the `terminus.infra` App-of-Apps pattern.

---

## Core Architectural Decisions

### Decision 1: Data Architecture — No Persistent Storage

**Decision:** No database. All detection state is in-memory and reconstructed on cold-start.

**Rationale:** Detection is idempotent. Missing alerts during a restart window (< 60s on k8s rolling restart) is an acceptable trade-off vs. the operational complexity of a database dependency. The 5-minute poll cycle guarantees re-detection on restart.

**In-memory state:**

```python
@dataclass
class SuppressEntry:
    pattern: str
    resource_key: str        # "{namespace}/{name}"
    suppressed_until: datetime
    suppressed_by: str       # Discord user ID

@dataclass
class ActiveAlert:
    alert_id: str            # UUID
    pattern: str
    resource_key: str
    severity: str            # "high" | "medium"
    channel_message_id: int  # Discord message ID for in-place edits
    created_at: datetime
    remediation_pending: bool

# Both protected by asyncio.Lock
suppress_map: dict[str, SuppressEntry]   # key = suppression_key
active_alerts: dict[str, ActiveAlert]    # key = alert_id
```

**Suppression key format:** `"{pattern}:{namespace}/{resource_name}"` — e.g. `"argocd-live-drift:argocd/fourdogs-central"`

### Decision 2: ArgoCD Diff Strategy (H1 — Resolved)

**Decision:** Option A — parse inline diff from the ArgoCD application object; fall back to conservative collapse if diff is unavailable.

**Mechanism:**

```
GET /api/v1/applications/{name}
  └── response.status.sync.status          → "OutOfSync" trigger
  └── response.status.summary.images[]     → current deployed images
  └── response.status.operationState       → last operation result
  └── response.status.resources[]          → per-resource sync status + diffs
```

**Classification logic:**
1. If `status.sync.status == "OutOfSync"`:
   - Extract changed resources from `status.resources[*]` where `status != "Synced"`
   - If ALL changed resources are container image fields only (`image:` change, no other fields) → classify as `argocd-image-promotion` (Medium)
   - Otherwise → classify as `argocd-live-drift` (High)
2. If diff parsing fails or returns empty → fallback to `argocd-live-drift` (High) — conservative

**Validation gate:** MVP0 sprint must include a test that exercises the classification branch against a real ArgoCD app diff payload. If the `status.resources` field does not carry enough detail for classification, the fallback path becomes permanent and the `argocd-image-promotion` pattern is deferred to MVP2.

### Decision 3: RBAC & Security

**k8s ServiceAccount (`watchdog-agent`):**

| Resource | Verbs | Scope |
|----------|-------|-------|
| pods, deployments, nodes | get, list, watch | ClusterRole (read-only) |
| pods (delete) | delete | Role — `terminus-watchdog` namespace only |

**ArgoCD permissions:**

| Action | Permission |
|--------|-----------|
| Read application status | ArgoCD RBAC policy: `p, role:watchdog, applications, get, */*, allow` |
| Sync application | ArgoCD RBAC policy: `p, role:watchdog, applications, sync, */*, allow` — **write action; requires Discord Approve** |

**Write action safety:**
- All write actions (ArgoCD sync, Temporal terminate, pod delete) require Discord button confirmation
- Confirmed action is logged to structured JSON: `{"ts": ..., "action": ..., "resource": ..., "approved_by": ..., "discord_user_id": ...}`
- No action executes without a correlated alert entry in `active_alerts`

**Secret hygiene:**
- No secrets in source control — all via Vault + ESO
- `DISCORD_BOT_TOKEN` never logged; masked in all log outputs
- Token rotation procedure documented in `docs/runbook.md`

### Decision 4: Self-Health Observer (H3 — Resolved)

**Decision:** Option D — Prometheus + Alertmanager with a watchdog alerting rule.

**Mechanism:**
- Watchdog exports `watchdog_last_heartbeat_timestamp_seconds` gauge (Prometheus format) on internal port `9090/metrics`
- Gauge is updated by `heartbeat_task()` every 6 hours (same task that posts the heartbeat Discord message)
- Alertmanager rule fires if gauge is absent OR stale > 6 hours:

```yaml
- alert: WatchdogHeartbeatMissed
  expr: absent(watchdog_last_heartbeat_timestamp_seconds) or (time() - watchdog_last_heartbeat_timestamp_seconds > 21600)
  for: 10m
  labels:
    severity: critical
  annotations:
    summary: "Terminus Watchdog heartbeat missed"
    description: "The watchdog has not posted a heartbeat in over 6 hours"
```

- Alert routes to Discord via the existing Alertmanager Discord webhook receiver (separate from the watchdog bot — avoids circular dependency)
- k8s liveness probe (HTTP GET `/healthz`) restarts the pod if the event loop is blocked > 30s

### Decision 5: Internal HTTP Server (health + metrics)

A minimal `aiohttp` server runs as an asyncio Task alongside the bot, serving:
- `GET /healthz` → 200 if event loop is responsive (k8s liveness probe)
- `GET /metrics` → Prometheus text format (scraped by Prometheus `ServiceMonitor`)

Port `9090` (internal only; no Ingress). This is the only HTTP server in the process — it does NOT handle Discord interactions.

---

## Implementation Patterns

### Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Python modules / files | snake_case | `argocd_poller.py` |
| Classes | PascalCase | `ArgoCDPoller` |
| Functions / methods | snake_case | `detect_live_drift()` |
| Constants | SCREAMING_SNAKE_CASE | `POLL_INTERVAL_SECONDS` |
| k8s resources | kebab-case | `watchdog-agent` ServiceAccount |
| Helm values keys | camelCase | `replicaCount`, `image.tag` |
| Alert pattern IDs | kebab-case | `argocd-live-drift` |
| Suppression keys | `"{pattern}:{namespace}/{name}"` | `"argocd-live-drift:argocd/fourdogs-central"` |

### Detection Pattern Interface

Every detector implements the `BaseDetector` ABC:

```python
class BaseDetector(ABC):
    @abstractmethod
    async def detect(self) -> list[Alert]:
        """Run one detection cycle. Returns list of new Alerts (empty if none)."""
        ...

    @property
    @abstractmethod
    def pattern_id(self) -> str:
        """Canonical pattern identifier, e.g. 'argocd-live-drift'"""
        ...
```

Detectors are **stateless** — they receive a read-only view of `suppress_map` and `active_alerts` on each call but never mutate state directly. State mutations happen only in `detection_loop()` after acquiring `asyncio.Lock`.

### Alert Model

```python
@dataclass(frozen=True)
class Alert:
    pattern: str              # e.g. "argocd-live-drift"
    severity: str             # "high" | "medium"
    title: str                # short title for Discord embed
    body: str                 # markdown body for Discord embed
    resource_namespace: str
    resource_name: str
    remediation_available: bool
    remediation_hint: str     # e.g. "Approve to sync fourdogs-central"

    @property
    def suppression_key(self) -> str:
        return f"{self.pattern}:{self.resource_namespace}/{self.resource_name}"
```

### Error Handling & Logging

- Structured logging via Python `logging` + `python-json-logger` (JSON output to stdout — scraped by Loki)
- Every detector wraps its poll cycle in a `try/except Exception` — a single failing detector never crashes the loop
- Failed detector logs `level=error, detector=X, exc_info=true` and skips to next cycle
- Discord send failures: 3 retries with exponential back-off; after 3 failures, write to stderr and continue
- External client timeout: all httpx/gRPC calls use explicit timeout (default 10s configurable via `DETECTION_TIMEOUT_SECONDS`)

### Testing Conventions

- Test file mirrors source: `tests/unit/detectors/test_argocd.py` → `watchdog/detectors/argocd.py`
- All external clients are mocked at the client wrapper layer (never mock httpx internals)
- Each detector has: unit test for each alert path + unit test for suppression path + unit test for error path
- Integration tests (`tests/integration/`) may exercise live cluster resources — gated behind `RUN_INTEGRATION_TESTS=1`
- Acceptance tests (`tests/acceptance/`) correspond 1:1 with PRD acceptance criteria — BDD naming: `test_ac_<n>_<short_description>`

### Configuration

All tunable parameters sourced from environment variables (populated from ConfigMap + k8s Secret). No hardcoded config in source:

```
DISCORD_BOT_TOKEN          (secret)
DISCORD_ALERTS_CHANNEL_ID
DISCORD_INFO_CHANNEL_ID
ARGOCD_URL
ARGOCD_TOKEN               (secret)
TEMPORAL_HOST
TEMPORAL_CERT_PEM          (secret)
TEMPORAL_KEY_PEM           (secret)
POLL_INTERVAL_SECONDS      default: 300
HEARTBEAT_INTERVAL_SECONDS default: 21600
DETECTION_TIMEOUT_SECONDS  default: 10
```

---

## Project Structure

Source code lives in a new standalone repository: **`terminus.watchdog`** (to be created in the `electricm0nk` GitHub org, cloned into `TargetProjects/terminus/watchdog/`).

```
terminus.watchdog/
├── pyproject.toml                    # uv-managed; all dependencies declared here
├── uv.lock
├── Dockerfile                        # multi-stage build; python:3.12-slim base
├── helm/
│   └── terminus-watchdog/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           ├── serviceaccount.yaml
│           ├── role.yaml             # pod delete in terminus-watchdog ns
│           ├── clusterrole.yaml      # read-only across monitored namespaces
│           ├── rolebinding.yaml
│           ├── clusterrolebinding.yaml
│           ├── configmap.yaml        # non-secret configuration
│           ├── externalsecret.yaml   # ESO → Vault sync
│           ├── service.yaml          # ClusterIP; ports 9090 (metrics/health)
│           └── servicemonitor.yaml   # Prometheus scrape
├── watchdog/
│   ├── __init__.py
│   ├── main.py                       # entrypoint; bot setup + task orchestration
│   ├── config.py                     # Settings dataclass; reads from env vars
│   ├── state.py                      # SuppressMap, ActiveAlertMap; asyncio.Lock
│   ├── models.py                     # Alert, ActiveAlert, SuppressEntry dataclasses
│   ├── health.py                     # aiohttp /healthz + /metrics Task
│   ├── clients/
│   │   ├── __init__.py
│   │   ├── argocd.py                 # httpx async ArgoCD REST client wrapper
│   │   ├── temporal.py               # temporalio client wrapper
│   │   ├── kubernetes.py             # kubernetes-asyncio wrapper
│   │   └── loki.py                   # httpx async Loki LogQL client
│   ├── detectors/
│   │   ├── __init__.py
│   │   ├── base.py                   # BaseDetector ABC
│   │   ├── argocd.py                 # argocd-live-drift, argocd-image-promotion
│   │   ├── temporal.py               # temporal-zombie-workflow, temporal-stale-workflow
│   │   ├── kubernetes.py             # node-not-ready, pod-crashloop, deployment-unavailable, etc.
│   │   └── loki.py                   # loki-error-spike, loki-panic-detected
│   ├── actions/
│   │   ├── __init__.py
│   │   ├── base.py                   # BaseAction ABC; action logging
│   │   ├── argocd_sync.py            # POST /api/v1/applications/{name}/sync
│   │   ├── temporal_terminate.py     # terminate workflow execution
│   │   └── pod_delete.py             # kubernetes-asyncio pod delete
│   └── discord/
│       ├── __init__.py
│       ├── bot.py                    # bot factory; on_ready; background task creation
│       ├── alerts.py                 # Alert → discord.Embed formatter; channel routing
│       └── interactions.py           # discord.ui.View; Approve/Snooze/Investigate handlers
├── tests/
│   ├── conftest.py                   # shared fixtures; mock client factories
│   ├── unit/
│   │   ├── detectors/
│   │   │   ├── test_argocd.py
│   │   │   ├── test_temporal.py
│   │   │   ├── test_kubernetes.py
│   │   │   └── test_loki.py
│   │   ├── actions/
│   │   │   ├── test_argocd_sync.py
│   │   │   ├── test_temporal_terminate.py
│   │   │   └── test_pod_delete.py
│   │   └── discord/
│   │       ├── test_alerts.py
│   │       └── test_interactions.py
│   ├── integration/
│   │   └── test_detection_loop.py    # requires RUN_INTEGRATION_TESTS=1
│   └── acceptance/
│       └── test_acceptance_criteria.py  # 1:1 with PRD ACs; BDD naming
├── docs/
│   └── runbook.md                    # required by Org Constitution Article 11
└── .github/
    └── workflows/
        ├── ci.yml                    # ruff + mypy + pytest on every PR
        └── release.yml               # build + push image on tag push
```

### Component Responsibility Map

| Epic → Component | Source Location |
|-----------------|----------------|
| E1: Detection patterns (all 11) | `watchdog/detectors/` |
| E2: Discord delivery | `watchdog/discord/alerts.py`, `watchdog/discord/bot.py` |
| E3: Interactive remediation (3 actions) | `watchdog/actions/`, `watchdog/discord/interactions.py` |
| E4: Suppression / snooze | `watchdog/state.py` + interaction handlers |
| E5: Heartbeat | `watchdog/main.py` (heartbeat_task) + `watchdog/discord/alerts.py` |
| E6: Self-health | `watchdog/health.py` + Helm `servicemonitor.yaml` + Alertmanager rule (in `terminus.infra`) |
| E7: Configuration | `watchdog/config.py` + Helm `configmap.yaml` + `externalsecret.yaml` |

---

## Architecture Validation

### Requirements Coverage

| PRD Requirement | Architecture Coverage | Status |
|-----------------|-----------------------|--------|
| 11 detection patterns across 4 sources | `watchdog/detectors/{argocd,temporal,kubernetes,loki}.py` | ✅ |
| Discord delivery to 2 channels | `watchdog/discord/alerts.py` — severity → channel routing | ✅ |
| 3 write actions with confirmation | `watchdog/actions/` + `watchdog/discord/interactions.py` Approve handler | ✅ |
| Detection latency < 6 min | 5-min `POLL_INTERVAL_SECONDS` + asyncio-native delivery | ✅ |
| ED25519 signature validation | Discord Gateway mode — handled by `discord.py` library | ✅ (H2 resolved) |
| In-memory suppression map | `watchdog/state.py` SuppressMap + asyncio.Lock | ✅ |
| Heartbeat to #platform-info every 6h | `heartbeat_task()` in `main.py` | ✅ |
| External missed-heartbeat detection | Prometheus gauge + Alertmanager rule (H3 resolved) | ✅ |
| Single container; k8s-native deployment | Helm chart → `terminus-watchdog` namespace | ✅ |
| TDD (Org Article 7) | `tests/unit/` + `pytest` — mirrored structure, all paths covered | ✅ |
| BDD acceptance criteria (Org Article 8) | `tests/acceptance/test_acceptance_criteria.py` 1:1 with PRD ACs | ✅ |
| Secrets never in source (Org Article 9) | Vault + ESO; token masked in logs | ✅ |
| Runbook required (Org Article 11) | `docs/runbook.md` | ✅ |
| Agent entry point parity (Org Article 12) | Single entrypoint `python -m watchdog.main`; same command in Docker + Helm | ✅ |

### H1 / H2 / H3 Resolution Summary

| Finding | Resolution | Residual Risk |
|---------|-----------|---------------|
| **H1:** ArgoCD diff classification | Option A: parse `status.resources[]` inline diff from app object. MVP0 validates before committing. Fallback: collapse to `argocd-live-drift` | MVP0 validation may force permanent fallback → `argocd-image-promotion` deferred to MVP2. Low risk — detection still works. |
| **H2:** Discord interaction ingress | Gateway mode eliminates public ingress requirement. Discord interactions arrive on persistent websocket. | None — architectural choice removes the risk entirely. |
| **H3:** Missed heartbeat detection | Prometheus `watchdog_last_heartbeat_timestamp_seconds` gauge + Alertmanager WatchdogHeartbeatMissed rule → Discord via Alertmanager webhook receiver | Depends on Prometheus/Alertmanager remaining healthy. Prometheus already runs on Terminus — acceptable shared dependency. |

### Open Technical Items (deferred to story-level)

| Item | Owner Epic | Notes |
|------|-----------|-------|
| Temporal mTLS cert provisioning | E7 (Config) | Need to verify Temporal cluster cert path and whether it is already in Vault |
| Alertmanager WatchdogHeartbeatMissed rule location | E6 (Self-health) | Rule lives in `terminus.infra` not `terminus.watchdog` — cross-repo dependency; story must call this out |
| ArgoCD `watchdog` RBAC policy + token creation | E7 (Config) | Provisioning story required in `terminus.infra` |
| k3s cluster DNS name for ArgoCD API | E1 (Detection) | Assumed `argocd.trantor.internal` — verify during MVP0 |
| Loki query endpoint URL | E1 (Detection) | Assumed `loki.monitoring.svc.cluster.local` — verify during MVP0 |

---

## Architecture Summary

**Terminus Watchdog Agent** is a single-process Python 3.12 service deployed as a Kubernetes Deployment in the `terminus-watchdog` namespace. It runs a single `asyncio` event loop hosting two concurrent workloads:

1. A **`discord.py` 2.4+ Gateway bot** — maintains a persistent websocket to Discord, sends alert embeds to `#platform-alerts` and `#platform-info`, and handles button interactions (Approve / Snooze / Investigate) without any public ingress.

2. A **5-minute detection loop** — polls ArgoCD REST, Temporal gRPC, k8s Watch, and Loki HTTP in parallel; classifies events against 11 defined patterns; deduplicates via in-memory suppression map.

All three open questions from the adversarial review are resolved:
- **H1** (ArgoCD classification): inline diff parse from `status.resources[]`, MVP0-validated, with conservative fallback
- **H2** (Discord ingress): eliminated by Gateway mode — no public URL, no certificate, no Cloudflare Tunnel
- **H3** (Self-health): Prometheus gauge + Alertmanager rule delivers missed-heartbeat alerts independently of the watchdog itself

Secrets are managed via Vault + External Secrets Operator. Write actions require Discord confirmation. Runbook is required (`docs/runbook.md`). TDD and BDD acceptance tests are structured to mirror the PRD's 12 acceptance criteria.

