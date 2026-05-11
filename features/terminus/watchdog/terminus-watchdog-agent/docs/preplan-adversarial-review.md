# PrePlan Adversarial Review — terminus-watchdog-agent

**Feature:** `terminus-watchdog-agent`  
**Phase:** PrePlan  
**Source:** manual-rerun  
**Date:** 2026-05-10  
**Reviewer:** Lens Adversarial Review Gate  
**Artifacts Reviewed:** `brainstorm.md`, `research.md`, `product-brief.md`

---

## Verdict

**PASS-WITH-WARNINGS**

Two critical findings must be resolved before TechPlan. The PrePlan phase may proceed to BusinessPlan, but these findings must be addressed in the PRD and architecture phases. No critical finding blocks this phase gate — both are resolvable through continued discovery in BusinessPlan/TechPlan.

---

## Critical Findings

### [C1] ArgoCD diff classification mechanism is unverified

The `argocd-image-promotion` vs `argocd-live-drift` classification is the core differentiator in the product brief. However, `GET /api/v1/applications` returns sync status only — not diff content. Actual diff resolution requires either `GET /api/v1/applications/{name}/manifests` (rendered manifest comparison) or a separate git query to compare recent commits against the live config.

Neither path was validated in the research phase. If field-level diff is unavailable via the ArgoCD API, the core classification collapses.

**Required action before TechPlan:** Validate ArgoCD manifest diff API (`/manifests` endpoint) against the running ArgoCD instance. Confirm whether image tag changes can be detected without git access. Document the chosen diff strategy in `architecture.md`.

---

### [C2] Temporal zombie detection: attempt count vs. elapsed time proxy

The research states that attempt count is not a native Temporal Visibility field and recommends using elapsed time as a proxy. However, the product brief's detection table retains attempt-count thresholds (`temporal-zombie-activity` > 50, `temporal-zombie-critical` > 500). These are different signals — an elapsed-time proxy and an attempt-count threshold are not interchangeable.

**Required action before TechPlan:** Reconcile the detection model. Either:
- (A) Drop attempt-count patterns; replace with elapsed-time-based detection with per-workflow-type thresholds; or
- (B) Commit to per-workflow history polling (`GetWorkflowExecutionHistory`) with an explicit cost assessment and polling frequency decision.

Document the chosen approach in `architecture.md`.

---

## High Risk Findings

### [H1] Discord interaction endpoint requires public HTTPS ingress — not addressed

Discord sends button interaction payloads via HTTP POST to a developer-configured HTTPS URL. This URL must be publicly reachable from Discord's servers. The Terminus homelab is on a private network. The brief does not address:
- Ingress mechanism (Cloudflare Tunnel, ngrok, public LoadBalancer, existing wildcard cert ingress)
- FQDN assignment
- TLS provisioning

**Required action before TechPlan:** Evaluate and select the public ingress path for the interaction endpoint. This is a deployment topology decision with security implications — must appear in `architecture.md`.

---

### [H2] Alert fatigue suppression is underspecified for CrashLoopBackOff

The brief proposes "suppress if pod recovers within 2 min." This is underspecified:
- What constitutes "recovered" (Running? Ready? No restarts for N minutes)?
- What happens to pods that oscillate (crash every 90s)?
- How does cooldown interact with the first-alert guarantee?

**Required action:** Define the CrashLoopBackOff suppression policy in the PRD as a functional requirement. Include the oscillation edge case explicitly.

---

### [H3] Temporal gRPC access from watchdog namespace is unverified

The Temporal SDK requires gRPC access to `temporal.terminus.svc.cluster.local:7233`. If the watchdog runs in a dedicated namespace (e.g., `terminus-watchdog`), a NetworkPolicy may block this. Unverified pre-TechPlan means this could break the deployment topology.

**Required action before TechPlan:** Verify gRPC connectivity from a test pod in the proposed watchdog namespace. Confirm NetworkPolicy allows cross-namespace access. Document in `architecture.md`.

---

## Medium Risk Findings

### [M1] Maintenance window annotation mechanism undefined

The brainstorm safety rule and product brief both reference "maintenance window annotations." No spec exists for: what creates the annotation, what resource it lives on, and what the watchdog does when a window is active.

**Required action:** Define the maintenance window mechanism in the PRD. At minimum: ConfigMap-based schedule configuration with time-range and affected-namespace fields.

---

### [M2] FourDogs business context baked into a platform tool

The `argocd-order-day-unsync` pattern hardcodes FourDogs Sunday order-day logic into a Terminus platform component. This is a cross-concern violation.

**Required action:** Replace hardcoded pattern with a configurable calendar/window config (e.g., `high-priority-windows` ConfigMap). The watchdog reads config; FourDogs specifics live in config, not code.

---

### [M3] Watchdog self-health monitoring is absent

A silent watchdog failure (pod crash, network partition) gives a false sense of safety. The brief requires the watchdog to not depend on monitored systems — but says nothing about who monitors the watchdog.

**Required action:** Add a watchdog liveness mechanism to the PRD. Minimum viable: a periodic Discord heartbeat message. Prefer a separate, lightweight health endpoint plus k8s liveness probe.

---

### [M4] `temporal-postgres-connectivity` log-parsing mechanism is unscoped

The pattern is sourced from "Temporal worker logs" — but the watchdog is not a log aggregation system. Accessing logs from a different pod requires log forwarding (Loki), sidecar, or metrics scraping. The detection path is not defined.

**Required action:** Either define the log access mechanism (Loki query via HTTP API is the most likely path given the Terminus stack) or drop this pattern from MVP1 and scope it to MVP2 when a log query path exists.

---

## Low Risk / Notes

### [L1] Discord bot provisioning not in deliverables table
The new Discord bot registration and Vault/k8s Secret setup are real work items. They should appear in the epics. The deliverables table in `product-brief.md` should note this as an infra prerequisite.

### [L2] Success metrics lack a formal baseline
"Operator diagnostic time < 2 min" is a good target but the baseline is described informally. Formalize current-state baseline in the PRD so the metric is measurable at retrospective.

### [L3] `secret-shared-detection` deferred without documented research gap
This pattern was introduced in brainstorm but never researched. Should be explicitly noted as "MVP2: Vault API research required" rather than silently dropped.

---

## Party-Mode Challenge Summary

Three planning perspectives challenged the artifact set:

- **Business Analyst:** Alert content must be readable in <90 seconds per Discord message. Success criterion "diagnostic time < 2 min" depends on message design, not just detection speed. Draft a sample alert message for `temporal-zombie-critical` as a PRD acceptance criterion.
- **Architect:** Cold-start alert storm risk. On watchdog restart, in-memory dedup is empty — the watchdog will re-alert every known problem. Cold-start suppression strategy (N-minute grace window on startup) must be specified.
- **Scrum Master:** MVP1 scope is large (11 patterns, Discord bot, RBAC, ingress, Vault, Temporal, ArgoCD). Prioritize a 3-pattern MVP0 slice before committing to full MVP1 scope. Candidate first slice: `k8s-crashloopbackoff` + `argocd-live-drift` + `temporal-zombie-critical`.

---

## Blind-Spot Questions for Operator Sign-Off

Before proceeding to BusinessPlan:

1. **ArgoCD diff:** Can `GET /api/v1/applications/{name}/manifests` return the diff needed for image-promotion classification? Test against your running instance before PRD authoring.
2. **Public ingress:** How will Discord reach the interaction endpoint? Does an existing ingress path (e.g., Cloudflare or existing wildcard cert) cover this, or is new infra required?
3. **Cold-start policy:** Should the watchdog suppress alerts for the first N minutes after restart to avoid re-alarming known problems?
4. **Zombie proxy:** What elapsed-time threshold constitutes a zombie for each workflow type in your system? (`SeedSecrets`: ? hours, `RunSmokeTest`: ? hours, `ReleaseWorkflow`: ? hours)
5. **MVP0 cut:** If you had to ship 3 detection patterns first (3 weeks), which 3 and in what order?

---

## Required Pre-TechPlan Actions

| # | Action | Severity | Owner |
|---|---|---|---|
| C1 | Validate ArgoCD manifest diff API; select diff strategy | Critical | TechPlan |
| C2 | Reconcile zombie detection model (elapsed-time vs. attempt-count) | Critical | TechPlan |
| H1 | Select and document public ingress path for Discord interaction endpoint | High | TechPlan |
| H2 | Define CrashLoopBackOff suppression policy including oscillation edge case | High | PRD |
| H3 | Verify Temporal gRPC access from watchdog namespace | High | TechPlan |
| M1 | Define maintenance window annotation mechanism | Medium | PRD |
| M2 | Replace FourDogs order-day hardcode with configurable calendar config | Medium | PRD |
| M3 | Add watchdog self-health / liveness mechanism | Medium | PRD |
| M4 | Define `temporal-postgres-connectivity` log access mechanism or defer to MVP2 | Medium | PRD |
