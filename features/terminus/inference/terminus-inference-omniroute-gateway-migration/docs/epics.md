# Epics — OmniRoute Gateway Migration

**Feature:** terminus-inference-omniroute-gateway-migration  
**Domain/Service:** terminus / inference  
**Track:** express  
**Generated:** 2026-05-25

---

## Epic 1: OmniRoute Deployment and Wiring (Sprint 1)

**Goal:** Deploy OmniRoute self-hosted in the `inference` K8s namespace, register the llamacpp provider, configure all five route profile combos, and confirm the auth bridge implementation path via a timebox spike.

**Deliverables:**
- OmniRoute running with resource limits, NetworkPolicy, readiness probe, and PVC
- llamacpp registered as `llamacpp-local` provider
- Five combos: chat, code, interactive, batch (strict), triage
- route-profiles.yaml committed
- Spike S-1.0 auth-bridge-decision.md committed
- Chore C-1: Article 16 .todo approved
- Chore C-2: OmniRoute version pin automation in Renovate

**Success Criteria:**
- All five combo types accept and return valid OpenAI-shaped responses
- OmniRoute config round-trip restore validated
- Auth bridge implementation path confirmed (Nginx or Go proxy)
- Article 16 .todo status = `approved` before Sprint 2

**Target Repo:** `TargetProjects/terminus/infra/terminus.infra`

---

## Epic 2: Auth Bridge and Hardening (Sprint 2)

**Goal:** Wire Vault JWT validation through the auth bridge into OmniRoute, port all contract tests to run against OmniRoute, and validate the circuit breaker + batch hard-fail behavior.

**Deliverables:**
- Auth bridge wired (Nginx annotation or Go proxy per Spike S-1.0 decision)
- Contract test suite ported to OmniRoute endpoint and passing
- Circuit breaker validation: combo-chat fails over to OpenAI on llamacpp down
- Batch hard-fail: combo-batch returns 503, never silently routes to OpenAI

**Success Criteria:**
- Vault JWT token rejected → 401 before reaching OmniRoute
- Valid Vault JWT token forwarded to OmniRoute with injected API key
- All existing contract tests pass against OmniRoute
- BDD scenarios green for circuit breaker and batch hard-fail

**Target Repos:**
- `TargetProjects/terminus/infra/terminus.infra` (auth bridge config)
- `TargetProjects/terminus/inference/terminus-inference-gateway` (contract test runner update)

---

## Epic 3: Cutover and Decommission (Sprint 3)

**Goal:** Cut over live ingress traffic to OmniRoute, validate all consumers through the new path, run the 24h parallel window, and remove the legacy gateway after all named consumers confirm success.

**Deliverables:**
- architecture.md finalized with confirmed auth bridge implementation
- Ingress re-pointed to OmniRoute; terminus-inference-gateway scaled to 0 (held for rollback)
- All three named consumers (Watchdog, Kaylee, Hermes) verified through OmniRoute
- 24h parallel window elapsed; terminus-inference-gateway decommissioned

**Success Criteria:**
- Named consumers: Watchdog, Kaylee, Hermes each verify ≥1 successful request through OmniRoute
- Rollback procedure tested and documented
- terminus-inference-gateway deployment and service removed from Helm chart
- architecture.md reflects confirmed implementation decisions

**Target Repo:** `TargetProjects/terminus/infra/terminus.infra`
