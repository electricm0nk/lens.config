---
feature: terminus-watchdog-agent
doc_type: prd
status: draft
goal: "Automated platform health monitor that detects and classifies operational events across ArgoCD, Temporal, and Kubernetes, delivers pre-diagnosed Discord alerts with operator context, and executes approved remediation actions (sync, terminate, pod delete) on operator confirmation."
stepsCompleted: [step-01-init, step-02-discovery, step-02b-vision, step-03-requirements]
vision:
  core: "Replace reactive cognitive load with pre-diagnosed, classified alerts delivered before the operator has to go looking"
  differentiator: "Classification-first — image-promotion vs live-drift, zombie vs stale — not raw event forwarding"
  alert-posture: quiet-hours (7am–10pm CT @mention for High; argocd-order-day-unsync always-on)
  delight-moment: "First zombie catch — knowing what was wrong in 10 seconds, not 30 minutes"
  scope-decision: "MVP1 includes real write actions (Approve = sync/terminate/pod-delete). Not read-only."
classification:
  projectType: event-correlation-and-remediation-agent
  domain: platform-engineering/incident-detection
  complexity: very-high
  projectContext: brownfield-platform-mixed-greenfield-deliverables
  externalDependencies:
    - discord-bot-new: terminus-watchdog (new registration required)
    - argocd-api-token: not yet provisioned
    - temporal-grpc-access: not yet verified
    - public-ingress: mechanism not yet selected (required for Discord interaction endpoint)
inputDocuments:
  - docs/terminus/watchdog/terminus-watchdog-agent/product-brief.md
  - docs/terminus/watchdog/terminus-watchdog-agent/research.md
  - docs/terminus/watchdog/terminus-watchdog-agent/brainstorm.md
  - docs/terminus/watchdog/terminus-watchdog-agent/preplan-adversarial-review.md
  - docs/terminus/watchdog/terminus-watchdog-agent/ux-design.md
key_decisions:
  - "MVP1 includes real Approve actions: argocd app sync, Temporal workflow terminate, kubectl delete pod"
  - "Alert posture: quiet hours 7am–10pm CT for High @mention; argocd-order-day-unsync always-on"
  - "CrashLoopBackOff: alert on first crash; escalate to High if >5 restarts in 10 min"
  - "Skip uses Discord Select Menu with 10/30/120 min snooze options"
  - "Cold-start 3-minute suppression window on startup"
  - "Heartbeat: every 6 hours to #platform-alerts"
  - "Two channels: #platform-alerts (High + heartbeat), #platform-info (Medium/Informational)"
open_questions:
  - "ArgoCD manifest diff API validation against running instance — required before TechPlan (preplan C1)"
  - "Public ingress mechanism for Discord interaction endpoint — required before TechPlan (preplan H1)"
  - "Temporal gRPC cross-namespace access verification — required before TechPlan (preplan H3)"
  - "Loki HTTP API availability and auth for temporal-postgres-connectivity detection"
depends_on: []
blocks: []
updated_at: '2026-05-11T00:00:00Z'
---

# Product Requirements Document — Terminus Watchdog Agent

**Author:** Todd Hintzmann
**Date:** 2026-05-11

---

## 1. Problem Statement

The Terminus homelab platform — k3s, ArgoCD, Temporal, Vault, and multiple application services across Proxmox VMs — has no automated health monitoring. Platform issues are caught manually. Two failure classes have been observed in production with significant time cost:

1. **ArgoCD OutOfSync with no classification.** Image-promotion syncs and genuine live-drift both show identical OutOfSync status. Distinguishing them requires manual inspection.
2. **Temporal zombie workflows.** A `SeedSecrets` activity ran at attempt 3,297 before being caught. A second instance ran similarly for `fourdogs-etailpet-sales-trigger`. Temporal retries indefinitely with no alerting on duration.

The operator (Todd) spends reactive diagnostic time on problems that should have been surfaced automatically with pre-diagnosis attached.

---

## 2. Scope Decision — MVP1 Includes Write Actions

The original product brief stated "Read-only, always. MVP1 has no write operations." This constraint is **revised for MVP1** based on operator preference.

**MVP1 includes interactive remediation actions:**
- `Approve` on ArgoCD alerts → triggers `argocd app sync` via ArgoCD REST API
- `Approve` on Temporal zombie alerts → terminates the workflow via Temporal SDK
- `Approve` on K8s CrashLoopBackOff alerts → deletes the pod via k8s API

All actions require explicit operator button press in Discord. No autonomous action is ever taken. Every action is logged.

Patterns with no safe automated action (`k8s-node-notready`, `k8s-deployment-unavailable`, `temporal-postgres-connectivity`) do not expose an `Approve` button.

---

## 3. Users

| Role | User | Need |
|------|------|------|
| Platform Operator | Todd | Pre-diagnosed alerts that minimize triage time; one-click remediation for known-safe actions |
| (MVP2) Platform Agents | Terminus-scoped agents | Programmatic signal consumption via output adapter |

---

## 4. Detection Requirements

### 4.1 Detection Pattern Catalog

| Pattern ID | Source | Severity | Detection Method | Threshold |
|------------|--------|---------|-----------------|-----------|
| `argocd-image-promotion` | ArgoCD REST API | Informational | Poll `/api/v1/applications` every 5 min; classify OutOfSync as image-promotion if diff shows only image tag change | OutOfSync + image-tag-only diff |
| `argocd-live-drift` | ArgoCD REST API | High | Poll `/api/v1/applications`; classify OutOfSync as live-drift if diff includes non-image-tag config change | OutOfSync + config-change diff |
| `argocd-stuck-sync` | ArgoCD REST API | Medium | Poll; detect `Progressing` health status exceeding threshold | > 5 min in Progressing |
| `argocd-order-day-unsync` | ArgoCD REST API + ConfigMap | High | OutOfSync in any fourdogs namespace app during a configured high-priority window | OutOfSync during window |
| `temporal-zombie-activity` | Temporal Visibility API | Medium | Poll `ListWorkflowExecutions` every 5 min; filter running workflows by elapsed time | Running > 2 hours |
| `temporal-zombie-critical` | Temporal Visibility API | High | Same poll; higher elapsed time threshold | Running > 24 hours |
| `temporal-stale-workflow` | Temporal Visibility API | Medium | Running workflow with no new history events in the stale window | No events in > 30 min |
| `temporal-postgres-connectivity` | Loki HTTP API | High | Poll Loki for Temporal worker log errors matching postgres connection failure pattern | Any match in last 5 min |
| `k8s-crashloopbackoff` | k8s Watch stream | High | Watch pod events; detect `CrashLoopBackOff` or rapid restart count | See §4.2 |
| `k8s-deployment-unavailable` | k8s Watch stream | Medium | Watch deployment events; detect 0 available replicas sustained | 0 available > 5 min |
| `k8s-node-notready` | k8s Watch stream | High | Watch node events; detect `NotReady` condition | NotReady > 2 min |

### 4.2 CrashLoopBackOff Detection and Escalation Policy

1. **First detection:** Pod enters `CrashLoopBackOff` state. Alert fires if the pod does not recover (reach `Running` + `Ready`) within 2 minutes.
2. **Oscillation escalation:** If the same pod accumulates > 5 restarts within a 10-minute window, the severity escalates to High and a new alert fires regardless of cooldown.
3. **Cooldown after first alert:** 30-minute suppression window per pod resource, except for escalation events.
4. **Recovery:** When the pod reaches `Running` + `Ready` state and holds it for 2 minutes, the active alert message is updated to Resolved.

### 4.3 ArgoCD Classification Dependency

**Note (open question):** The `argocd-image-promotion` vs `argocd-live-drift` classification requires access to manifest diff content via the ArgoCD REST API (`/api/v1/applications/{name}/manifests` or equivalent). This API capability must be validated against the running ArgoCD instance before TechPlan. If manifest diff is not available without git access, both patterns collapse into a single `argocd-out-of-sync` alert at High severity.

The architecture must document the chosen diff strategy.

### 4.4 Temporal Detection Notes

Elapsed time is used as the zombie proxy. Attempt count is not a native Temporal Visibility field and requires per-workflow history polling — this approach is excluded from MVP1 due to cost. The elapsed-time thresholds are:
- `temporal-zombie-activity`: running > 2 hours
- `temporal-zombie-critical`: running > 24 hours
- `temporal-stale-workflow`: running with no history events > 30 minutes

Per-workflow-type threshold overrides must be configurable via ConfigMap (e.g., `SeedSecrets` has a known-problematic history → lower threshold).

### 4.5 `temporal-postgres-connectivity` Loki Dependency

This pattern requires access to Temporal worker pod logs via the Loki HTTP API. If Loki is unreachable or the query auth is not resolvable in TechPlan, this pattern is deferred to MVP2. The architecture must document the Loki query path, auth method, and polling frequency.

---

## 5. Alert Delivery Requirements

### 5.1 Discord Channel Architecture

| Channel | Patterns | @mention |
|---------|---------|---------|
| `#platform-alerts` | High severity + watchdog heartbeat | Yes — within quiet hours |
| `#platform-info` | Medium + Informational | No |

### 5.2 Alert Posture — Quiet Hours

- **Quiet hours window:** 10:00 PM CT to 7:00 AM CT
- **During quiet hours:** High severity alert messages post to `#platform-alerts` but do not @mention. The unread indicator is the only notification.
- **Exception — always-on patterns:** `argocd-order-day-unsync` fires with @mention at any hour when a high-priority window is active. This pattern is explicitly exempt from quiet hours because order-day windows are pre-planned.
- **All other patterns:** Respect quiet hours without exception in MVP1.

### 5.3 Alert Message Requirements

Every alert message must include:

- Severity-coded embed color (High=`#FF4444`, Medium=`#FF8C00`, Informational=`#4A90E2`)
- Emoji prefix matching severity
- Plain-language event description (2 sentences max)
- Structured fields: Resource, Duration, Diagnosis (classified cause), Recommended action
- Footer: `{pattern-id} · Detected {local-time CT} ({UTC})`
- Fully scannable in under 90 seconds

### 5.4 Alert Deduplication

- In-memory suppression map: `{resource_key} → suppressed_until`
- Default cooldown: 30 minutes per resource
- `argocd-order-day-unsync` bypasses cooldown during an active high-priority window
- CrashLoopBackOff oscillation escalation bypasses cooldown (see §4.2)
- Cold-start: suppression map pre-populated during 3-minute startup grace period (no alerts sent during grace window)

---

## 6. Interaction Requirements

### 6.1 Approve Actions

| Pattern | Approve Action | API / SDK |
|---------|---------------|-----------|
| `argocd-*` (OutOfSync patterns) | `argocd app sync {app-name}` | ArgoCD REST: `POST /api/v1/applications/{name}/sync` |
| `temporal-zombie-*`, `temporal-stale-workflow` | Terminate workflow | Temporal SDK: `TerminateWorkflow(id, reason="watchdog-operator-approve")` |
| `k8s-crashloopbackoff` | Delete pod | k8s API: `DELETE /api/v1/namespaces/{ns}/pods/{name}` |
| `k8s-node-notready` | None — no `Approve` button | — |
| `k8s-deployment-unavailable` | None — no `Approve` button | — |
| `temporal-postgres-connectivity` | None — no `Approve` button | — |

**Pre-action re-verification:** Before executing any Approve action, the watchdog must re-query the source system to confirm the condition is still active. If the condition has already resolved, the action is skipped and the operator is notified via message update.

**Action safety requirements (for architecture):**
- Signature validation on every Discord interaction payload (ED25519)
- Rate limit: max 3 Approve actions per resource per 5-minute window
- All actions logged with: timestamp, operator Discord user ID, resource, action taken, result
- Idempotent: if the sync/terminate/delete was already triggered, re-execution is a no-op or returns success

### 6.2 Skip / Snooze

- Implemented as a Discord Select Menu (not a button) with options: 10 min / 30 min / 120 min
- Snooze updates the in-memory suppression map for the resource
- Original message updates to show snooze expiry time
- Alert re-fires after snooze window if condition is still active

### 6.3 Investigate

- Sends an ephemeral reply (visible only to presser) containing deep-links to the relevant system UI
- ArgoCD: `https://argocd.terminus.internal/applications/{app-name}`
- Temporal: `https://temporal.terminus.internal/namespaces/{ns}/workflows/{id}/{run-id}`
- K8s: kubectl command hint or k9s command
- Original message is not updated

### 6.4 Message Lifecycle Updates

All state transitions update the original Discord message in-place:

1. **Active:** Full alert embed, buttons enabled
2. **Acknowledged:** Buttons disabled; Resolution field appended (action, operator, timestamp, outcome)
3. **Snoozed:** Buttons remain visible but reduced; embed footer shows "Snoozed until {time}"
4. **Resolved:** Embed color changes to `#00C851`; Resolved field appended; buttons disabled
5. **Action failed:** Resolution field appended with error detail; buttons re-enabled for retry

The watchdog maintains an active alert map: `{resource_key} → (message_id, channel_id)` for all in-place updates.

---

## 7. Watchdog Self-Health Requirements

### 7.1 Heartbeat

- Posts to `#platform-alerts` every 6 hours
- Content: uptime, last cycle timestamp, next cycle timestamp, alert summary (sent / suppressed / snoozed)
- No @mention for heartbeat messages

### 7.2 Missed Heartbeat

- An external health observer (architecture to define: k8s liveness probe exporter, dead man's switch, or separate lightweight probe) must detect when the heartbeat stops and send a `#platform-alerts` message with @mention
- This is a self-health design requirement that the architecture must resolve. The watchdog cannot reliably detect its own absence.

### 7.3 Cold-Start Suppression

- On startup, enter a 3-minute grace window
- During the grace window: run all detection cycles and populate the suppression map with current conditions; do not send any alerts
- After the grace window: only conditions not present at startup generate new alerts
- On startup: post a cold-start notice to `#platform-alerts`:
  - "Watchdog starting up — 3-minute grace window active. Alerts resume at {time}."
- Grace window is not configurable in MVP1

### 7.4 Source System Unavailability

If a detection source (ArgoCD API, Temporal, Loki, k8s) is unreachable during a poll cycle:
- Log the failure internally
- Do not fire alerts based on stale or missing data
- After 3 consecutive failed poll cycles for a source, post a single warning to `#platform-info`: "{source} polling unavailable — watchdog detection degraded"
- Resume normal alerting when source recovers

---

## 8. Configuration Requirements

### 8.1 Detection Thresholds ConfigMap

The following values must be configurable via a `watchdog-config` ConfigMap in the watchdog namespace without code changes:

```yaml
detection:
  temporal_zombie_activity_threshold_minutes: 120   # default: 2 hours
  temporal_zombie_critical_threshold_minutes: 1440  # default: 24 hours
  temporal_stale_no_events_threshold_minutes: 30
  temporal_per_workflow_overrides:
    SeedSecrets: 30                                 # lower threshold for known-bad pattern
  argocd_stuck_sync_threshold_minutes: 5
  k8s_crashloopbackoff_recovery_grace_seconds: 120
  k8s_crashloopbackoff_escalation_restarts: 5
  k8s_crashloopbackoff_escalation_window_minutes: 10
  k8s_node_notready_threshold_minutes: 2
  k8s_deployment_unavailable_threshold_minutes: 5
  default_cooldown_minutes: 30
  cold_start_grace_minutes: 3
  heartbeat_interval_hours: 6
  source_unavailable_alert_after_failures: 3
```

### 8.2 High-Priority Windows ConfigMap

High-priority windows define periods when `argocd-order-day-unsync` and other always-on behaviors are active:

```yaml
high_priority_windows:
  - name: "fourdogs-order-day"
    schedule: "FREQ=WEEKLY;BYDAY=SU;BYHOUR=9;BYMINUTE=0;BYSECOND=0"
    duration_hours: 6
    affected_namespaces:
      - "fourdogs"
      - "fourdogs-etailpet-trigger"
    behaviors:
      - bypass_cooldown: true
      - always_mention: true
      - patterns:
          - argocd-order-day-unsync
```

### 8.3 Maintenance Windows ConfigMap

```yaml
maintenance_windows:
  - name: "infra-maintenance"
    start: "2026-05-15T02:00:00Z"
    end: "2026-05-15T06:00:00Z"
    affected_namespaces: ["*"]
    suppress_alerts: false   # false = alert but add maintenance badge; true = suppress and summarize after
```

### 8.4 Quiet Hours ConfigMap

```yaml
alert_posture:
  quiet_hours:
    enabled: true
    start_local: "22:00"     # 10pm CT
    end_local: "07:00"       # 7am CT
    timezone: "America/Chicago"
    mention_suppression: true  # suppress @mention only; alert still posts
    always_on_patterns:        # bypass quiet hours for these patterns
      - argocd-order-day-unsync
```

---

## 9. Non-Functional Requirements

| Category | Requirement |
|----------|-------------|
| **Detection latency** | < 6 minutes from anomaly to Discord alert (worst case: 5-min poll + delivery) |
| **Availability** | ≥ 99% uptime (k8s Deployment, single replica, self-healing) |
| **Stateless detection** | Every detection cycle fetches fresh from source. No cached state drives detection decisions. In-memory suppression map is the only state, and it is reconstructed from scratch on restart (cold-start grace window handles this). |
| **No dependency on monitored systems for self-health** | The watchdog must not depend on ArgoCD, Temporal, or k8s for its own alerting delivery path. Discord is the only delivery dependency. |
| **RBAC: minimum required permissions** | ServiceAccount: `get`, `list`, `watch` on pods, deployments, nodes (cluster-wide read); ArgoCD token: `applications/list`, `applications/get`, `applications/sync` (read + sync only); Temporal: workflow visibility + terminate (no admin); k8s pod delete: scoped to monitored namespaces only |
| **Action security** | Discord payload signature validation (ED25519) required before any action; all actions logged; rate limit enforced |
| **Configuration** | All thresholds and windows configurable via ConfigMap; no code changes required for threshold updates |
| **Observability** | Structured JSON log output; Prometheus metrics for: alerts sent, actions executed, detection cycle duration, source poll failures |
| **Deployability** | Single container image, deployed via Helm chart or Kustomize manifest to `terminus-watchdog` namespace |

---

## 10. Out of Scope (MVP1)

| Item | Deferral |
|------|---------|
| Autonomous remediation (no human approval) | Never |
| Secret leak detection | MVP2 — requires Vault API research |
| ArgoCD Notifications webhook (push-based) | MVP2 — MVP1 uses polling |
| Agent-to-agent signaling | MVP2 |
| Historical trending / dashboards | Not planned |
| Multi-user approval flows | Not planned |
| `Approve` button for `k8s-node-notready` | Not planned — no safe automated node action |
| Attempt-count zombie detection | MVP2 — requires per-workflow history polling |
| Custom runbook authoring | MVP2 |
| `secret-shared-detection` | MVP2 — Vault API research required |
| Alert posture customization per pattern | MVP2 |

---

## 11. Success Criteria

| Metric | Target | Measurement |
|--------|--------|-------------|
| Time to first alert after anomaly | < 6 minutes | Detection cycle timestamp vs. anomaly start timestamp in logs |
| False positive rate | < 5% of alerts over first 30 days | Operator-tagged "false positive" button (or manual log) |
| Zombie workflows surfaced within 30 minutes of crossing threshold | 100% | Retrospective: compare Temporal history to alert log |
| ArgoCD live-drift alerts during order-day windows | 100% | Retrospective: compare ArgoCD event history to alert log |
| Operator diagnostic time per alert | < 2 min (baseline: ~15–30 min for complex incidents) | Informal operator log for first 30 days |
| Watchdog uptime | ≥ 99% | k8s pod restart history over 30 days |
| Actions executed successfully | ≥ 95% of Approve presses result in confirmed action | Action outcome log |

**Baseline (current state, informal):** Operator diagnostic time for complex incidents is approximately 15–30 minutes. The `SeedSecrets` zombie ran for an estimated 4+ days before manual discovery.

---

## 12. Acceptance Criteria

### AC-1: ArgoCD OutOfSync Classification
- **Given** an app is OutOfSync because CI pushed a new image tag
- **When** the watchdog polls and classifies it
- **Then** a `argocd-image-promotion` Informational alert is posted to `#platform-info` within 6 minutes, with Diagnosis field stating the classification reason
- **(Dependent on** manifest diff API validation — see open questions)

### AC-2: Temporal Zombie Detection
- **Given** a workflow has been running for > 24 hours
- **When** the watchdog polls Temporal Visibility API
- **Then** a `temporal-zombie-critical` High alert is posted to `#platform-alerts` with `@Todd` mention within 6 minutes (if within quiet hours), or without mention (if outside quiet hours)

### AC-3: Approve ArgoCD Sync
- **Given** an active `argocd-live-drift` alert message exists in Discord
- **When** the operator presses `Approve`
- **Then** the watchdog re-verifies the OutOfSync condition is still active, calls `POST /api/v1/applications/{name}/sync`, updates the Discord message with resolution (buttons disabled, Resolution field shows outcome + timestamp), and logs the action

### AC-4: Approve Temporal Workflow Terminate
- **Given** an active `temporal-zombie-critical` alert message exists
- **When** the operator presses `Approve`
- **Then** the watchdog terminates the workflow via Temporal SDK with reason `"watchdog-operator-approve"`, updates the Discord message with resolution, and logs the action

### AC-5: Approve Pod Delete
- **Given** an active `k8s-crashloopbackoff` alert message exists
- **When** the operator presses `Approve`
- **Then** the watchdog deletes the named pod via k8s API, updates the Discord message, and logs the action

### AC-6: Skip Snooze
- **Given** any active alert message with a Skip select menu
- **When** the operator selects "Snooze 30 minutes"
- **Then** the alert is suppressed for 30 minutes, the message updates to show snooze expiry time, and the alert re-fires after 30 minutes if the condition persists

### AC-7: Quiet Hours Behavior
- **Given** the current time is 11:00 PM CT (outside quiet hours)
- **When** a `k8s-node-notready` High alert fires
- **Then** the alert message posts to `#platform-alerts` without @mention

### AC-8: Cold-Start Suppression
- **Given** the watchdog pod restarts while `k8s-crashloopbackoff` is already active for a known pod
- **When** the watchdog starts and completes its 3-minute grace window
- **Then** no new alert fires for the pre-existing condition; the condition is in the suppression map; a cold-start notice was posted at startup

### AC-9: Alert Content Scannable in 90 Seconds
- **Given** any alert embed
- **When** a reviewer reads it cold
- **Then** they can identify: what it is, which resource, how long, likely cause, and recommended action — within 90 seconds, without clicking any links

### AC-10: Heartbeat
- **Given** the watchdog has been running for 6 hours
- **When** the heartbeat timer fires
- **Then** a heartbeat embed posts to `#platform-alerts` with: last cycle time, next cycle time, and cycle summary (alerts sent / suppressed / snoozed count)

### AC-11: CrashLoopBackOff Escalation
- **Given** a pod that crashed once (first alert fired) continues crashing
- **When** it accumulates > 5 restarts within a 10-minute window
- **Then** a new High-severity alert fires regardless of the 30-minute cooldown

### AC-12: Discord Signature Validation
- **Given** an inbound Discord interaction payload
- **When** the signature validation fails (wrong key, expired, or tampered)
- **Then** the request is rejected with HTTP 401; no action is taken; the failure is logged

---

## 13. MVP0 Slice (First 2-Week Sprint)

To reduce risk and validate the core detection + delivery pipeline early, the first deployable slice is:

| Pattern | Rationale |
|---------|-----------|
| `k8s-crashloopbackoff` | Uses k8s Watch API (no external token needed); directly observed in production |
| `temporal-zombie-critical` | Highest-value pattern; validated against known incidents |
| `argocd-live-drift` | Core differentiator; validates ArgoCD API token + diff strategy |

Approve actions for all three are included in MVP0. Discord bot, interaction endpoint, and public ingress must be operational for MVP0.

---

## 14. Risks and Open Items

| # | Risk / Open Item | Severity | Action |
|---|---|---|---|
| 1 | **ArgoCD manifest diff API** — classification depends on unvalidated API capability | Blocking pre-TechPlan | Validate `/api/v1/applications/{name}/manifests` against running ArgoCD before architecture |
| 2 | **Public ingress for Discord interaction endpoint** — mechanism not selected | Blocking pre-TechPlan | Select Cloudflare Tunnel, existing wildcard cert, or other; document in architecture |
| 3 | **Temporal gRPC cross-namespace access** — not yet verified from proposed watchdog namespace | High | Verify from test pod before architecture commit |
| 4 | **Loki HTTP API availability** — required for `temporal-postgres-connectivity` | Medium | Confirm Loki endpoint + auth in TechPlan; drop to MVP2 if unavailable |
| 5 | **Interaction endpoint security** — Discord payload signature validation implementation correctness is critical; bug = unauthenticated prod actions | High | Architecture must specify signature validation library/implementation; unit test required |
| 6 | **Write action blast radius** — if watchdog pod is compromised, it has pod-delete + workflow-terminate permissions | High | Scope k8s RBAC to monitored namespaces only; document security model in architecture |
| 7 | **Alert fatigue during rolling deploys** — CrashLoopBackOff events can be high-frequency | Medium | Cooldown policy defined in §4.2; configurable thresholds in §8.1 |
| 8 | **Watchdog self-health observer** — missed heartbeat requires external mechanism | Medium | Architecture must define the external health observer design |

