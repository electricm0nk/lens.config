# Product Brief: Terminus Watchdog Agent

**Feature:** `terminus-watchdog-agent`  
**Domain/Service:** `terminus / watchdog`  
**Track:** Full  
**Phase:** PrePlan  
**Date:** 2026-05-10  
**Author:** Lens Agent (PrePlan)

---

## The Problem

The Terminus homelab platform — running k3s, ArgoCD, Temporal, Vault, and multiple application services across Proxmox VMs — has no automated health monitoring. Platform issues are caught manually: when the operator (Todd) happens to notice something is wrong.

Two specific failure classes have been observed repeatedly in production:

**1. ArgoCD OutOfSync with no classification.** When CI pushes a new image tag, apps appear OutOfSync. This is indistinguishable from genuine live drift caused by an in-cluster manual change. Both show the same status icon, both require manual inspection to understand. During high-order-volume windows (FourDogs Sunday order day), a delayed deploy is not benign — but there is no mechanism to flag it differently.

**2. Temporal zombie workflows.** A Temporal activity for `fourdogs-catalog-trigger` ran `SeedSecrets` at **attempt 3,297** before being caught — the Semaphore task template had never been created. A second instance ran similarly for `fourdogs-etailpet-sales-trigger`. Temporal's default retry policy retries indefinitely; nothing alerts on attempt count. These workflows consume worker slots silently for days or weeks.

The cost of the current approach is **reactive operator time**: Todd spends time diagnosing things he should have been notified about, and time noticing things that a computer should have already noticed.

---

## The Solution

A lightweight, **read-only, stateless watchdog agent** that:

1. **Watches** the Terminus platform — ArgoCD apps, Temporal workflows, Kubernetes pods and nodes
2. **Classifies** what it sees — distinguishing image-promotion from live-drift, transient errors from zombie storms
3. **Delivers pre-diagnosed alerts** to Discord with one-click actions: `Approve`, `Skip`, `Investigate`

The watchdog is not a fixing system. It is an automated pair of eyes and a structured voice. Every remediation requires explicit human approval. The agent never takes write actions in MVP1.

---

## Users and Stakeholders

| Role | User | Need |
|------|------|------|
| Platform Operator | Todd | Early warning of real problems; pre-diagnosed alerts that minimize diagnostic triage time |
| (Future) Platform Agents | Terminus-scoped agents (not Kaylee) | Programmatic signal consumption from watchdog output adapter |

The MVP1 consumer is Todd, via Discord. The architecture exposes an output adapter interface so that future agent consumers can receive the same signals without code changes.

---

## Scope — MVP1

### In Scope

| Detection Pattern | Source | Alert Class |
|---|---|---|
| `argocd-image-promotion` | ArgoCD REST API | Informational — pending deploy |
| `argocd-live-drift` | ArgoCD REST API | High — in-cluster divergence from git |
| `argocd-stuck-sync` | ArgoCD REST API | Medium — sync triggered but stalled |
| `argocd-order-day-unsync` | ArgoCD API + calendar | High — unsynced during peak window |
| `temporal-zombie-activity` | Temporal Visibility API | Medium — attempt count > 50 |
| `temporal-zombie-critical` | Temporal Visibility API | High — attempt count > 500 |
| `temporal-stale-workflow` | Temporal Visibility API | Medium — running > threshold, no progress |
| `temporal-postgres-connectivity` | Temporal worker logs | High — postgres connection failure to Temporal backend |
| `k8s-crashloopbackoff` | k8s Watch stream | High — pod crash-looping |
| `k8s-deployment-unavailable` | k8s Watch stream | Medium — sustained unavailable replicas |
| `k8s-node-notready` | k8s Watch stream | High — node down |

### Out of Scope (MVP1)

- Autonomous remediation — no write actions
- Secret leak detection (`secret-shared-detection`) — deferred to MVP2
- Agent-to-agent signaling — Discord only in MVP1
- Custom runbook authoring — canned runbook stubs per pattern
- Historical trending / dashboards
- Multi-user approval flows

---

## Non-Goals

- **Not a replacement for Prometheus/Grafana.** Metrics dashboards remain the source of truth for performance data. The watchdog is for operational state events, not time-series metrics.
- **Not a deployment pipeline.** The watchdog observes ArgoCD and Temporal — it does not own or trigger deploys.
- **Not a chat assistant.** It does not answer questions; it delivers alerts and waits for a button press.
- **Not Kaylee.** Kaylee is scoped to FourDogs. The watchdog serves the Terminus platform.

---

## Architecture Principles

These non-negotiable constraints were derived from the brainstorming phase:

1. **Read-only, always.** MVP1 has no write operations. Approval buttons trigger actions through a human-confirmed gate only (future MVP).
2. **Stateless.** Every detection cycle fetches fresh from source. No cached state drives action decisions.
3. **Self-contained.** The watchdog does not depend on any system it monitors for its own health or alert delivery.
4. **Never suppress.** Alerts are not swallowed because a pattern was seen before. De-duplication is display-only; detection always runs.
5. **Maintenance-aware.** Respect annotated maintenance windows — escalate to human, not to action, during flagged windows.

---

## Detection Architecture

| Signal Source | Detection Method | Rationale |
|---|---|---|
| Temporal zombies | Poll `ListWorkflowExecutions` every 5 min | No watch stream exists in Temporal |
| ArgoCD sync state | Poll `GET /api/v1/applications` every 5 min (MVP1); ArgoCD Notifications webhook (MVP2) | Self-contained MVP1; lower latency via webhook for production |
| K8s pods / deployments / nodes | Watch stream (`?watch=true`) via Python k8s SDK | Native k8s mechanism, event-driven, low latency |

**Alert deduplication:** In-memory suppression map per resource. Configurable cooldown (default 30 min). Zombie workflows bypass cooldown (every poll cycle surfaces them until resolved).

**Output adapter:** `AlertDispatcher` interface with Discord implementation in MVP1. Swap adapter config to route to a future agent consumer — no code change required.

---

## Discord Alert Pattern

Each alert message includes:
- **What was detected** — plain-language event description
- **Pre-diagnosis** — likely cause, classified pattern ID
- **Recommended action** — specific, not generic
- **Buttons:** `Approve` (style=Success), `Skip [Xmin]` (style=Secondary), `Investigate` (style=Danger)

Button `custom_id` encodes: `action:domain:resource[:param]` — e.g., `approve:argocd:kaylee-agent`.

Approved actions in MVP1: none (button acknowledged, logged, no action taken). Actions are scaffolded for MVP2 (ArgoCD sync trigger, Temporal workflow terminate).

---

## Success Criteria

| Metric | Target |
|---|---|
| Time to first alert after anomaly | < 6 minutes (worst case: 5-min poll + delivery) |
| False positive rate | < 5% of alerts over first 30 days |
| Zombie workflows surfaced within 30 minutes of crossing threshold | 100% |
| ArgoCD live-drift alerts during order-day windows | 100% |
| Operator diagnostic time per alert | < 2 min (vs. current ~15–30 min for complex incidents) |
| Agent uptime | ≥ 99% (self-healing k8s deployment) |

---

## Risks and Open Items

| # | Risk / Open Item | Severity | Mitigation |
|---|---|---|---|
| 1 | **Constitution track conflict** — `terminus/watchdog` constitution `permitted_tracks` excludes `full`; must update before plan gate | Blocking | Add `full` to `terminus/watchdog/constitution.md` permitted_tracks |
| 2 | **Temporal gRPC access** — watchdog needs cluster-internal access to `temporal.terminus.svc.cluster.local:7233` | Medium | Confirm service discovery from proposed deployment namespace; Temporal SDK uses gRPC |
| 3 | **ArgoCD API token scope** — must provision read-only token | Low | Standard ArgoCD RBAC: `applications/list` + `get` |
| 4 | **Discord bot provisioning** — new bot registration, token storage in Vault/k8s Secret | Low | Standard onboarding; bot scoped to a dedicated `#platform-alerts` channel |
| 5 | **Alert fatigue** — CrashLoopBackOff events can be high-frequency during rolling deploys | Medium | Initial backoff: suppress if pod recovers within 2 min; configurable per-resource cooldowns |
| 6 | **k8s RBAC** — ServiceAccount needs read permissions across namespaces | Low | Standard: `get`, `list`, `watch` on pods, deployments, nodes, events |

---

## MVP2 Preview (Out of Scope Now)

- **Capability registry**: Promote high-frequency, high-cost patterns from observation log into structured capability units with their own diagnosis logic and safety gates
- **ArgoCD Notifications webhook**: Push-based ArgoCD event delivery (lower latency than poll)
- **Approve-to-sync**: Button-triggered ArgoCD sync with confirmation gate
- **Approve-to-terminate**: Button-triggered Temporal workflow termination
- **Agent adapter**: Output adapter routing alerts to a Terminus-scoped platform agent

---

## Deliverables for This Feature

| Artifact | Phase |
|---|---|
| `brainstorm.md` | PrePlan ✅ |
| `research.md` | PrePlan ✅ |
| `product-brief.md` | PrePlan ✅ |
| Adversarial review gate | PrePlan |
| `prd.md` | BusinessPlan |
| `ux-design.md` | BusinessPlan |
| `architecture.md` | TechPlan |
| Epics + stories | DevProposal |
