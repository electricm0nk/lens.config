# Terminus Watchdog Agent — Research Summary

**Feature:** `terminus-watchdog-agent`  
**Phase:** PrePlan  
**Date:** 2026-05-11  
**Researcher:** Lens Agent

---

## Research Questions

Four open questions surfaced during brainstorming required external research before committing to architecture:

1. **Temporal Visibility API** — watch vs. poll for zombie workflow detection
2. **ArgoCD event stream vs. poll** — how to detect OutOfSync/degraded state changes
3. **Discord bot interactions** — new or existing; button patterns for approve/skip/investigate
4. **Kubernetes Watch API** — resource events vs. polling for pod/deployment anomalies

---

## Q1: Temporal Visibility API — Zombie Workflow Detection

### Findings

**Source:** `docs.temporal.io/visibility`, `docs.temporal.io/references/sdk-metrics`

Temporal does **not** provide a push/stream API for workflow state changes. Detection must be **poll-based**.

**Key capabilities:**

- **`ListWorkflowExecutions`** — SQL-like filter API with `ExecutionStatus`, `WorkflowType`, `StartTime`, `CloseTime`, and custom Search Attributes
- **Example query for stuck workflows:**
  ```sql
  ExecutionStatus = 'Running' AND WorkflowType = 'ReleaseWorkflow' AND StartTime < '2026-05-11T00:00:00Z'
  ```
- **Attempt count** — not a native Visibility field; must be derived from workflow history via `GetWorkflowExecutionHistory` (expensive). For MVP1, use elapsed time threshold as zombie proxy.
- **SDK metrics** available: `temporal_workflow_failed` (Counter), `temporal_activity_execution_failed` (Counter), `temporal_workflow_task_execution_failed` (Counter). These require a metrics scraping path (Prometheus), not a query path.
- Requires Temporal Server v1.20+ with SQL backend (MySQL 8.0.17+ or PostgreSQL 12+). Our k3s cluster uses PostgreSQL — confirmed compatible.

### Architecture Decision

**Primary detection: poll `ListWorkflowExecutions` every 5 minutes.**

- Filter: `ExecutionStatus = 'Running' AND StartTime < now() - threshold`
- Threshold: configurable per workflow type (e.g. ReleaseWorkflow = 30min, default = 60min)
- No attempt count check in MVP1 — elapsed time is sufficient proxy
- Stateless: each poll is independent; no state store required

**No watch stream exists** — poll is the only viable path.

---

## Q2: ArgoCD State Detection — OutOfSync and Health

### Findings

**Source:** `argo-cd.readthedocs.io/operator-manual/notifications/`, `argo-cd.readthedocs.io/developer-guide/api-docs/`

**Two paths available:**

### Path A: ArgoCD REST API Polling

- Endpoint: `GET /api/v1/applications` (returns all apps with sync+health status)
- Auth: Bearer token (ArgoCD API token)
- No native watch/streaming — must poll
- Returns `syncStatus.status` (`Synced`, `OutOfSync`) and `health.status` (`Healthy`, `Degraded`, `Progressing`, `Missing`, `Unknown`)
- Low complexity, no ArgoCD config changes required

### Path B: ArgoCD Notifications Webhook (Push)

- ArgoCD Notifications is an installable controller that sends events to external endpoints
- Built-in triggers: `on-sync-failed`, `on-sync-status-unknown`, `on-health-degraded`, `on-deployed`
- Watchdog exposes an HTTP endpoint; ArgoCD notifies on state change
- Reduces latency (push vs. poll delay) but requires ArgoCD Notifications controller install + configuration
- Adds operational dependency: if webhook delivery fails, watchdog misses events

### Architecture Decision

**MVP1: Poll `GET /api/v1/applications` every 5 minutes.**  
**MVP2: Add ArgoCD Notifications webhook as primary, retain poll as fallback.**

Rationale: MVP1 keeps the watchdog self-contained with no external config changes. Webhook reduces latency and is the preferred production model, but adds installation ceremony better suited to MVP2.

---

## Q3: Discord Bot — Buttons and Interaction Handling

### Findings

**Source:** `docs.discord.com/developers/components/reference`

**Discord supports interactive button messages via the Message Components system.**

### Key concepts:

- **Action Row** (type=1): top-level container, holds up to 5 buttons
- **Button** (type=2): clickable element with styles: Primary(1), Secondary(2), Success(3), Danger(4), Link(5)
  - Must have `custom_id` (string, 1-100 chars) — returned in interaction payload when clicked
  - `custom_id` encodes action intent (e.g. `approve:release:prod`, `skip:release:prod`, `investigate:argocd:kaylee`)
- When a user clicks a button, Discord sends an **Interaction** to the bot's registered interaction endpoint (HTTP POST)
- Bot must respond within 3 seconds (acknowledge or reply)

### Approve/Skip/Investigate pattern:

```json
{
  "content": "🚨 ArgoCD: kaylee-agent is OutOfSync (15min)\n**Action required**",
  "components": [{
    "type": 1,
    "components": [
      {"type": 2, "style": 3, "label": "✅ Approve Resync", "custom_id": "approve:argocd:kaylee-agent"},
      {"type": 2, "style": 2, "label": "⏭ Skip (10min)", "custom_id": "skip:argocd:kaylee-agent:10"},
      {"type": 2, "style": 4, "label": "🔍 Investigate", "custom_id": "investigate:argocd:kaylee-agent"}
    ]
  }]
}
```

### Bot options: new vs. existing

- **Existing FourDogs bot** (if one exists) — reuse for MVP1 to avoid provisioning
- **New dedicated bot** — cleaner separation, no shared token risk; preferred for production
- Watchdog needs: Bot token (send messages), webhook endpoint (receive interactions), channel ID target

### Architecture Decision

**MVP1: New dedicated Discord bot** (`terminus-watchdog`) with:
- Bot token for sending alert messages with buttons
- Interaction endpoint URL registered in Discord portal → routes to watchdog HTTP server
- `custom_id` encoding: `action:domain:resource[:param]` pattern
- After button click: update original message (disable buttons, show resolution) via `PATCH /webhooks/{app_id}/{token}/messages/@original`

---

## Q4: Kubernetes Watch API

### Findings

**Source:** `kubernetes.io/docs/reference/using-api/api-concepts/#efficient-detection-of-changes`

**Kubernetes provides a native watch/stream mechanism — `?watch=true`.**

- **Pattern**: Initial `LIST` to get current state + `resourceVersion`, then `GET ?watch=1&resourceVersion=<rv>` for live stream
- **Event types**: `ADDED`, `MODIFIED`, `DELETED` + synthetic `BOOKMARK` for progress marks
- **Reconnection**: On `410 Gone` (history expired), restart with fresh list
- **`sendInitialEvents=true`** (beta, k8s 1.32+): Combines initial state + watch stream in one request
- Python client (`kubernetes` library): `watch.Watch()` wraps this pattern with automatic reconnect
- **Resources to watch:** `pods` (CrashLoopBackOff, OOMKilled), `deployments` (unavailable replicas), `nodes` (NotReady)

### Architecture Decision

**Use Kubernetes watch streams for pod/deployment anomaly detection.**

- Pod watch: filter for `status.containerStatuses[*].state.waiting.reason in [CrashLoopBackOff, OOMKilled]`
- Deployment watch: detect `status.unavailableReplicas > 0` sustained for > threshold
- Node watch: detect `conditions[Ready].status = False`
- Use `kubernetes` Python SDK (`watch.Watch()`) which handles reconnect and `resourceVersion` tracking automatically
- Scope: watch all namespaces or configurable namespace list (MVP1: all)

**Watch is clearly superior to polling for k8s** — event-driven, low latency, supported by SDK.

---

## Synthesis: Recommended Detection Architecture

| Signal Source | Method | Interval/Trigger | Priority |
|---|---|---|---|
| Temporal zombie workflows | Poll `ListWorkflowExecutions` | 5 min | High |
| ArgoCD OutOfSync/Degraded | Poll `GET /api/v1/applications` | 5 min (MVP1); webhook (MVP2) | High |
| K8s pod CrashLoopBackOff | Watch stream | Immediate on event | High |
| K8s deployment unavailable | Watch stream | Immediate on event | Medium |
| K8s node NotReady | Watch stream | Immediate on event | Medium |

**Common patterns:**
- All detectors are stateless — no persistence layer needed for MVP1
- Alert deduplication: in-memory suppression map (resource → last_alerted_at); configurable cooldown (default 30min)
- Output adapter pattern: `AlertDispatcher` interface → Discord implementation (MVP1), extensible to PagerDuty/Slack (MVP2)

---

## Open Items / Risks

1. **Constitution conflict**: `terminus/watchdog` constitution restricts `permitted_tracks: [express, quickdev, hotfix-express, spike]` — excludes `full`. Must update before adversarial gate.
2. **Temporal connection**: Watchdog needs Temporal gRPC endpoint + namespace. Confirm `temporal.terminus.svc.cluster.local:7233` is accessible from proposed deployment namespace.
3. **ArgoCD API token**: Must provision a read-only ArgoCD API token. Confirm token scoping supports `applications/list` only.
4. **Discord bot provisioning**: New bot registration in Discord Developer Portal required. Token stored in Vault or k8s Secret.
5. **K8s RBAC**: Watchdog ServiceAccount needs `get`, `list`, `watch` on `pods`, `deployments`, `nodes`, `events` across namespaces.
6. **Alert fatigue risk**: CrashLoopBackOff events can be high-frequency during rolling deploys. Need backoff / suppression strategy before first production alert.

---

## Next Steps

→ `product-brief.md` — synthesize brainstorm + research into product brief (vision, users, scope, non-goals, success criteria)  
→ Constitution update — add `full` to `permitted_tracks` in `terminus/watchdog/constitution.md`  
→ Adversarial review gate
