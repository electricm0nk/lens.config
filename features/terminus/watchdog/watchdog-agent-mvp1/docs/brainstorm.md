# Brainstorm: terminus-watchdog-agent

**Feature**: terminus-watchdog-agent  
**Domain**: terminus / watchdog  
**Phase**: PrePlan  
**Date**: 2026-05-10  
**Method**: Three-phase brainstorming — Reverse Brainstorming → Five Whys → Assumption Reversal

---

## Context

The Terminus homelab platform runs k3s, ArgoCD, Temporal, Vault, and multiple application services across Proxmox VMs. Currently, platform health issues are caught manually — when Todd notices something is wrong. This feature defines an IT Watchdog Agent that surfaces what is currently caught by eye, pre-diagnoses it, and delivers a one-button-approval Discord alert so response time drops from "whenever I notice it" to "seconds after it happens."

The watchdog is **not** a fixing system. It is an automated pair of eyes and a structured voice.

---

## Phase 1: Reverse Brainstorming — Safety Boundary Rules

**Technique**: Asked "what would make this watchdog dangerous?" to identify failure modes, then flipped each to a design constraint.

### Danger → Safety Rule Table

| # | Failure Mode (Dangerous Behavior) | Design Constraint (Safety Rule) |
|---|-----------------------------------|----------------------------------|
| 1 | Takes remediation action without human approval | Every remediation requires explicit human approval via Discord button — no autonomous write actions |
| 2 | Acts on stale or cached state and makes a wrong call | Always fetch fresh from source at decision time — no cached state drives action |
| 3 | Depends on the system it is monitoring for its own health signals | Self-monitoring pipeline is fully independent of monitored systems |
| 4 | Silently suppresses repeated alerts — "we've seen this before" | Never suppress — always surface, even if a pattern was seen before; de-duplicate only in display, not in detection |
| 5 | Depends on another agent (e.g., Kaylee) being alive to deliver an alert | No agent dependency for alert delivery — if Kaylee is the system being watched, the watchdog cannot use Kaylee to tell you about it |
| 6 | Takes action during a maintenance window and breaks something expected to be offline | Respect maintenance window annotations; escalate-only mode during flagged windows |
| 7 | Executes irreversible or destructive actions | MVP1: read-only — observe and report only; no write, delete, or restart actions |

These seven rules define the non-negotiable boundary surface for all MVP phases.

---

## Phase 2: Five Whys — Root Cause Investigation from Live Data

**Technique**: Mined live production logs and ArgoCD state to find the actual recurring patterns that motivated this feature. Two root cause chains were traced.

### Root Cause Chain 1: ArgoCD OutOfSync — Silent Pending Deploy

**Observed symptom**: Multiple ArgoCD apps show OutOfSync status. Looks like live config drift.

**Five Whys trace**:
1. Why OutOfSync? → Image tags in `terminus.infra` values.yaml were updated by CI after a successful build
2. Why not deployed? → ArgoCD doesn't auto-sync these apps; sync must be manually triggered or via a webhook
3. Why no alert? → ArgoCD OutOfSync looks identical whether it's a benign pending-deploy or a live drift from a manual change
4. Why didn't we notice sooner? → No classification — ArgoCD has no concept of "expected" vs "unexpected" sync delta
5. Why does this matter? → A genuine drift from a prod manual change would look exactly the same; we have no way to know without manual inspection

**Root cause**: ArgoCD exposes no semantic distinction between "CI pushed a new tag and we haven't deployed yet" vs "someone edited a resource in-cluster and now it doesn't match git." Both surface as OutOfSync.

**Watchdog implication**: Implement OutOfSync classification before alerting:

| Class | Description | Alert Behavior |
|-------|-------------|----------------|
| `image-promotion` | values.yaml image tag advanced by CI — pending deploy | Informational (notify, approve-to-deploy button) |
| `live-drift` | In-cluster resource differs from git without a corresponding commit | High-priority alert — manual intervention likely needed |
| `stuck-sync` | Sync was triggered but has not completed within threshold | Alert — possible resource conflict or webhook failure |

**Special case**: If an `image-promotion` OutOfSync is detected during a known high-order-volume window (e.g., Sunday order day for FourDogs), escalate to high-priority — a delayed deploy during peak traffic is not benign.

---

### Root Cause Chain 2: Temporal Zombie Workflows — Silent Retry Storm

**Observed symptom**: Temporal worker logs showed `SeedSecrets` activity for `fourdogs-catalog-trigger` at attempt **3297** with error: `semaphore: no template with name "seed-fourdogs-catalog-trigger-secrets" in project 3`

**Five Whys trace**:
1. Why is attempt 3297 running? → Temporal retries indefinitely unless a non-retryable error is returned
2. Why hasn't it been marked non-retryable? → The `SeedSecrets` activity doesn't distinguish "template not found" from transient Semaphore errors; it retries both
3. Why is the template missing? → The Semaphore template was never created for this workflow — a setup gap when the service was onboarded
4. Why no alert after thousands of retries? → No retry-count threshold monitoring exists; Temporal surfaces it only in the UI if you look
5. Why didn't we catch it during deploy? → The `ReleaseWorkflow` runs async post-deploy; the smoke test passed, so the pipeline showed green

**Root cause**: Missing Semaphore template + Temporal's default retry policy + no retry-storm detection = background zombie workflows that consume worker slots silently.

**Second confirmed instance**: `RunSmokeTest` for `fourdogs-etailpet-sales-trigger` — error `semaphore: no template with name "verify-fourdogs-etailpet-sales-trigger-deploy"` at attempt 10+. Same failure class, different workflow.

**Watchdog implication**: Monitor Temporal workflow activity attempt counts. Detection patterns:

| Pattern | Threshold | Alert |
|---------|-----------|-------|
| Activity attempt count > 50 | Per activity instance | Alert with workflow ID, activity type, last error |
| Activity attempt count > 500 | Per activity instance | High-priority — zombie workflow, consuming worker slots |
| Workflow running > N days with no progress | Configurable per workflow type | Stale workflow alert |

**Also observed**: Temporal postgres connectivity timeout (`dial tcp 10.0.0.56:5432: connect: connection timed out`) logged 2026-05-09 16:03–16:16. Classified as a separate, transient postgres connectivity event — resolved without intervention. Watchdog should detect and classify postgres connectivity drops to Temporal's backend.

---

## Phase 3: Assumption Reversal — Architecture Decisions

**Technique**: Identified implicit design assumptions, reversed each to find the better alternative, then locked a decision.

### Decision Table

| # | Assumed | Reversed | Decision |
|---|---------|---------|---------|
| 1 | Watchdog polls on a fixed schedule | Polls are too slow and miss transient events; event-driven is better | **Event-primary + heartbeat poll fallback** — subscribe to ArgoCD/k8s watch events where available; heartbeat poll at configurable interval as belt-and-suspenders |
| 2 | Watchdog maintains internal state cache | State cache becomes stale in a dynamic cluster; acting on stale state is dangerous | **Stateless** — always go fresh to source at decision time; state is derived, never stored as a decision input |
| 3 | Watchdog alerts, Todd diagnoses | Pre-diagnosis alert puts all context work on Todd; delay between alert and response increases | **Runbook-first output** — each alert includes pre-diagnosed context (what was seen, what it likely means, recommended action); Discord message includes approve / skip / investigate buttons |
| 4 | One unified watchdog watches everything | Single scope makes it hard to add capability-specific logic or gate high-risk actions | **Two-phase scope**: MVP1 = broad observation (watch everything, learn patterns); MVP2 = capability registry (promote learned patterns into structured capability units with their own diagnosis logic and safety gates) |
| 5 | Todd is the only consumer of output | Agent-to-agent signaling is a natural next step as the platform grows | **MVP1 = Discord-direct** (one consumer, no agent dependency); architecture treats Discord as an **output adapter**, not the output layer — future agent consumer is a config-time adapter swap, not a code rewrite |

### Note on Future Agent Consumer

Kaylee is permanently scoped to FourDogs and is walled off from the Terminus platform. She is not a candidate for watchdog signal consumption. Future agent consumers (for Terminus platform signaling) will be separate agents. The adapter pattern ensures the watchdog implementation does not assume Discord is permanent.

---

## MVP1 Detection Patterns

The following patterns were identified during the brainstorming session as the concrete detection targets for MVP1. These are grounded in real incidents observed during this session.

| Pattern ID | Source | What to Detect | Alert Class |
|------------|--------|----------------|-------------|
| `argocd-image-promotion` | ArgoCD API / git diff | OutOfSync where diff is limited to image tag in values.yaml, matching a recent CI commit | Informational |
| `argocd-live-drift` | ArgoCD API | OutOfSync where diff is NOT image-tag-only — in-cluster resource differs from git | High |
| `argocd-stuck-sync` | ArgoCD API | Sync triggered but not completed within threshold | Medium |
| `argocd-order-day-unsync` | ArgoCD API + calendar | Any OutOfSync during known order-day window | High |
| `temporal-zombie-activity` | Temporal API / metrics | Activity attempt count > 50 on a single instance | Medium |
| `temporal-zombie-critical` | Temporal API / metrics | Activity attempt count > 500 — worker slot leaking | High |
| `temporal-stale-workflow` | Temporal API | Workflow running > threshold with no state transitions | Medium |
| `temporal-postgres-connectivity` | Temporal worker logs | Postgres connection timeout/refused errors in Temporal namespace | High |
| `secret-shared-detection` | Vault + k8s | Secret used by more than N namespaces without explicit shared tag | Medium |

---

## MVP1 → MVP2 Transition Notes

- Every alert emitted in MVP1 should include a `pattern_id` field (even if initially `unknown`) — enables clustering and promotion later
- Pattern promotion from observation log to capability registry is the primary MVP2 work item
- MVP2 capability units each carry: diagnosis logic, safety gate, approved action surface, rollback definition
- Lessons from MVP1 drive which patterns get promoted first (highest frequency + highest operator cost)

---

## Open Items Before Research Phase

1. **Constitution conflict**: `terminus/watchdog` constitution `permitted_tracks` = `[express, quickdev, hotfix-express, spike]`. The `full` track (current feature.yaml value) is not listed. Must resolve before plan gate — options: add `full` to the constitution scaffold, or discuss track change.
2. **Temporal API access**: Confirm Temporal visibility API is accessible from within the cluster for workflow state queries (vs. requiring CLI only).
3. **ArgoCD watch API**: Confirm ArgoCD exposes a watch/event stream vs. requiring poll-only.
4. **Discord bot architecture**: Determine if an existing Discord bot exists in the platform or if this is a new bot deployment.
