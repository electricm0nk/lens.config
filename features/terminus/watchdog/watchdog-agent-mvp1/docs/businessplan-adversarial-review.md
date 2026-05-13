# BusinessPlan Adversarial Review — terminus-watchdog-agent

**Feature:** `terminus-watchdog-agent`
**Phase:** BusinessPlan
**Source:** phase-complete
**Date:** 2026-05-11 (rerun)
**Reviewer:** Lens Adversarial Review Gate
**Artifacts Reviewed:** `prd.md`, `ux-design.md`
**Predecessor Context:** `product-brief.md`, `research.md`, `brainstorm.md`, `preplan-adversarial-review.md`
**Prior Verdict:** FAIL (2026-05-11 — empty PRD + scope inconsistency)

---

## Verdict

**PASS-WITH-WARNINGS**

The PRD body is now complete and the scope decision (MVP1 write actions) is documented consistently across both artifacts. No critical findings remain. Three high-risk findings require resolution before TechPlan begins; they are documented here and tracked in the PRD open questions. Three medium findings are documented and do not block the phase gate.

---

## Prior Critical Findings — Resolution Status

### [C1] PRD body was empty — RESOLVED
`prd.md` now contains full functional requirements (§4), interaction requirements (§6), NFRs (§9), 12 acceptance criteria (§12), success criteria (§11), out-of-scope table (§10), and an MVP0 slice definition (§13).

### [C2] Scope inconsistency (read-only vs. write actions) — RESOLVED
PRD §2 explicitly documents the scope decision: MVP1 includes real write actions (Approve = ArgoCD sync, Temporal terminate, pod delete). The goal field is updated. Both artifacts now agree.

---

## High Risk Findings

### [H1] ArgoCD manifest diff API unvalidated — classification collapses if unavailable

The `argocd-image-promotion` vs `argocd-live-drift` distinction is the primary differentiator of the product. Both the PRD (§4.3) and ux-design.md acknowledge this as an open question. However, neither artifact defines a fallback behavior in concrete terms.

If the ArgoCD manifest diff API is unavailable, the two patterns collapse into `argocd-out-of-sync`. The ux-design currently has two distinct alert templates. The PRD notes "both patterns collapse into a single argocd-out-of-sync alert at High severity" but does not define:
- Which channel `argocd-out-of-sync` would route to (High → `#platform-alerts` or medium-degraded?)
- Whether `argocd-image-promotion` as a pattern is fully removed from scope or becomes unreachable dead code
- The merged alert template content

**Required action before TechPlan:** Validate the ArgoCD manifest API. If it cannot distinguish image-tag changes from config drift, update both prd.md and ux-design.md with the merged fallback pattern definition before architecture is authored.

---

### [H2] Discord interaction endpoint security model absent

`prd.md` §12 AC-12 specifies Discord signature validation as a functional requirement. §9 NFR table specifies "Discord payload signature validation (ED25519) required." §6.1 calls out the security requirements. However, neither artifact addresses:
- Which public ingress mechanism will be used (Cloudflare Tunnel, existing wildcard cert ingress, ngrok, etc.)
- How the bot token and interaction public key are stored (Vault? k8s Secret?)
- The specific signature validation library to use for Python (e.g., `PyNaCl` or `cryptography`)
- Rate limiting implementation: "max 3 Approve actions per resource per 5-minute window" (PRD §6.1) — where does this state live?

The security model must be defined in `architecture.md` before any interaction endpoint code is written. With real write actions enabled, a signature validation bug now has a production blast radius.

**Required action in TechPlan:** Architecture must specify ingress path, secret storage, signature validation approach, and rate limit state design.

---

### [H3] Watchdog self-health observer is architecturally undefined

PRD §7.2 states: "An external health observer (architecture to define) must detect when the heartbeat stops." This is deferred cleanly to TechPlan. However, the absence of this design creates a delivery risk — MVP1 could ship a watchdog with a heartbeat that no one monitors.

The PRD gives no constraints on acceptable solutions. Three viable options exist (k8s liveness probe + exporter, dead man's switch webhook, separate lightweight process) with meaningfully different complexity. Without a direction from the PRD, the architect has no guardrails.

**Required action in TechPlan:** Architecture must select and spec the external self-health observer mechanism. The PRD should be updated to note the selected approach direction (simple preferred: k8s liveness probe + Discord-alerting sidecar or exporter is lowest complexity).

---

## Medium Risk Findings

### [M1] ConfigMap schema completeness

PRD §8.1–§8.4 defines four ConfigMaps. The YAML examples are well-formed. However:
- `temporal_per_workflow_overrides` in §8.1 uses a flat map of `WorkflowType: threshold_minutes`. The detection logic must enumerate workflow types via Temporal Visibility API — the override lookup must match exactly. A mismatch in workflow type name (e.g., `SeedSecrets` vs `fourdogs-catalog-trigger/SeedSecrets`) means the override silently never applies. The architecture must define the exact key format.
- §8.2 `high_priority_windows` uses an RFC 5545 RRULE string for `schedule`. Parsing RRULE in Python requires the `python-dateutil` or `icalendar` library — this dependency must be captured in the architecture.

---

### [M2] Rate limit state for Approve actions

PRD §6.1 states "max 3 Approve actions per resource per 5-minute window." This rate limit state must live somewhere. In-memory is simplest but is reset on restart. A restarted watchdog could re-execute Approve actions at full rate if an operator was mid-triage during the restart. This is a mild race condition but worth a design note in the architecture.

---

### [M3] Temporal gRPC namespace access unverified

PRD open questions list this. The watchdog will run in a dedicated namespace `terminus-watchdog`. Temporal service is at `temporal.terminus.svc.cluster.local:7233`. If a NetworkPolicy exists in the Temporal namespace blocking cross-namespace gRPC, the entire Temporal detection family fails silently. Verification must happen before TechPlan locks the deployment topology.

---

## Party-Mode Challenge Round

Three planning perspectives challenge the complete businessplan artifact set:

---

**Dr. Elena Vasquez (Security Architect):**
You're building an agent that can delete production pods and terminate workflows. The threat model matters here. I see signature validation as a requirement. What I don't see: what happens on a restart when the interaction endpoint briefly has no valid public key loaded? Does the startup sequence guarantee the key is available before the first Discord interaction can arrive? And your rate limit — in-memory, reset on restart — means an operator who got rate-limited before the watchdog restarted gets a fresh window immediately. For prod pod deletes, that's worth a note. The architecture needs to address startup state ordering.

---

**James Chen (Platform Engineer):**
I'm happy the MVP0 slice is there. Three patterns, 2 weeks. But I want to stress-test the delivery path: the Discord bot needs to be provisioned, the interaction endpoint needs to be publicly accessible, the ArgoCD token needs to be provisioned, and the Temporal gRPC access needs to be verified — all before MVP0 demo. That's 4 external dependencies you can't control. What's the build-vs-stub plan if any one of them blocks week 1? Does the architecture define a local test harness so development isn't blocked on infra provisioning?

---

**Maria Santos (QA Engineer):**
12 acceptance criteria, but no failure-mode acceptance criteria. AC-3 tests Approve when the sync succeeds. What does the operator see if `POST /api/v1/applications/{name}/sync` returns 500? AC-5 tests Approve when pod delete succeeds. What if the pod is gone by the time the delete fires (already recovered)? The PRD §6.4 says "Action failed: Resolution field appended with error detail; buttons re-enabled for retry." That's a good start. But the architecture needs to define the specific error messages for each failure class so the UX is consistent, not generic "Action failed" for everything.

---

### Blind-Spot Challenge Questions

1. **Startup state ordering:** What guarantees the Discord interaction endpoint is not accepting requests before the signature validation key is loaded? Is there a readiness probe / startup probe on the interaction HTTP server?

2. **Local dev / test harness:** How will you test the ArgoCD sync, Temporal terminate, and pod delete actions locally without a live cluster? Is there a mock/stub mode for the three external action APIs?

3. **Detection cycle ordering:** During a poll cycle, if ArgoCD is temporarily unavailable (503), does the watchdog fire `argocd-*` alerts or suppress them? The PRD covers "3 consecutive failures" for a degradation alert but not what happens on cycle 1 or cycle 2.

4. **Temporal zombie — resolved detection:** When the operator presses Approve and the workflow is terminated, how does the watchdog know to update the alert to Resolved? It needs to re-poll or receive confirmation from the Temporal SDK response. The detection loop and the action outcome must be reconciled.

5. **Alert fan-out:** If `k8s-node-notready` fires for a node, and then multiple pods go into `CrashLoopBackOff` because of that node's eviction, will the watchdog fire one alert or N alerts (one per pod)? The deduplication policy doesn't address cascading events from a single root cause.

---

## Required Actions Before TechPlan

| # | Action | Severity |
|---|--------|---------|
| H1 | Validate ArgoCD manifest diff API; update prd+ux-design with fallback if unavailable | High |
| H2 | Define ingress, secret storage, signature validation approach in architecture.md | High |
| H3 | Select external self-health observer mechanism; document in architecture.md | High |
| M1 | Define temporal_per_workflow_overrides key format; add icalendar/dateutil dependency note | Medium |
| M2 | Address rate limit state on restart in architecture design note | Medium |
| M3 | Verify Temporal gRPC cross-namespace access before locking deployment topology | Medium |

**Feature:** `terminus-watchdog-agent`
**Phase:** BusinessPlan
**Source:** phase-complete
**Date:** 2026-05-11
**Reviewer:** Lens Adversarial Review Gate
**Artifacts Reviewed:** `prd.md`, `ux-design.md`
**Predecessor Context:** `product-brief.md`, `research.md`, `brainstorm.md`, `preplan-adversarial-review.md`

---

## Verdict

**FAIL**

Two critical findings block phase completion. The PRD has no body content — it is a frontmatter shell with no functional requirements, acceptance criteria, or success definition. The UX design introduces a material MVP1 scope change (real interactive actions replacing the "read-only" product brief constraint) without the scope revision being documented, reconciled, or approved anywhere in the artifact set. Neither finding is addressable by warnings — both block the TechPlan handoff.

---

## Critical Findings

### [C1] PRD body is empty — no requirements exist

`prd.md` contains valid frontmatter (`doc_type: prd`, goal, vision, open questions) and 4 lines of header content. The document body has no functional requirements, no non-functional requirements, no out-of-scope definition, no acceptance criteria, no success metrics, and no user story set.

The `validate-phase-artifacts.py` script passes because it detects the `doc_type: prd` field in frontmatter — it does not validate content. A hollow PRD shell is not a PRD.

**Impact:** The TechPlan phase requires the PRD as its primary input contract. An empty PRD means architecture will be authored against the product-brief alone, which is a PrePlan artifact — this is a phase regression, not a handoff. All epics and story authoring in DevProposal will also lack a requirements foundation.

**Required action:** Complete `prd.md` with a full requirements body before re-running the businessplan review gate. Required sections at minimum:
- Functional requirements (per detection pattern and interaction type)
- Non-functional requirements (latency, availability, maintainability, RBAC)
- Out-of-scope definition
- Success criteria (quantified, measurable)
- Acceptance criteria per major capability
- CrashLoopBackOff suppression policy (flagged in preplan adversarial review H2)
- Maintenance window mechanism (flagged in preplan adversarial review M1)
- Cold-start suppression behavior
- Alert fatigue / deduplication policy

---

### [C2] MVP1 scope revision introduced in UX design is unreconciled

`ux-design.md` Section 1 documents a "MVP1 Scope Revision" stating that **real write actions are enabled in MVP1**: Approve triggers `argocd app sync`, Temporal workflow termination, and `kubectl delete pod`. This directly contradicts:
- `product-brief.md`: "Read-only, always. MVP1 has no write operations."
- `product-brief.md`: "The watchdog is not a fixing system."
- `prd.md` frontmatter goal: "Lightweight read-only watchdog agent"

The scope revision is noted as open in the ux-design frontmatter (`open_questions`). It is not documented in any decision log, not reflected in the PRD goal statement, and not approved in any planning artifact.

**Impact:** This is not a cosmetic inconsistency. Transitioning the watchdog from read-only observer to active remediator changes:
- The security model (the system can now delete production pods and terminate workflows)
- The architecture (action endpoints, authentication, idempotency, re-verification gates)
- The testing strategy (action consequences must be tested safely)
- The RBAC surface (ServiceAccount needs write permissions)
- The risk profile (a bug in the watchdog can now cause outages, not just missed alerts)

**Required action before TechPlan:** Make an explicit scope decision documented in the PRD:
- **Option A — Keep read-only (MVP1):** Revert ux-design.md to scaffolded buttons; mark interactive actions as MVP2 scope; update product-brief risk table with cold-start suppression note
- **Option B — Enable write actions (MVP1):** Update product-brief principles; add "interactive remediation" to the PRD as a first-class capability with safety gate requirements; remove the "read-only, always" principle

This decision must be made by the user and reflected in the PRD before the review gate can re-run.

---

## High Risk Findings

### [H1] ArgoCD diff classification still unverified — UX design depends on it

The preplan adversarial review flagged [C1]: the `argocd-image-promotion` vs `argocd-live-drift` classification requires manifest diff capability from the ArgoCD REST API. This has not been validated against the running instance.

The ux-design.md provides separate templates for `argocd-image-promotion` and `argocd-live-drift` with distinct Diagnosis field content (e.g., "Manifest diff shows config change not present in git HEAD"). This content assumes successful manifest comparison is possible.

If `/api/v1/applications/{name}/manifests` cannot be used to reliably distinguish image-tag changes from config drift, the classification collapses and the UX design for these two patterns must be unified into a single `argocd-out-of-sync` alert.

**Required action:** Validate the ArgoCD manifest diff API before TechPlan. If classification is not achievable, update ux-design.md to merge both patterns into a single `argocd-out-of-sync` alert.

---

### [H2] Public interaction endpoint security model absent — now more critical with write actions

The preplan adversarial review flagged [H1]: Discord interaction endpoint must be publicly reachable from Discord's servers. The homelab runs on a private network. This was unaddressed.

With real write actions enabled in MVP1 (C2), the security implications escalate materially:
- The interaction endpoint now executes ArgoCD syncs, Temporal workflow terminations, and pod deletes
- Discord does sign interaction payloads (ED25519 signature on the request body), but this validation must be implemented correctly or any caller can trigger production actions
- The ingress path (Cloudflare Tunnel, existing wildcard cert, etc.) must be selected and documented

**Required action before TechPlan:** Select the ingress mechanism; confirm Discord payload signature validation is implemented (not just described); document the full security model for the interaction endpoint in `architecture.md`.

---

## Medium Risk Findings

### [M1] Alert posture for High severity is ambiguous

The PRD frontmatter states `alert-posture: passive (notify during waking hours; no 3am pages for MVP1)`. The ux-design key decisions state `@mention Todd for High severity only` with no time-of-day filter. These are contradictory.

`k8s-node-notready` at 3am is a High severity alert. Under the `passive` posture, it should not @mention. Under the ux-design rule, it should. This is an unresolved policy decision with real operational impact.

**Required action:** Define the alert posture explicitly in the PRD: either (a) `@mention` fires for High at all times; or (b) `@mention` fires only within a defined waking hours window (configurable); or (c) `@mention` fires always for `argocd-order-day-unsync` and `k8s-node-notready` (infrastructure class) but respects quiet hours for other High patterns.

---

### [M2] CrashLoopBackOff suppression policy still underspecified

The ux-design.md defines "suppress if pod recovers within 2 min" but the adversarial review from preplan ([H2]) also required handling for oscillating pods (crash every 90s). The ux-design states: "Oscillating pods (crash cycle < 2 min) fire on every crash-then-recover cycle after the first alert." This creates an alert storm for an oscillating pod.

The stated behavior — alert on every recovery cycle — is the opposite of deduplication and will generate high-frequency noise for pathological cases. This needs an explicit policy decision.

**Required action in PRD:** Define the CrashLoopBackOff escalation policy:
- First event: suppressed if pod recovers within 2 min
- Second event in same pod within the cooldown window: fire alert
- Oscillating pod (> N crashes in M minutes): escalate severity; one alert per escalation, not per cycle

---

### [M3] `temporal-postgres-connectivity` log access path still unscoped

Preplan adversarial review M4 flagged this pattern: the detection source is "Temporal worker logs" but the watchdog is not a log aggregation system. The ux-design.md shows a template for this pattern without addressing how the watchdog accesses those logs.

The architecture section of ux-design.md notes "Loki API for temporal-postgres-connectivity detection" as an open architecture requirement — but Loki is in-cluster and may have its own auth requirements.

**Required action in architecture.md:** Scope the Loki query path: Loki HTTP API address, auth method, query syntax for Temporal worker error logs, polling frequency. If Loki is not available, drop this pattern to MVP2.

---

### [M4] High-priority window ConfigMap format undefined

The ux-design.md introduces a `high_priority_windows` ConfigMap key for the FourDogs order day pattern, but provides no schema. This was also flagged in preplan [M2] as a cross-concern violation.

**Required action:** Define the ConfigMap schema in the PRD. At minimum: `name`, `cron_or_window` (ISO 8601 interval or cron expression), `affected_namespaces`, `escalation_behavior` (bypass-cooldown vs. severity-upgrade vs. extra-mention).

---

## Low Risk / Notes

### [L1] Watchdog missed-heartbeat alert requires external observer — dependency not documented
The ux-design section 11 states: "the k8s liveness probe to notify Discord externally — this is a self-health design requirement." The mechanism for doing this is unspecified and is an open architecture dependency. Add to `architecture.md` requirements.

### [L2] `temporal-postgres-connectivity` pattern duration field is ambiguous
The ux-design template shows `Duration: 5 min` for this pattern. Duration of what — the log error window, the polling interval, or elapsed time since first error? Define the duration semantics for log-derived patterns.

### [L3] `k8s-node-notready` has no Approve button — but @mention fires
`k8s-node-notready` is High severity, gets `@mention`, but has no `Approve` action. The operator is pinged but has only `Investigate` available. This is correct behavior but should be explicitly documented in the PRD as intentional ("no automated action for node failures in MVP1").

### [L4] Investigate deep-links require internal DNS resolution
Investigate URLs (ArgoCD: `https://argocd.terminus.internal/...`, Temporal: `https://temporal.terminus.internal/...`) are internal-only. Discord ephemeral reply sends these to the Discord client, which runs on the operator's machine. The operator must have access to `*.terminus.internal` DNS (VPN, split-horizon DNS, or SSH port-forward). This is an operational assumption that should be noted in the PRD.

---

## Party-Mode Challenge Round

Three planning perspectives challenge the businessplan artifacts:

---

**Sarah Chen (Product Manager):**
The PRD goal says "read-only watchdog agent" but the UX says it approves syncs and terminates workflows. You're building two different products. A read-only observer and an interactive remediator have different RTO/RCA expectations, different testing bars, and different operator trust requirements. Before anyone codes a single line, you need one coherent goal statement that the whole team agrees on. Which product are you actually building?

Also — you have 11 detection patterns and real write actions. That's a very large MVP1. The preplan adversarial review suggested a 3-pattern MVP0 slice. Where is the MVP0 vs MVP1 line drawn in your PRD? What ships in sprint 1?

---

**Marcus Williams (Security Engineer):**
A publicly exposed HTTP endpoint that receives Discord payloads and deletes production pods — this is a high-value attack surface. You mentioned Discord signs payloads. Good. But:
1. What happens if the signature validation has a bug? You're deleting pods on unauthenticated requests.
2. What is the rate limit on Approve actions? Can someone @mention-spam the bot to trigger repeated pod deletes?
3. The Temporal workflow termination action uses a hardcoded reason string `watchdog-zombie-terminate`. What prevents someone who knows this string from forging a termination request outside of Discord?
4. The watchdog runs in a dedicated namespace with cross-namespace write permissions. What is the blast radius if the watchdog pod is compromised?

None of these are documented in the artifact set. A security threat model for the interaction endpoint must be in the architecture before TechPlan.

---

**Alex Rivera (Platform SRE):**
I want to know what happens when the watchdog itself is the problem. Section 12 says cold-start suppression waits 3 minutes before alerting. Good. But:
- What if the watchdog restarts in the middle of a real incident? Condition is active, suppression window fires, operator gets nothing for 3 minutes.
- What if the watchdog's ArgoCD API token expires? Silence. Operator thinks everything is fine.
- What if Loki is down and temporal-postgres-connectivity can't poll? Same problem.

You have exactly one mechanism for watchdog self-health: the heartbeat message. But what triggers the "Watchdog: Heartbeat Missed" alert? Something external must send it. Have you designed that external observer? If not, you have a watchdog watching everything except itself.

---

### Blind-Spot Challenge Questions for the Operator

Before `prd.md` is completed, answer these:

1. **Scope decision:** Are real write actions (ArgoCD sync, workflow terminate, pod delete) in MVP1, or are they MVP2? Pick one and commit — the PRD goal, architecture, security model, and testing strategy all depend on this answer.

2. **MVP0 cut:** If you had to demo a working watchdog in 2 weeks, which 2 detection patterns would you implement first and why?

3. **3am alert policy:** `k8s-node-notready` at 3am — do you want to be woken up? What about `temporal-zombie-critical`? Define your quiet hours policy explicitly.

4. **Security model for write actions:** If Discord payload signature validation fails (implementation bug), what is the blast radius? Is `kubectl delete pod` in a production namespace acceptable as a worst-case? What rate-limit or confirmation gate would give you comfort?

5. **External watchdog observer:** If the watchdog pod goes down and stays down, how do you find out? Answer this before committing to the MVP1 heartbeat design.

---

## Required Actions Before Re-Running Gate

| # | Action | Severity | Owner |
|---|--------|---------|-------|
| C1 | Complete `prd.md` body with full requirements, acceptance criteria, and success metrics | Critical | BusinessPlan |
| C2 | Make explicit scope decision: read-only vs. write-action MVP1; update PRD goal and product-brief accordingly | Critical | BusinessPlan |
| H1 | Validate ArgoCD manifest diff API against running instance; update ux-design.md if classification is not achievable | High | Pre-TechPlan |
| H2 | Select public ingress mechanism for Discord interaction endpoint; document security model (signature validation, rate limiting) | High | TechPlan |
| M1 | Define alert posture (waking hours filter vs. always-on for High severity); document in PRD | Medium | BusinessPlan |
| M2 | Define CrashLoopBackOff escalation policy for oscillating pods in PRD | Medium | BusinessPlan |
| M3 | Scope Loki query path for `temporal-postgres-connectivity` or drop to MVP2 | Medium | TechPlan |
| M4 | Define high-priority-windows ConfigMap schema in PRD | Medium | BusinessPlan |
