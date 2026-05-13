---
feature: terminus-watchdog-agent
doc_type: ux-design
status: draft
goal: "Design the Discord-native operator UX for the Terminus Watchdog Agent — alert message format, interaction flows, severity visual system, channel architecture, and action handling."
key_decisions:
  - Two-channel Discord structure (High → #platform-alerts, Medium/Info → #platform-info)
  - "@mention Todd for High severity only"
  - "Real actions in MVP1: Approve (sync/terminate/delete), Skip (dropdown snooze), Investigate (ephemeral deep-link)"
  - "Heartbeat: periodic status message in #platform-alerts every 6 hours"
  - "Timestamps: America/Chicago + UTC"
  - "Skip uses Discord Select Menu component (not a button) for 10/30/120 min snooze"
  - "Approve updates original message to show who pressed what and outcome"
open_questions:
  - "MVP1 scope revision: real Approve actions contradict product-brief 'read-only' principle — architecture must confirm action endpoint design and safety gates"
  - "ArgoCD diff classification (C1 from adversarial review) still unvalidated — image-promotion vs. live-drift distinction may not be achievable without manifest API validation"
  - "Temporal zombie detection uses elapsed-time proxy (not attempt count) — alert template should reflect this"
depends_on: []
blocks:
  - architecture.md
updated_at: '2026-05-11T00:00:00Z'
---

# UX Design — Terminus Watchdog Agent

**Author:** Todd Hintzmann (via Lens BusinessPlan)
**Date:** 2026-05-11
**Feature:** `terminus-watchdog-agent`

---

## 1. Design Context

The Terminus Watchdog Agent has no web UI. The operator interface is **Discord**. Every UX decision in this document is a decision about Discord message structure, component layout, interaction behavior, and operator cognitive load.

The primary user is Todd — a solo platform operator who receives alerts while working and needs to triage them in under 90 seconds without leaving Discord.

**MVP1 Scope Revision:** The original product brief specified read-only operation with scaffolded buttons for future MVP. After stakeholder review, **real actions are enabled in MVP1**: `Approve` triggers ArgoCD sync / Temporal termination / pod delete, `Skip` snoozes the alert, `Investigate` surfaces deep-links. This changes the architecture from passive observer to interactive operator console. The architecture document must address action endpoint security, confirmation gates, and idempotency.

---

## 2. Design Principles

| Principle | Application |
|-----------|-------------|
| **90-second rule** | Every alert must be fully scannable in under 90 seconds: pattern name, resource, duration, diagnosis, recommended action — all visible without expanding or clicking |
| **Classification-first** | Show the *classified* pattern name (`argocd-live-drift`), not the raw status (`OutOfSync`). The classification is the diagnosis. |
| **Actionable or silent** | If the watchdog has nothing useful to say, it says nothing. Informational alerts are sent to `#platform-info` so they don't create noise in the primary alert channel. |
| **Dedup-visible, not dedup-silent** | When an alert is suppressed by cooldown, the suppression is logged internally. The operator sees the suppression count in the heartbeat message, not in the alert channel. |
| **One action per button** | Each button does one specific thing. No ambiguity about what `Approve` will do — the recommended action field in the alert states it explicitly before the button is pressed. |

---

## 3. Channel Architecture

### 3.1 Channel Layout

| Channel | Purpose | Severity | @mention |
|---------|---------|---------|---------|
| `#platform-alerts` | High-severity alerts + watchdog heartbeat | High | Yes — `@Todd` |
| `#platform-info` | Medium and Informational alerts | Medium, Informational | No |

All High-severity patterns route to `#platform-alerts`. All Medium and Informational patterns route to `#platform-info`. The heartbeat message also posts to `#platform-alerts` so the operator knows the system is alive without having to check a separate channel.

### 3.2 Channel Configuration Guidance

```
#platform-alerts
  Description: Terminus platform health — High severity alerts and watchdog status
  Notification settings: @mentions only (not all messages)
  
#platform-info
  Description: Terminus platform — Medium and informational events
  Notification settings: Unread count only (no push notifications)
```

---

## 4. Severity Visual System

### 4.1 Discord Embed Color Codes

Discord embeds have a left-side color bar. The color communicates severity at a glance.

| Severity | Color | Hex | Emoji Prefix |
|----------|-------|-----|--------------|
| High | Red | `#FF4444` | 🚨 |
| Medium | Orange | `#FF8C00` | ⚠️ |
| Informational | Blue | `#4A90E2` | ℹ️ |
| Resolved | Green | `#00C851` | ✅ |
| Suppressed (cooldown) | Gray | `#888888` | 🔕 |

### 4.2 @mention Rules

- High severity alerts: begin message content with `<@TODD_USER_ID>` before the embed
- Medium and Informational: no @mention, rely on channel unread indicator

### 4.3 Severity-to-Pattern Mapping

| Pattern ID | Severity | Channel |
|------------|---------|---------|
| `argocd-live-drift` | High | `#platform-alerts` |
| `argocd-stuck-sync` | Medium | `#platform-info` |
| `argocd-order-day-unsync` | High | `#platform-alerts` |
| `argocd-image-promotion` | Informational | `#platform-info` |
| `temporal-zombie-critical` | High | `#platform-alerts` |
| `temporal-zombie-activity` | Medium | `#platform-info` |
| `temporal-stale-workflow` | Medium | `#platform-info` |
| `temporal-postgres-connectivity` | High | `#platform-alerts` |
| `k8s-crashloopbackoff` | High | `#platform-alerts` |
| `k8s-deployment-unavailable` | Medium | `#platform-info` |
| `k8s-node-notready` | High | `#platform-alerts` |

---

## 5. Alert Embed Message Format

### 5.1 Base Embed Structure

Every alert follows a consistent embed structure:

```
[Color bar: severity color]
[Title]   {emoji} {Pattern Display Name}
[Description]  Plain-language event description (1–2 sentences)

Fields:
  Resource       {namespace} / {resource-name}
  Duration       {how long this condition has existed}
  Diagnosis      {classified cause — human-readable pattern description}
  Recommended    {specific action statement, not generic}

[Footer]  {pattern-id} · Detected {local time} ({UTC})
```

### 5.2 Field Definitions

| Field | Content | Max Length |
|-------|---------|-----------|
| Resource | `{namespace}/{resource-name}` or `{app-name}` | 80 chars |
| Duration | Human-readable elapsed time: `4 min`, `2 hr 15 min`, `3 days` | 30 chars |
| Diagnosis | One-sentence classified cause | 200 chars |
| Recommended | One specific imperative sentence | 200 chars |
| Footer | `{pattern-id} · Detected {local-time} ({UTC})` | — |

### 5.3 Component Layout

```
[Action Row 1 — Buttons]
  [✅ Approve]    [🔍 Investigate]

[Action Row 2 — Skip Select Menu]
  [Skip ▾]  (options: Snooze 10 min / Snooze 30 min / Snooze 2 hours)
```

**Note:** `Skip` uses a Discord **Select Menu** component (not a button) because it carries a duration parameter. This requires Action Row 2 to be a separate row from the buttons.

### 5.4 Button Specifications

| Component | Type | Style | Label | custom_id pattern |
|-----------|------|-------|-------|-------------------|
| Approve | Button | Success (3) | ✅ Approve | `approve:{domain}:{resource}` |
| Investigate | Button | Secondary (2) | 🔍 Investigate | `investigate:{domain}:{resource}` |
| Skip | Select Menu | — | Skip ▾ | `skip:{domain}:{resource}:{minutes}` |

`custom_id` encoding:
- `domain`: `argocd` / `temporal` / `k8s`
- `resource`: app name, workflow ID, or `{namespace}/{pod-name}`
- `minutes`: snooze duration from dropdown selection (10 / 30 / 120)

---

## 6. Alert Information Architecture — Per Pattern

### 6.1 ArgoCD Patterns

#### `argocd-live-drift` (High)

```
🚨 ArgoCD: Live Drift Detected

Description:
  kaylee-agent is out of sync due to an in-cluster configuration change,
  not an image promotion. Manual intervention required.

Resource:     fourdogs-kaylee / kaylee-agent
Duration:     22 min
Diagnosis:    Manifest diff shows config change not present in git HEAD — this
              is not an image tag update. Live drift from git source.
Recommended:  Review in-cluster changes; if drift is accidental, approve sync
              to restore git state.

Footer:  argocd-live-drift · Detected 09:42 CT (15:42 UTC)
```

**Approve action:** `argocd app sync {app-name} --force`
**Skip:** Snooze re-alert for selected duration
**Investigate:** Ephemeral reply with: ArgoCD app URL, git diff link

---

#### `argocd-image-promotion` (Informational)

```
ℹ️ ArgoCD: Image Promotion Pending

Description:
  kaylee-agent is OutOfSync — CI has pushed a new image tag.
  This is a routine deploy, not a live drift event.

Resource:     fourdogs-kaylee / kaylee-agent
Duration:     3 min
Diagnosis:    OutOfSync state matches a recent image tag push. No config drift detected.
Recommended:  No action required — ArgoCD auto-sync will apply, or approve to
              trigger immediately.

Footer:  argocd-image-promotion · Detected 09:12 CT (15:12 UTC)
```

**Approve action:** `argocd app sync {app-name}`
**Skip:** Snooze re-alert
**Investigate:** Ephemeral reply with: ArgoCD app URL, last commit link

---

#### `argocd-stuck-sync` (Medium)

```
⚠️ ArgoCD: Sync Stalled

Description:
  fourdogs-central has been in a Progressing sync state for 8 minutes.
  Sync appears stalled.

Resource:     fourdogs / fourdogs-central
Duration:     8 min
Diagnosis:    Sync triggered but health status has not recovered. Likely a
              deployment rollout stuck waiting for pod readiness.
Recommended:  Check pod events in fourdogs namespace; if stuck pod is identified,
              delete it to unblock rollout.

Footer:  argocd-stuck-sync · Detected 10:04 CT (16:04 UTC)
```

---

#### `argocd-order-day-unsync` (High)

```
🚨 ArgoCD: Unsync During Order Day Window

Description:
  fourdogs-central is OutOfSync during a high-priority order window (Sunday 9am–3pm CT).
  Delayed deploys during this window directly impact order processing.

Resource:     fourdogs / fourdogs-central
Duration:     11 min
Diagnosis:    App is OutOfSync during a configured high-priority window. Standard
              cooldown rules are bypassed for this alert class.
Recommended:  Immediate review — approve sync or investigate cause before order
              volume peaks.

Footer:  argocd-order-day-unsync · Detected 09:52 CT (15:52 UTC)
```

**Note:** This pattern bypasses cooldown deduplication — it fires on every poll cycle during an active window.

---

### 6.2 Temporal Patterns

#### `temporal-zombie-critical` (High)

```
🚨 Temporal: Zombie Workflow — Critical

Description:
  fourdogs-catalog-trigger / CatalogSeedSecrets has been running for 4 days 7 hours
  with no completion. This workflow is consuming worker slots.

Resource:     fourdogs-etailpet-trigger / fourdogs-catalog-trigger
Duration:     4d 7h
Workflow ID:  ce8f3a12-... (truncated)
Diagnosis:    Workflow elapsed time exceeds critical threshold (72h). Pattern matches
              prior SeedSecrets zombie incident. Possible missing task template or
              non-existent resource dependency.
Recommended:  Terminate this workflow. If the task template is missing, create it
              before restarting.

Footer:  temporal-zombie-critical · Detected 11:15 CT (17:15 UTC)
```

**Approve action:** `temporal workflow terminate --workflow-id {workflow-id} --reason "watchdog-zombie-terminate"`
**Skip:** Snooze
**Investigate:** Ephemeral reply with: Temporal Web UI workflow URL

---

#### `temporal-zombie-activity` (Medium)

```
⚠️ Temporal: Long-Running Workflow

Description:
  fourdogs-etailpet-sales-trigger / RunSmokeTest has been running for 3 hours 20 min.

Resource:     fourdogs-etailpet-trigger / fourdogs-etailpet-sales-trigger
Duration:     3h 20min
Workflow ID:  a4c91b07-... (truncated)
Diagnosis:    Workflow elapsed time exceeds activity threshold (2h). May be legitimate
              long-running work or may be stalled. Not yet at critical threshold.
Recommended:  Monitor. If duration continues to grow without progress, escalate to
              zombie-critical.

Footer:  temporal-zombie-activity · Detected 11:15 CT (17:15 UTC)
```

---

#### `temporal-stale-workflow` (Medium)

```
⚠️ Temporal: Stale Workflow — No Progress

Description:
  ReleaseWorkflow for fourdogs-central has been running for 45 min with no
  activity events in the last 30 min.

Resource:     fourdogs / ReleaseWorkflow
Duration:     45 min (stale 30 min)
Workflow ID:  b2e77c44-... (truncated)
Diagnosis:    Workflow history shows no new events in the stale window. May be
              waiting on an external signal or stuck at an activity retry.
Recommended:  Inspect workflow history for the last activity and its retry state.

Footer:  temporal-stale-workflow · Detected 14:22 CT (20:22 UTC)
```

---

#### `temporal-postgres-connectivity` (High)

```
🚨 Temporal: Postgres Connectivity Lost

Description:
  Temporal worker logs show postgres connection failures in the last 5 minutes.
  Active workflows may be stalled.

Resource:     terminus / temporal-worker
Duration:     5 min
Diagnosis:    Log pattern "connection refused" or "deadline exceeded" detected in
              Temporal worker logs via Loki query. This typically indicates postgres
              pod restart or network partition.
Recommended:  Check postgres pod health in terminus namespace. Temporal workflows
              will auto-retry — no manual termination needed unless postgres remains down.

Footer:  temporal-postgres-connectivity · Detected 08:33 CT (14:33 UTC)
```

---

### 6.3 Kubernetes Patterns

#### `k8s-crashloopbackoff` (High)

```
🚨 K8s: CrashLoopBackOff

Description:
  fourdogs-central pod fourdogs-central-7d9f4b-xkr2p is in CrashLoopBackOff
  with 8 restarts in the last 12 minutes.

Resource:     fourdogs / fourdogs-central-7d9f4b-xkr2p
Duration:     12 min (8 restarts)
Diagnosis:    Pod is crash-looping. Restart count exceeds threshold. Not a
              transient startup crash — this is a sustained failure.
Recommended:  Check pod logs for the crash cause. Delete the pod to force a
              fresh rollout if the cause is identified.

Footer:  k8s-crashloopbackoff · Detected 16:08 CT (22:08 UTC)
```

**Approve action:** `kubectl delete pod {pod-name} -n {namespace}`
**Suppression note:** Pod that recovers (Running + Ready) within 2 minutes after detection does not generate an alert. Oscillating pods (crash cycle < 2 min) fire on every crash-then-recover cycle after the first alert.

---

#### `k8s-deployment-unavailable` (Medium)

```
⚠️ K8s: Deployment Unavailable

Description:
  fourdogs-central deployment has 0 available replicas for the last 6 minutes.

Resource:     fourdogs / fourdogs-central
Duration:     6 min (0/1 available)
Diagnosis:    Desired replicas present but unavailable. Likely rollout in progress
              or pod readiness probe failing.
Recommended:  Check rollout status and pod events in fourdogs namespace.

Footer:  k8s-deployment-unavailable · Detected 16:09 CT (22:09 UTC)
```

---

#### `k8s-node-notready` (High)

```
🚨 K8s: Node NotReady

Description:
  k3s-worker-04 has been in NotReady state for 7 minutes.

Resource:     k3s-worker-04
Duration:     7 min
Diagnosis:    Node heartbeat has stopped or kubelet is unresponsive. Workloads on
              this node are at risk of eviction after 5 min (default tolerations).
Recommended:  SSH to k3s-worker-04 and check kubelet status and network
              connectivity. Cordon the node if recovery is not imminent.

Footer:  k8s-node-notready · Detected 21:30 CT (03:30 UTC)
```

---

## 7. Alert Lifecycle States

An alert message moves through these states. Each state change updates the original Discord message in place.

### 7.1 Active State (Initial)
Standard embed format as described above. Buttons enabled.

### 7.2 Acknowledged State (after button press)
When the operator presses any button, the original message is updated:

- Buttons are **disabled** (grayed out)
- A `Resolution` field is appended to the embed:

```
Resolution:   ✅ Approve pressed by Todd at 09:44 CT — argocd app sync kaylee-agent queued
```

or:

```
Resolution:   ⏭ Snoozed 30 min by Todd at 09:44 CT — re-alerts at 10:14 CT
```

or:

```
Resolution:   🔍 Investigate pressed by Todd at 09:44 CT — deep-links sent
```

The color bar changes to `#888888` (gray) for Approved/Investigate. For Skip, the color bar remains at severity color until the snooze expires.

### 7.3 Resolved State (condition clears)
If the watchdog detects that the condition is resolved (e.g., ArgoCD app is now Synced, workflow terminated, node is Ready), it updates the message again:

- Color bar changes to `#00C851` (green)
- Title prefix changes to ✅
- A `Resolved` field is appended:

```
Resolved:     Condition cleared at 09:52 CT (15:52 UTC) — 10 min after detection
```

Resolved updates only occur for the most recent active alert for a given resource. Snoozed alerts that self-resolve are also updated.

---

## 8. Approve Action Specifications

Actions executed by `Approve` in MVP1:

| Pattern Group | Approve Action | Command |
|--------------|---------------|---------|
| `argocd-*` | Trigger app sync | `argocd app sync {app-name} [--force]` via ArgoCD API `POST /api/v1/applications/{name}/sync` |
| `temporal-zombie-*` + `temporal-stale-workflow` | Terminate workflow | Temporal SDK `TerminateWorkflow(workflowId, reason="watchdog-zombie-terminate")` |
| `k8s-crashloopbackoff` | Delete pod | `kubectl delete pod {pod-name} -n {namespace}` via k8s API |
| `argocd-order-day-unsync` | Trigger app sync | Same as `argocd-*` |
| `k8s-node-notready` | None — no safe automated action | `Approve` button is **hidden** for this pattern. Show `Investigate` only. |
| `k8s-deployment-unavailable` | None — informational | `Approve` button is **hidden**. Show `Investigate` only. |
| `temporal-postgres-connectivity` | None — postgres recovery is external | `Approve` button is **hidden**. Show `Investigate` only. |

**Safety requirement (for architecture):** Each Approve action must:
1. Verify the condition is still active before executing (re-query source before action)
2. Log the action with timestamp, operator, resource, and outcome
3. Return a concrete success/failure message to Discord (update the acknowledged state field)

---

## 9. Investigate Deep-Link Specifications

`Investigate` sends an ephemeral reply (visible only to the button presser) with resource-specific deep-links:

| Pattern Domain | Links Provided |
|---------------|---------------|
| ArgoCD | ArgoCD app URL: `https://argocd.terminus.internal/applications/{app-name}` |
| Temporal | Temporal Web UI: `https://temporal.terminus.internal/namespaces/{ns}/workflows/{id}/{run-id}` |
| K8s pod | Cluster dashboard (if available) or k9s command hint: `k9s -n {namespace} -l "pod={pod-name}"` |
| K8s node | `ssh root@{node-ip}` + node describe kubectl command |

---

## 10. Skip / Snooze UX

Skip uses a **Discord Select Menu** (Component Type 3) in a dedicated Action Row:

```json
{
  "type": 1,
  "components": [{
    "type": 3,
    "custom_id": "skip:{domain}:{resource}",
    "placeholder": "Skip ▾",
    "options": [
      {"label": "Snooze 10 minutes",  "value": "10",  "description": "Re-alerts at 10-min mark if still active"},
      {"label": "Snooze 30 minutes",  "value": "30",  "description": "Re-alerts at 30-min mark if still active"},
      {"label": "Snooze 2 hours",     "value": "120", "description": "Re-alerts at 2-hr mark if still active"}
    ]
  }]
}
```

**Snooze behavior:** The in-memory suppression map is updated with `{resource}: snoozed_until = now + duration`. The alert re-fires after the snooze window if the condition is still active. The existing message is updated to show "Snoozed until {time}" as described in the lifecycle states.

---

## 11. Heartbeat Message Format

Sent to `#platform-alerts` every 6 hours:

```
[Embed: Blue #4A90E2]
[Title]  🟢 Watchdog Healthy

Fields:
  Last cycle:    09:40 CT (15:40 UTC)
  Next cycle:    15:40 CT (21:40 UTC)
  Cycle result:  ✅ No alerts — all systems nominal

or:

  Cycle result:  2 alerts sent, 1 suppressed (cooldown), 0 snoozed

Footer:  terminus-watchdog v0.1.0 · Uptime: 14h 22min
```

**When watchdog is unhealthy** (missed heartbeat or startup failure), the prior heartbeat message's embed color changes to `#FF4444` (red) and a new message is sent:

```
🚨 Watchdog: Heartbeat Missed

Description: No heartbeat received in 7 hours. Watchdog pod may be down.

Footer:  Generated by k8s liveness probe alert or external health check
```

**Architecture note:** The missed-heartbeat alert requires either a separate lightweight exporter or the k8s liveness probe to notify Discord externally. This is a self-health design requirement for `architecture.md`.

---

## 12. Cold-Start Behavior

From the adversarial review: on restart, the in-memory suppression map is empty. All currently-active problems will fire as new alerts.

**Cold-start suppression window:** On startup, the watchdog enters a **3-minute grace window** during which it:
1. Runs all detection cycles normally
2. Populates the suppression map with current state
3. Does NOT send any alerts

After the grace window, only **newly detected conditions** (conditions not present at startup) generate alerts. Conditions already present at startup are added to the suppression map with a `grace_period_observed: true` flag.

**Cold-start notification:**

```
[Embed: Blue]
ℹ️ Watchdog: Starting Up

Description:
  Terminus Watchdog Agent started. 3-minute grace window active —
  existing conditions are being cataloged. Alerts resume at 09:43 CT.

Footer:  terminus-watchdog v0.1.0 · terminus-watchdog-{pod-name}
```

---

## 13. Maintenance Window UX

Maintenance windows are defined via a ConfigMap in the watchdog namespace:

```yaml
# configmap: watchdog-config
maintenance_windows:
  - name: "infra-maintenance"
    schedule: "2026-05-15T02:00:00Z/PT4H"  # ISO 8601 interval
    affected_namespaces: ["*"]
    suppress_alerts: false  # suppress=false: still alert, but add maintenance badge
```

**`suppress_alerts: false` (default):** Alerts still fire but include a `Maintenance` badge in the embed footer:

```
Footer:  k8s-node-notready · Detected 02:15 CT (08:15 UTC) · ⚙️ Maintenance window active
```

**`suppress_alerts: true`:** Alerts are suppressed for the window duration. A summary is sent to `#platform-info` after the window closes:

```
[Embed: Gray]
ℹ️ Maintenance Window Complete

Description:
  infra-maintenance window closed. 4 alerts were suppressed during the window.

Fields:
  Window:    02:00–06:00 CT (08:00–12:00 UTC)
  Suppressed: k8s-deployment-unavailable (2), argocd-stuck-sync (2)
```

**High-priority windows** (e.g., FourDogs order day) are defined in the same ConfigMap under `high_priority_windows` and used by the `argocd-order-day-unsync` pattern to identify when peak-window alerting applies.

---

## 14. Operator Mental Model

This section documents the intended operator cognitive experience — how Todd perceives and uses the system.

### Normal Day (no alerts)
- `#platform-alerts` is quiet except for the 6-hour heartbeat messages
- `#platform-info` may have `argocd-image-promotion` informational messages during active development
- Operator knows the platform is healthy without checking anything

### High-Severity Alert (e.g., zombie workflow)
1. Push notification fires (Discord @mention)
2. Todd opens `#platform-alerts`
3. Reads the alert embed in under 90 seconds: knows what it is, how long it's been happening, what to do
4. Presses `Approve` to terminate the zombie workflow
5. Original message updates: buttons disabled, resolution field shows "workflow terminated"
6. Within 2 minutes: embed updates to ✅ Resolved

### Ambiguous Alert (e.g., ArgoCD OutOfSync — not sure if image or drift)
1. Todd opens `#platform-alerts`
2. Diagnosis field states: "Manifest diff shows config change not present in git HEAD — this is live drift, not image promotion" (or the inverse for image-promotion)
3. If he wants to verify: presses `Investigate` → ephemeral reply shows ArgoCD app URL + git diff link
4. Confident decision: `Approve` or `Skip`

### Order Day (Sunday morning, FourDogs volume window)
1. Any ArgoCD OutOfSync in the fourdogs namespace fires immediately to `#platform-alerts` regardless of normal cooldown
2. Badge in footer: `🏪 High-priority window active`
3. Same interaction pattern — but operator knows to prioritize resolution immediately

---

## 15. Open Architecture Requirements (for architecture.md)

These UX decisions create direct requirements for the architecture:

| UX Decision | Architecture Requirement |
|-------------|--------------------------|
| Real Approve actions in MVP1 | Action endpoint per domain (ArgoCD API, Temporal SDK, k8s API); re-verify condition before action; idempotent execution |
| Public Discord interaction endpoint | HTTPS ingress from Discord servers; Cloudflare Tunnel or equivalent; mTLS not required (Discord signs payloads) |
| Original message update after action | Bot stores `(message_id, channel_id)` in memory per active alert for `PATCH /webhooks/{app_id}/{token}/messages/@original` |
| Select Menu for Skip duration | Discord interaction handler must parse Select Menu interactions (type=3), not just button interactions (type=2) |
| Alert lifecycle Resolved updates | Watchdog maintains active alert map: `{resource} → message_id`; clears on resolved detection |
| Cold-start suppression window | 3-minute startup grace period logic; suppression map pre-populated before any alerts sent |
| Heartbeat missed-alert | External health check required (or k8s liveness probe endpoint that sends Discord DM/message) |
| Maintenance window ConfigMap | ConfigMap watch or periodic read in detection loop |
| `k8s-node-notready` has no Approve | Per-pattern button visibility rules in message builder |
