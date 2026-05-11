# Adversarial Review: terminus-watchdog-agent / techplan

**Reviewed:** 2026-05-11T00:00:00Z
**Source:** phase-complete
**Overall Rating:** pass-with-warnings

---

## Summary

The TechPlan architecture for `terminus-watchdog-agent` is sound at the macro level. All three businessplan adversarial review open questions (H1/H2/H3) are resolved with credible architectural choices: Gateway mode eliminates the H2 public-ingress problem entirely; inline ArgoCD diff parsing with MVP0 validation gate is a defensible approach to H1; and the Prometheus+Alertmanager self-health pattern correctly uses the existing observability stack for H3. No critical findings were identified. Two High findings require acceptance or mitigation before story-writing: the Discord 3-second interaction acknowledgement deadline constraint is not reflected in the concurrent architecture design, and Temporal mTLS cert provisioning has not been confirmed in Vault — a gap that will block MVP0. Three Medium findings address k8s Watch reconnection, Discord authorization, and write-action rate limiting. FinalizePlan can proceed with these findings documented and accepted as open items in stories.

---

## Findings

### Critical

_None._

### High

| # | Dimension | Finding | Recommendation |
|---|-----------|---------|----------------|
| H1 | Logic Flaws | Discord requires interaction acknowledgements within **3 seconds** of button press. If an asyncio.Lock contention or slow detection-loop poll delays the interaction handler by >3s, Discord shows "This interaction failed" to the operator. The architecture specifies `asyncio.Lock` for all shared state mutations and allows up to 10s detection timeouts — these can interact badly during a burst alert scenario. There is no documented mitigation (e.g., immediate `interaction.defer()` before acquiring locks). | In `watchdog/discord/interactions.py`, all interaction handlers must call `await interaction.response.defer()` as their first statement before any lock acquisition or action execution. Document this as a mandatory pattern in the implementation patterns section or in a story acceptance criterion. |
| H2 | Coverage Gaps | Temporal mTLS cert provisioning is listed as an "open technical item" — "Need to verify if already in Vault." If the cert path does not exist in Vault, the Temporal client will fail at startup initialization and the entire pod will crash-loop. This is a dev-blocker: no integration test or MVP0 slice can proceed until the cert path is confirmed and the ESO ExternalSecret mapping is validated. | Add an explicit pre-dev task to confirm Vault path `terminus/watchdog/temporal-cert` and `terminus/watchdog/temporal-key` exist (or to create them). This story must complete before any Temporal-dependent detector story begins. |

### Medium

| # | Dimension | Finding | Recommendation |
|---|-----------|---------|----------------|
| M1 | Complexity and Risk | k8s Watch streams expire after approximately 5 minutes (WATCH_TIMEOUT or server-side lease expiry). The `kubernetes-asyncio` client does not auto-reconnect by default. A silently expired Watch stream means the watchdog stops receiving pod/node events while appearing healthy — the liveness probe and Prometheus gauge won't catch this. | Add an explicit reconnection loop pattern to `watchdog/detectors/kubernetes.py`: on `ApiException` (410 Gone) or stream closure, log a warning and re-subscribe. Add a `watchdog_k8s_watch_reconnects_total` counter metric. |
| M2 | Assumptions and Blind Spots | The write-action Approve flow logs `discord_user_id` but imposes no authorization check. Any server member who can see the alert message (i.e., anyone in the Discord server with access to `#platform-alerts`) can click Approve and trigger an ArgoCD sync or pod delete. The constitution (Article 9: Security-first) requires explicit authorization. | Define an ops allowlist: a `DISCORD_OPS_USER_IDS` env var (comma-separated Discord user IDs). Interaction handlers must check `interaction.user.id in config.ops_allowlist` before executing any write action; respond with an ephemeral "Unauthorized" message otherwise. Document in runbook. |
| M3 | Coverage Gaps | Write-action rate limiting is mentioned in the architecture security section but no mechanism is specified. Under a rapid repeated-approval scenario, the same ArgoCD app could be synced multiple times in quick succession. | Add a per-resource write-action cooldown in `watchdog/state.py`: a `last_action_map: dict[str, datetime]` that blocks a second write action on the same resource within a configurable `WRITE_ACTION_COOLDOWN_SECONDS` (default: 300). Enforce in `BaseAction` before execution. |

### Low

| # | Dimension | Finding | Recommendation |
|---|-----------|---------|----------------|
| L1 | Assumptions and Blind Spots | Loki auth is not addressed. The `loki.monitoring.svc.cluster.local` endpoint may require a bearer token or basic auth header in some Terminus Loki configurations. | Verify Loki auth requirements during MVP0 setup; add `LOKI_AUTH_TOKEN` secret field to ESO mapping as a precaution even if currently unused. |
| L2 | Complexity and Risk | Discord rate limiting at cluster-wide incident scale: if 6+ apps go OutOfSync simultaneously, the bot will attempt rapid sequential message sends. `discord.py` handles rate limits transparently with automatic backpressure, but message delivery delay could be 10-30s under heavy throttling. | Accept as-is for homelab scale. Document in runbook. If alert delivery latency becomes problematic, batch concurrent alerts into a single embed in a follow-up story. |

---

## Accepted Risks

| Finding | Rationale |
|---------|-----------|
| L2 — Discord rate limiting under incident scale | Single-operator homelab. Delivery delay < 30s is acceptable. Not worth architectural complexity. |
| Prometheus + Alertmanager dependency for H3 | Both services already run on Terminus. Circular dependency risk is low. Acknowledged in architecture. |

---

## Party-Mode Challenge

**Reina (SRE / Platform Engineer):** The runbook is listed as required but its contents aren't specified. When the k8s Watch stream silently expires — and it will — how does the on-call operator know detection is running but blind? The liveness probe and heartbeat won't catch this. You need a "detection loop alive" signal per source family, and it needs to be in the runbook with explicit recovery steps.

**Damien (Security Architect):** The write-action approval flow logs `discord_user_id` but doesn't define an authorization check. Any Discord server member can click Approve. For a bot that deletes pods and syncs ArgoCD apps, that's an untrusted actor with write access to your cluster. This is a Medium finding but it reads as High to me given Article 9 requirements.

**Sam (Implementation Lead):** The ArgoCD Option A diff strategy assumes `status.resources[*]` provides field-level diffs. In practice, pre-sync ArgoCD app objects expose per-resource sync status but not field-level diffs — those live in `status.operationState.syncResult` *after* a sync completes. Option A may permanently fall back to Option C. The MVP0 validation story must test this assumption against a live ArgoCD app in an OutOfSync state before the image-promotion classifier is built.

---

## Gaps You May Not Have Considered

1. **Write-action authorization**: any server member can click Approve. Define an ops allowlist before dev begins.
2. **k8s Watch reconnection**: the Watch stream expires silently. Reconnection logic and a reconnect-counter metric are needed.
3. **Discord Gateway intents**: the bot application must be configured in the Discord Developer Portal with correct privileged intents before deployment. This is a manual step — document in runbook.
4. **ArgoCD Option A validation scope**: if MVP0 confirms Option A doesn't provide field-level diff pre-sync, who owns scoping the `argocd-image-promotion` pattern deferral to MVP2? This should be an explicit story condition.
5. **Discord interaction `defer()` pattern**: all interaction handlers must defer immediately as their first statement. This is a cross-cutting implementation constraint — surface it in the patterns section or as a definition-of-done item in the interaction handler stories.

---

## Open Questions Surfaced

| # | Question | Disposition |
|---|----------|-------------|
| OQ1 | Does `status.resources[*]` in the ArgoCD REST API provide field-level diff data before a sync? | Validate in MVP0 — acceptance criterion required in story |
| OQ2 | Is Temporal mTLS cert already in Vault at `terminus/watchdog/temporal-cert`? | Confirm before story sprint begins; provisioning task if absent |
| OQ3 | Does Loki at `loki.monitoring.svc.cluster.local` require authentication? | Verify during MVP0 cluster setup |
| OQ4 | Has the Discord bot application been registered with correct Gateway intents in the Developer Portal? | Pre-dev task; document in runbook |
| OQ5 | What is the ops user allowlist for write-action authorization? | Architect (Todd) to specify — feeds `DISCORD_OPS_USER_IDS` config |
