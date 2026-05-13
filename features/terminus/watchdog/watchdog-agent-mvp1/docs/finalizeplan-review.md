---
feature: terminus-watchdog-agent
doc_type: finalizeplan-adversarial-review
phase: finalizeplan
source: phase-complete
verdict: pass-with-warnings
reviewer: lens-adversarial-review
reviewed_at: '2026-05-11T00:00:00Z'
findings_summary:
  high: 3
  medium: 3
  low: 2
  total: 8
predecessor_review_verdict: pass-with-warnings
predecessor_review_artifact: techplan-adversarial-review.md
---

# FinalizePlan Adversarial Review — Terminus Watchdog Agent

**Phase:** FinalizePlan  
**Source:** phase-complete  
**Verdict:** pass-with-warnings  
**Date:** 2026-05-11  

---

## Reviewed Artifact Set

| Artifact | Status |
|----------|--------|
| `product-brief.md` | Present |
| `research.md` | Present |
| `brainstorm.md` | Present |
| `prd.md` | Present — stale metadata (see H1) |
| `ux-design.md` | Present |
| `architecture.md` | Present |
| `preplan-adversarial-review.md` | Present |
| `businessplan-adversarial-review.md` | Present |
| `techplan-adversarial-review.md` | Present (pass-with-warnings) |

---

## Pre-Review Fixes Applied

The following metadata corrections were resolved during this review gate based on user clarifications and predecessor review carry-over:

| Fix | Applied To | Detail |
|-----|-----------|--------|
| `argocd-order-day-unsync` clarified as distinct detection pattern | Architecture context | Distinct always-on pattern monitoring the `fourdogs-central` (order) ArgoCD app regardless of quiet-hours. Carries forward as a 12th detection pattern entry required in the PRD patterns table. |
| Discord user ID resolved | Config context | `DISCORD_OPS_USER_IDS=386667264683802634` — carried into story acceptance criteria for write-action auth setup |
| Temporal namespace confirmed | Architecture context | Temporal namespace = `default`. Carried into all Temporal integration stories. |
| Cold-start suppression scope confirmed | Architecture context | All 11 detection patterns suppressed for 30 minutes on cold-start (not 3 minutes — user confirmed actual cold-start is ~30 min). Suppression window updated from 3 minutes to 30 minutes. |

---

## Findings

### HIGH

#### H1 — Stale PRD Metadata Blocks Dev-Phase Routing

**Dimension:** Logic Flaw  
**Finding:** `prd.md` frontmatter carries three stale fields that will confuse the `/dev` phase agent and any downstream tooling:
1. `status: draft` — the PRD has been through two adversarial review cycles. It should be `approved`.
2. `open_questions` list contains four entries explicitly marked "required before TechPlan" — all four are now resolved: H1 (ArgoCD diff → inline `status.resources[]`), H2 (public ingress → Gateway mode), H3 (Temporal gRPC → confirmed accessible in `default` namespace), Loki auth (carried as story-level open item, not a TechPlan blocker). These must be cleared or moved to resolved.
3. `externalDependencies` lists `public-ingress: mechanism not yet selected` — resolved by Gateway mode.

**Impact:** `/dev` skill reads `prd.md` status to determine if planning is approved. `draft` status may block or warn incorrectly.  
**Remediation Required:** Update `prd.md` frontmatter before bundle commit: `status: approved`, clear resolved `open_questions`, update `public-ingress` dependency entry.

---

#### H2 — Unverified Temporal gRPC Cross-Namespace Access

**Dimension:** Risk  
**Finding:** The architecture specifies `temporalio` SDK connecting to Temporal gRPC in the `default` namespace. This access path has not been verified against the running Terminus Temporal cluster. The Temporal SDK requires a valid mTLS client certificate pair. It is unknown whether:
- The existing Temporal cluster requires per-client cert provisioning
- The `temporal-cert` and `temporal-key` secrets have been provisioned in Vault at `terminus/watchdog/temporal-cert` and `terminus/watchdog/temporal-key`
- The `temporal-system` namespace RBAC permits external client connections from `terminus-watchdog`

**Impact:** First Temporal integration story will fail at connection establishment if cert or access is not pre-provisioned.  
**Remediation:** Story 0 / pre-dev setup checklist must include: (a) verify Temporal gRPC endpoint is reachable from `terminus-watchdog` namespace, (b) provision client cert in Vault, (c) confirm `default` namespace is accessible.

---

#### H3 — `argocd-order-day-unsync` Missing From Detection Pattern Spec

**Dimension:** Coverage Gap  
**Finding:** `argocd-order-day-unsync` appears in `prd.md` `key_decisions` frontmatter as `argocd-order-day-unsync always-on` but is not defined as a detection pattern in the PRD patterns table. User confirmed this is a **distinct always-on pattern** targeting the `fourdogs-central` (order) ArgoCD app specifically — alerting on OutOfSync regardless of quiet hours.

This pattern has no:
- Entry in the 11-pattern detection table
- Severity definition
- Alert channel assignment
- Suppression/snooze behavior spec
- Quiet-hours exception rationale documented

**Impact:** Story authors will have no spec to implement from. The pattern will either be omitted from implementation or implemented incorrectly.  
**Remediation:** Add `argocd-order-day-unsync` as a 12th detection pattern in `prd.md` with full spec. Carry into epics/stories as its own story.

---

### MEDIUM

#### M1 — Cold-Start Suppression Window Updated to 30 Minutes

**Dimension:** Logic Flaw  
**Finding:** `prd.md` `key_decisions` specifies a 3-minute cold-start suppression window. User clarified actual cluster cold-start time is closer to 30 minutes. A 3-minute window would cause false-positive alerts during the real startup period.  
**Scope:** All 11 detection patterns (user confirmed).  
**Remediation:** Update cold-start suppression window to 30 minutes in `prd.md` key_decisions and carry into the detection loop story acceptance criteria.

---

#### M2 — Four Infrastructure Provisioning Tasks Not Captured as Stories

**Dimension:** Coverage Gap  
**Finding:** The architecture correctly identifies four cross-repo infrastructure provisioning tasks required before implementation can complete, but none are represented as stories or pre-dev checklist items:
1. Create new GitHub repo `terminus.watchdog`
2. Configure ESO `ExternalSecret` objects in `terminus.infra` for the `terminus-watchdog` namespace
3. Add Alertmanager `WatchdogHeartbeatMissed` rule to `terminus.infra` (cross-repo)
4. Provision ArgoCD RBAC token for watchdog service account in `terminus.infra` (cross-repo)

**Remediation:** Capture as a Pre-Dev Infrastructure epic or Story 0 block in the epics bundle.

---

#### M3 — Discord Bot Application Registration Not Confirmed

**Dimension:** Assumption  
**Finding:** A new Discord bot application must be registered in the Discord Developer Portal before any integration can be tested. Required steps that are not yet confirmed done:
- Create application and bot user "terminus-watchdog"  
- Enable `SERVER MEMBERS INTENT` and `MESSAGE CONTENT INTENT` under Privileged Gateway Intents
- Generate bot token → provision to Vault at `terminus/watchdog/discord-bot-token`
- OAuth2 scopes: `bot`, permissions: `Send Messages`, `Read Message History`, `Use Slash Commands`
- Invite bot to the Terminus Discord server with the generated OAuth2 URL
- Note: Bot does NOT need `MANAGE_WEBHOOKS` since delivery is via Gateway not webhook

**Remediation:** Include as a pre-dev manual setup checklist item in Story 0 or `implementation-readiness.md`.

---

### LOW

#### L1 — Loki Auth Status Still Open

**Dimension:** Risk (carry-forward from TechPlan review finding L1)  
**Finding:** Loki HTTP LogQL query API auth mechanism for temporal-postgres-connectivity detection remains unverified. Low priority since Loki is used for only one detection pattern. Acceptable as a story-level open item.  
**Remediation:** Story acceptance criteria for `temporal-postgres-connectivity` detector should include: verify Loki endpoint auth, document in story.

---

#### L2 — `DISCORD_OPS_USER_IDS` Value Now Known

**Dimension:** Config (carry-forward from TechPlan review finding M2 — resolved)  
**Status:** Resolved during this review gate.  
**Resolution:** `DISCORD_OPS_USER_IDS=386667264683802634`. Carry into Vault secret provisioning story and Helm values example. Story acceptance criteria must include: ESO syncs this value from Vault into `DISCORD_OPS_USER_IDS` environment variable in the watchdog pod.

---

## Party-Mode Challenge Summary

Three blind-spot challenge questions surfaced and resolved during this review:

| Question | Outcome |
|----------|---------|
| Is `argocd-order-day-unsync` a distinct pattern or an alias? | **Distinct pattern** — added as H3 finding, requires spec |
| What is `DISCORD_OPS_USER_IDS` value? | **Resolved** — `386667264683802634` |
| Which Temporal namespace? | **Resolved** — `default` |
| Cold-start suppression scope and window? | **Resolved** — 30 min, all 11 patterns |

---

## Predecessor Finding Carry-Forward

Findings from `techplan-adversarial-review.md` that must appear as story acceptance criteria in the downstream bundle:

| TechPlan Finding | Disposition | Story Target |
|-----------------|-------------|-------------|
| H1 — Interaction defer pattern | Story AC: Temporal zombie detector uses defer + background task for Approve interaction | temporal-zombie-detector story |
| H2 — Temporal cert provisioning | Story AC: Pre-dev infra task — provision cert in Vault before integration tests | Story 0 / pre-dev checklist |
| M1 — k8s Watch reconnection | Story AC: Reconnection loop with `watchdog_k8s_watch_reconnects_total` counter | k8s watch detector story |
| M2 — Write-action auth allowlist | Story AC: `DISCORD_OPS_USER_IDS` ESO-synced from Vault | write-action story |
| M3 — Write-action rate limiting | Story AC: Rate limit write-action approvals (1 per resource per 5 min) | write-action story |
| L1 — Loki auth | Story AC: Document Loki auth mechanism | temporal-postgres-connectivity story |
| L2 — Discord rate limiting | Story AC: Implement message send retry with backoff | discord delivery story |

---

## Gate Result

**Verdict: pass-with-warnings**

All findings are Medium or lower after H1, H2, H3 remediation is applied before bundle generation. No finding requires re-architecting. Three High findings are metadata corrections that must be applied to `prd.md` before the downstream bundle is generated.

**Required pre-bundle actions:**
1. Update `prd.md` frontmatter: `status: approved`, clear resolved `open_questions`, fix `public-ingress` dependency entry
2. Add `argocd-order-day-unsync` as 12th detection pattern in `prd.md` patterns table
3. Update cold-start suppression window from 3 min to 30 min in `prd.md` key_decisions
4. Add Temporal namespace (`default`) and `DISCORD_OPS_USER_IDS` (`386667264683802634`) to architecture config section

**Proceed to bundle generation after pre-bundle fixes are applied.**
