# Sprint Plan — OmniRoute Gateway Migration

**Feature:** terminus-inference-omniroute-gateway-migration  
**Domain/Service:** terminus / inference  
**Track:** express  
**Phase:** expressplan  
**Owner:** Todd Hintzmann  
**Date:** 2026-05-25

---

## Sprint Overview

| Sprint | Theme | Key Deliverables | Risk |
|--------|-------|-----------------|------|
| Sprint 1 | Deploy + Wire | OmniRoute running in K8s, llamacpp provider registered, all five combos configured and manually verified | Medium — auth bridge design choice |
| Sprint 2 | Auth + Harden | Auth bridge wired end-to-end, contract tests green against OmniRoute, circuit breaker validated, batch-strict-fail confirmed | Medium — Vault JWT path |
| Sprint 3 | Cutover + Decommission | Ingress re-pointed to OmniRoute, cutover validation complete, terminus-inference-gateway decommissioned, Article 16 .todo reviewed | Low — rollback procedure verified |

**Estimated sprint size:** 3 sprints × ~3-4 stories each  
**Total story count:** 10 stories + 1 spike + 1 spike + 2 chores

---

## Epic: OmniRoute Gateway Migration

### Sprint 1 — Deploy + Wire

---

**Story 1.1: OmniRoute K8s Deployment**

*As a platform operator, I can access a self-hosted OmniRoute instance running inside the inference namespace so that inference traffic stays on-premises.*

**Acceptance Criteria:**
- OmniRoute pod runs in `inference` namespace with resource limits (cpu: 1000m, memory: 1Gi)
- PVC `omniroute-data-pvc` (1Gi, ReadWriteOnce) is provisioned and mounted at `/app/data`
- OmniRoute responds to `GET /health` from inside the cluster
- Image tag is pinned to a specific semver release (not `:latest`)
- K8s NetworkPolicy: allow llamacpp egress, allow api.openai.com:443, deny all other external egress
- OmniRoute config exported from data volume to committed YAML; round-trip restore validated (create config → export → delete volume → import → verify config matches)
- OmniRoute readiness probe configured to delay Ready until `/health` returns 200 (not just container start)

**Open Questions (must resolve before story completes):**
- What OmniRoute semver version to pin? (check `docker.io/diegosouzapw/omniroute` available tags and select latest stable)
- Is `gateway-vault-token` K8s secret the consumer-facing token or a service-account token? (impacts auth bridge design)
- OmniRoute config exported from data volume to committed YAML; round-trip restore validated (create config → export → delete volume → import → verify config matches)
- OmniRoute readiness probe configured to delay Ready until `/health` returns 200 (not just container start)

**Open Questions (must resolve before story completes):**
- What OmniRoute semver version to pin? (check `docker.io/diegosouzapw/omniroute` available tags and select latest stable)
- Is `gateway-vault-token` K8s secret the consumer-facing token or a service-account token? (impacts auth bridge design)
---

**Spike S-1.0: Auth Bridge Implementation Path (Sprint 1 — timebox 2h)**

*As a developer, I have confirmed whether Nginx ingress `auth-url` annotation can validate Vault JWT tokens and inject the OmniRoute API key so that the auth bridge implementation path is decided before Sprint 2 begins.*

**Spike Goal:** Validate Nginx `auth-url` against Vault `/v1/auth/token/lookup-self`. If accessible and functioning: document Nginx annotation as the confirmed implementation path. If not accessible or insufficient: confirm Go auth proxy as implementation path.

**Timebox:** 2 hours. Do not exceed; produce a decision document regardless of outcome.

**Output:** `docs/terminus/inference/terminus-inference-omniroute-gateway-migration/auth-bridge-decision.md` committed before Sprint 2 begins.

**Gate:** Story 2.1 may not begin until this spike output is committed and reviewed.

**Dependencies:** None (can start alongside Story 1.1)  
**Assigned:** Sprint 1

---


**Dependencies:** None  
**Depends On:** Nothing

---

---

**Spike S-1.0: Auth Bridge Implementation Path (Sprint 1 — timebox 2h)**

*As a developer, I have confirmed whether Nginx ingress `auth-url` annotation can validate Vault JWT tokens and inject the OmniRoute API key so that the auth bridge implementation path is decided before Sprint 2 begins.*

**Spike Goal:** Validate Nginx `auth-url` against Vault `/v1/auth/token/lookup-self`. If accessible and functioning: document Nginx annotation as the confirmed implementation path. If not accessible or insufficient: confirm Go auth proxy as implementation path.

**Timebox:** 2 hours. Do not exceed; produce a decision document regardless of outcome.

**Output:** `docs/terminus/inference/terminus-inference-omniroute-gateway-migration/auth-bridge-decision.md` committed before Sprint 2 begins.

**Gate:** Story 2.1 may not begin until this spike output is committed and reviewed.

**Dependencies:** None (can start alongside Story 1.1)  
**Assigned:** Sprint 1

---

**Story 1.2: llamacpp Provider Registration**
- OmniRoute combo config exported to committed YAML (validation that round-trip from Story 1.1 works with combo state)
- llamacpp queue-depth test: send 2 concurrent requests when `parallel_slots=1`; verify OmniRoute queues second request correctly and both complete (no dropped requests, no timeout for reasonable 60s wait)

*As a platform operator, I can register the llamacpp backend as a custom OpenAI-compatible provider in OmniRoute so that inference requests route to the local GPU worker.*

**Acceptance Criteria:**
- llamacpp registered in OmniRoute as provider `llamacpp-local` at `http://llamacpp.inference.svc.cluster.local:9090`
- `POST /v1/chat/completions` through OmniRoute returns a valid OpenAI-shaped response using llamacpp
- Provider registration is exported as committed config (not dashboard-only)
- No credentials required for llamacpp-local (internal endpoint)

**Dependencies:** Story 1.1  
**Depends On:** llamacpp service running in inference namespace

---

**Story 1.3: Route Profile Combos (All 5)**

*As a consumer, my existing route profile semantics (chat, code-completion, interactive, batch, triage) are preserved through OmniRoute combos so that no consumer changes are required at cutover.*

**Acceptance Criteria:**
- Five combos created: `combo-chat`, `combo-code`, `combo-interactive`, `combo-batch`, `combo-triage`
- Each combo configured per route-profiles manifest (routing strategy, timeout, retry policy, fallback)
- `combo-batch` uses strict routing with no fallback provider; returns 503 explicitly if llamacpp is unavailable
- `combo-triage` is priority with 60s timeout and 2048 max tokens
- All five combos return valid OpenAI-shaped responses for representative test requests via direct OmniRoute API call
- Route-profiles manifest committed at `docs/terminus/inference/terminus-inference-omniroute-gateway-migration/route-profiles.yaml`
- OmniRoute combo config exported to committed YAML (validation that round-trip from Story 1.1 works with combo state)
- llamacpp queue-depth test: send 2 concurrent requests when `parallel_slots=1`; verify OmniRoute queues second request correctly and both complete (no dropped requests, no timeout for reasonable 60s wait)

**Dependencies:** Story 1.2

---

### Sprint 2 — Auth + Harden

---

**Story 2.1: Vault JWT Auth Bridge**

*As an existing consumer, I can continue to use my Vault-issued JWT token to call the inference gateway endpoint so that I require no credential changes after the migration.*

**Acceptance Criteria:**
- Auth bridge validates incoming Vault JWT tokens using Vault's `/v1/auth/token/lookup-self` endpoint
- Valid JWT: request forwarded to OmniRoute with `Authorization: Bearer <omniroute-api-key>` header injected
- Invalid/expired JWT: bridge returns HTTP 401 with `{"error": "unauthorized"}` body; request does not reach OmniRoute
- OmniRoute API key is stored in Vault, injected as K8s secret; not committed to source control (Article 9)
- If implemented via Nginx ingress annotation: annotation config committed; no new service required
- If implemented via auth proxy service: proxy written in Go (Svc-7), TDD red-green (Org-7); unit test coverage ≥ 90% for JWT validation and header injection paths

**Dependencies:** Story 1.1  
**Unresolved:** Implementation path (Nginx annotation vs. auth proxy) — requires evaluating whether a Vault JWT validation endpoint is accessible to Nginx auth-url. **Must be resolved before story begins.**

---

**Story 2.2: Contract Test Suite Port to OmniRoute**

*As a developer, the existing contract test suite passes against the OmniRoute endpoint so that I have confidence the API contract is preserved before cutover.*

**Acceptance Criteria:**
- All existing contract tests in `terminus-inference-gateway` adapter suite pass when run against `http://omniroute.inference.svc.cluster.local:20128/v1`
- Test runner updated to target OmniRoute endpoint via environment variable
- No tests skipped or modified to pass — all existing assertions hold
- CI pipeline can execute tests against OmniRoute (not just terminus-inference-gateway)

**Dependencies:** Story 1.3, Story 2.1— Full (Org-3 finalization)**

*As a future maintainer, I can read the complete architecture documentation for the OmniRoute deployment so that all operational decisions (including the confirmed auth bridge implementation) are captured and auditable.*

**Acceptance Criteria:**
- `docs/terminus/inference/terminus-inference-omniroute-gateway-migration/architecture.md` updated with confirmed auth bridge implementation + rationale
- Covers complete K8s deployment topology, route profile manifest reference, observability wiring, Article 16 deviation decision
- Note: an architecture stub was produced as part of FinalizePlan to satisfy Org-3 before dev begins; this story finalizes it with Sprint 2 decisions
- Reviewed and approved before cutover proceeds

**Dependencies:** Spike S-1.0 resolved (auth bridge implementation choice known); architecture stub from FinalizePlan already committedest
- OmniRoute circuit breaker opens within the configured failure threshold
- BDD scenario: `Given llamacpp is down, When a chat request is sent via combo-chat, Then the request fails over to openai-fallback within circuit breaker timeout`
- Circuit breaker recovers (closes) within 30 seconds of llamacpp returning to healthy
- Alert `LlamaCppProviderDown` (or equivalent OmniRoute health alert) fires during downtime

**Acceptance Criteria (batch hard-fail):**
- BDD scenario: `Given llamacpp is down, When a batch request is sent via combo-batch, Then a 503 response is returned with {"error": "provider unavailable"}`
- Batch requests do NOT silently fall over to OpenAI under any circumstance

**Dependencies:** Story 1.3gracefully drained (existing long-polling connections, up to 300s timeout, allowed to complete) before ingress is re-pointed
- `terminus-inference-gateway` deployment scaled to 0 replicas (NOT deleted — held for rollback window)
- End-to-end smoke test passes: authenticated request through ingress → OmniRoute → llamacpp → response
- All named consumers verified: Watchdog, Kaylee, and Hermes each execute at least one successful request through OmniRoute before parallel window begins
- All five route-profile combos are reachable via the public ingress endpoint
- Rollback verified: scaling `terminus-inference-gateway` back to 1 and re-patching ingress restores previous behavior in < 5 minutes
- 24h parallel window starts only after all three named consumers verified
### Sprint 3 — Cutover + Decommission

---

**Story 3.1: Architecture Documentation — Full (Org-3 finalization)**

*As a future maintainer, I can read the complete architecture documentation for the OmniRoute deployment so that all operational decisions (including the confirmed auth bridge implementation) are captured and auditable.*

**Acceptance Criteria:**
- `docs/terminus/inference/terminus-inference-omniroute-gateway-migration/architecture.md` updated with confirmed auth bridge implementation + rationale
- Covers complete K8s deployment topology, route profile manifest reference, observability wiring, Article 16 deviation decision
- Note: an architecture stub was produced as part of FinalizePlan to satisfy Org-3 before dev begins; this story finalizes it with Sprint 2 decisions
- Reviewed and approved before cutover proceeds

**Dependencies:** Spike S-1.0 resolved (auth bridge implementation choice known); architecture stub from FinalizePlan already committed

---

**Story 3.2: Ingress Cutover + Parallel Window**

*As a consumer, my requests to `https://inference-gateway.trantor.internal/v1` are served by OmniRoute so that the migration is live.*

**Acceptance Criteria:** — Review Approval (Sprint 1)**

The `.todo` entry at `docs/terminus/.todo/2026-05-25-omniroute-external-tool-deviation.md` has been created. This chore is to complete the review and mark it `approved` before Sprint 2 begins.

**Sprint assignment:** Sprint 1  
**Reviewer:** Todd Hintzmann (sole operator)

**Acceptance Criteria:**
- `2026-05-25-omniroute-external-tool-deviation.md` status field updated from `pending-review` to `approved`
- Approved by: Todd Hintzmann
- Committed to control repo before Sprint 2 begins

**Gate:** Blocking — dev phase may not begin until this chore is complete and committed
- Rollback verified: scaling `terminus-inference-gateway` back to 1 and re-patching ingress restores previous behavior in < 5 minutes
- 24h parallel window starts only after all three named consumers verified

**Dependencies:** Story 2.2, Story 2.3, Story (Sprint 1)**

**Sprint assignment:** Sprint 1 (alongside Story 1.1)  
**Reviewer:** Todd Hintzmann

Add a Dependabot/Renovate rule to notify on new OmniRoute semver releases. Do not auto-update — notification only. This ensures the version pin is evaluated consciously on each upstream release.

**Acceptance Criteria:**
- Dependabot/Renovate config updated in `terminus.infra` to watch `diegosouzapw/omniroute` Docker image
- Notification-only (no auto-merge)
- Committed to terminus.infra before Story 1.1 completes

**Story 3.3: Decommission terminus-inference-gateway**

*AsTarget Repo Mapping

| Sprint | Stories | Target Repo |
|--------|---------|-------------|
| Sprint 1 | 1.1, 1.2, 1.3, Spike S-1.0 | `TargetProjects/terminus/infra/terminus.infra` |
| Sprint 2 | 2.1 (auth proxy path) | New service in `terminus.infra` or Nginx annotation config in `terminus.infra` |
| Sprint 2 | 2.2 | `TargetProjects/terminus/inference/terminus-inference-gateway` (contract test runner update) |
| Sprint 2 | 2.3 | `TargetProjects/terminus/infra/terminus.infra` |
| Sprint 3 | 3.1 | Control repo docs path |
| Sprint 3 | 3.2, 3.3 | `TargetProjects/terminus/infra/terminus.infra` |

---

## Unresolved Risks (FinalizePlan applied; remaining deferrals)

| ID | Risk | Owner | Resolution Path d from inference namespace Helm chart
- Helm release synced and ArgoCD reports healthy
- Deployment, service, and ingress-backend entries for terminus-inference-gateway removed
- `terminus-inference-gateway` source repo archived or marked deprecated
- No CI pipeline triggered for terminus-inference-gateway after decommission

**Dependencies:** Story 3.2 (cutover validated; parallel window elapsed ≥ 24h before deletion)

---

## Chores

**Chore C-1: Article 16 .todo Entry — Review Approval (Sprint 1)**

The `.todo` entry at `docs/terminus/.todo/2026-05-25-omniroute-external-tool-deviation.md` has been created. This chore is to complete the review and mark it `approved` before Sprint 2 begins.

**Sprint assignment:** Sprint 1  
**Reviewer:** Todd Hintzmann (sole operator)

**Acceptance Criteria:**
- `2026-05-25-omniroute-external-tool-deviation.md` status field updated from `pending-review` to `approved`
- Approved by: Todd Hintzmann
- Committed to control repo before Sprint 2 begins

**Gate:** Blocking — dev phase may not begin until this chore is complete and committed.

---

**Chore C-2: OmniRoute Version Pin Automation (Sprint 1)**

**Sprint assignment:** Sprint 1 (alongside Story 1.1)  
**Reviewer:** Todd Hintzmann

Add a Dependabot/Renovate rule to notify on new OmniRoute semver releases. Do not auto-update — notification only. This ensures the version pin is evaluated consciously on each upstream release.

**Acceptance Criteria:**
- Dependabot/Renovate config updated in `terminus.infra` to watch `diegosouzapw/omniroute` Docker image
- Notification-only (no auto-merge)
- Committed to terminus.infra before Story 1.1 completes

---

## Target Repo Mapping

| Sprint | Stories | Target Repo |
|--------|---------|-------------|
| Sprint 1 | 1.1, 1.2, 1.3, Spike S-1.0 | `TargetProjects/terminus/infra/terminus.infra` |
| Sprint 2 | 2.1 (auth proxy path) | New service in `terminus.infra` or Nginx annotation config in `terminus.infra` |
| Sprint 2 | 2.2 | `TargetProjects/terminus/inference/terminus-inference-gateway` (contract test runner update) |
| Sprint 2 | 2.3 | `TargetProjects/terminus/infra/terminus.infra` |
| Sprint 3 | 3.1 | Control repo docs path |
| Sprint 3 | 3.2, 3.3 | `TargetProjects/terminus/infra/terminus.infra` |

---

## Unresolved Risks (FinalizePlan applied; remaining deferrals)

| ID | Risk | Owner | Resolution Path |
| R-1 | Auth bridge implementation path (Nginx annotation vs. auth proxy) | Todd | Test Nginx `auth-url` with Vault JWT endpoint; decide before Sprint 2 begins |
| R-2 | OmniRoute config state is runtime-managed; GitOps strategy unconfirmed | Todd | Export config to committed YAML on day 1; validate round-trip restore in Sprint 1 |
| R-3 | llamacpp `parallel_slots=1` queue depth under concurrent OmniRoute routing | Todd | Load test during Sprint 1 proof-of-concept; adjust combo timeout if queue wait is observable |

These risks must be resolved and committed to FinalizePlan before sprint execution begins.
