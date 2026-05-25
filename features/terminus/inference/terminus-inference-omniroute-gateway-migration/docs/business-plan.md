# Business Plan — OmniRoute Gateway Migration

**Feature:** terminus-inference-omniroute-gateway-migration  
**Domain/Service:** terminus / inference  
**Track:** express  
**Phase:** expressplan  
**Owner:** Todd Hintzmann  
**Date:** 2026-05-25

---

## Problem Statement

The current `terminus-inference-gateway` is a homegrown Go service that routes inference requests to the local llamacpp backend. It was built to solve an immediate need: provide an OpenAI-compatible API endpoint with Vault JWT authentication on top of a single local GPU worker. While it solves that problem, it carries a growing operational tax:

- **Single-backend hardwiring.** The gateway has a static provider adapter model. Adding, swapping, or load-balancing inference backends requires code changes and redeployment.
- **No routing intelligence.** There is no automatic fallback, circuit breaking, or strategy-based routing. If llamacpp is down, all consumers fail immediately.
- **Manual route profile management.** Route profiles (chat, code-completion, interactive, batch, triage) are implemented as bespoke gateway logic rather than declarative configuration.
- **Limited observability for routing decisions.** Prometheus alerts exist for backend availability, but there is no provider-level cost tracking, semantic cache, or quota monitoring.
- **Ongoing maintenance burden.** The gateway is custom Go code with a full OpenAPI spec, oapi-codegen types, contract test suite, and CI pipeline. Every feature addition is an internal engineering cost.

The inference layer is expected to grow in complexity: additional local GPU workers, potential OpenAI fallback for quota overflow, cost-tag-based routing, and batch-specific routing constraints all require gateway capabilities the homegrown solution was not designed to provide.

---

## Proposed Solution

Replace `terminus-inference-gateway` with **OmniRoute** — an open-source (MIT licensed), Docker-deployable AI gateway with 160+ provider support, 13 routing strategies, circuit breaker, semantic cache, and an OpenAI-compatible API.

**Deployment model:** OmniRoute runs as a self-hosted Docker container inside the `inference` Kubernetes namespace. All inference traffic remains on-premises. The llamacpp backend is registered as a custom OpenAI-compatible provider. No inference traffic leaves the local network.

**Key capabilities gained:**
- Declarative provider + combo configuration (maps directly to existing route profiles)
- Circuit breaker with configurable cooldowns — automatic recovery from llamacpp transients
- Semantic cache — exact + similarity-based deduplication reduces redundant GPU invocations
- Priority / weighted / cost-optimized routing strategies — enables multi-GPU or cloud-overflow routing without code changes
- Request idempotency (5-second dedup window) — prevents double-sends from retries
- Usage analytics and cost tracking per provider
- Prompt injection guard (built-in)

---

## Target Users

| Consumer | Current Experience | Experience After Migration |
|----------|-------------------|---------------------------|
| Watchdog triage workers | HTTP to `inference-gateway.trantor.internal/v1` | Same endpoint, same API contract — no change required |
| Kaylee agent | HTTP to `inference-gateway.trantor.internal/v1` | Same endpoint, no change required |
| Hermes platform | HTTP to `inference-gateway.trantor.internal/v1` | Same endpoint, no change required |
| Platform developers | Manual route profile code changes for new routing behaviors | Declarative combo + provider config via OmniRoute dashboard |
| Operators | Limited fallback visibility | Circuit breaker status, health monitor, usage analytics |

---

## Business Goals

1. **Eliminate homegrown gateway maintenance** — stop investing engineering time in custom router code that can be replaced by a maintained open-source tool.
2. **Unblock multi-provider routing** — enable automatic fallback to OpenAI (or other providers) when llamacpp is at capacity, without code changes.
3. **Preserve zero data-exfiltration guarantee** — all inference traffic must remain on-premises. OmniRoute is deployed self-hosted; `cloud.omniroute.online` is not used.
4. **Maintain API contract stability** — all existing consumers call `https://inference-gateway.trantor.internal/v1` with the OpenAI API shape. The migration must be transparent to consumers.
5. **Reduce time-to-route for new routing behaviors** — moving from code change + CI + deploy to declarative combo configuration.

---

## Scope

### In Scope
- OmniRoute Docker deployment in the `inference` Kubernetes namespace
- Vault JWT authentication bridge (existing consumers use Vault JWT tokens; OmniRoute uses API key auth — a thin ingress auth adapter is required)
- Migration of all five existing route profiles: chat, code-completion, interactive, batch, triage
- llamacpp registration as a custom OpenAI-compatible provider in OmniRoute
- OpenAI as an optional fallback provider (for chat, code-completion, interactive, triage route profiles)
- Prometheus observability wiring
- Decommission of `terminus-inference-gateway` Go service after cutover validation
- Article 16 compliance: `.todo` entry documenting OmniRoute as a third-party dependency replacing an internal component

### Out of Scope
- New route profiles beyond the five that exist today (post-migration feature work)
- OmniRoute MCP server features (agent tool use)
- OmniRoute memory or skills framework
- Media generation routes
- Multi-GPU load balancing (deferred to a follow-on feature after OmniRoute is stable)

---

## Dependencies

| Dependency | Type | Risk |
|-----------|------|------|
| llamacpp Kubernetes service (`llamacpp.inference.svc.cluster.local:9090`) | Upstream | Low — service is stable, 118.5 tok/s verified |
| Vault JWT authentication (`vault.trantor.internal`) | Upstream | Medium — auth bridge design must preserve existing consumer token flow |
| OpenAI API key (optional fallback) | External | Low — gated behind rate-limiting; only activated if llamacpp unavailable |
| OmniRoute Docker image (`diegosouzapw/omniroute:latest`) | Third-party | Medium — pin to specific version tag; do not use `:latest` in production |
| Kubernetes PersistentVolume for OmniRoute data | Infra | Low — existing storage class supports this |

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Vault JWT → OmniRoute auth bridge adds latency | Medium | Medium | Implement auth at K8s ingress layer (not in-process) |
| OmniRoute Node.js runtime adds memory/CPU overhead vs. Go binary | Low | Low | Monitor pod resource usage; set reasonable K8s resource limits |
| OmniRoute configuration state is not GitOps-managed by default | Medium | Medium | Export OmniRoute config as committed YAML; treat data volume as source of truth only for runtime state |
| OmniRoute upstream changes break self-hosted behavior | Low | Medium | Pin version; subscribe to OmniRoute release notes; do not auto-update |
| Consumers depend on undocumented response fields that differ between gateways | Low | High | Run contract test suite against OmniRoute before cutover |

---

## Success Criteria

1. All five route profiles return correct responses through OmniRoute with the same request/response contract as the current gateway.
2. The Vault JWT auth flow is preserved end-to-end — existing consumers require no token changes.
3. llamacpp circuit breaker fires correctly on backend downtime and recovers automatically.
4. Prometheus `LlamaCppProviderDown` alert continues to fire appropriately (or equivalent OmniRoute health alert is wired).
5. No inference traffic leaves the local network (`cloud.omniroute.online` is not used).
6. The Article 16 `.todo` entry is committed and marked as reviewed before the dev phase begins.
7. `terminus-inference-gateway` Go service is decommissioned and removed from the Helm chart after successful cutover validation.
