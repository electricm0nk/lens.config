# Tech Plan — OmniRoute Gateway Migration

**Feature:** terminus-inference-omniroute-gateway-migration  
**Domain/Service:** terminus / inference  
**Track:** express  
**Phase:** expressplan  
**Owner:** Todd Hintzmann  
**Date:** 2026-05-25

---

## Architecture Overview

The migration replaces the homegrown `terminus-inference-gateway` Go service with a self-hosted OmniRoute Docker container. The external API surface and consumer contracts remain unchanged. The internal routing layer is replaced wholesale.

### Current Architecture

```
Consumer (Vault JWT) 
  → K8s Ingress (inference-gateway.trantor.internal/v1)
    → terminus-inference-gateway (Go, Chi router, Vault JWT validation)
      → llamacpp (http://llamacpp.inference.svc.cluster.local:9090)
```

### Target Architecture

```
Consumer (Vault JWT)
  → K8s Ingress (inference-gateway.trantor.internal/v1)
    → Auth Proxy sidecar / ingress annotation (Vault JWT → OmniRoute API key header)
      → OmniRoute (Node.js, Docker, port 20128)
        → llamacpp provider (http://llamacpp.inference.svc.cluster.local:9090)  ← primary
        → openai provider (api.openai.com)  ← fallback only (chat, interactive, triage, code-completion)
```

---

## Authentication Bridge Design

**Problem:** Existing consumers authenticate with Vault-issued JWT tokens. OmniRoute uses its own API key access control system. These are incompatible without a bridge.

**Selected approach: K8s Ingress auth annotation + header injection**

Option evaluated: run a thin auth proxy in front of OmniRoute that validates the Vault JWT, strips it, and injects the OmniRoute API key as `Authorization: Bearer <omniroute-key>`.

Implementation options:
1. **Nginx ingress `auth-url` annotation** — offload JWT validation to an existing Vault auth service, inject static OmniRoute API key downstream. No new service needed if a Vault JWT validation endpoint already exists.
2. **Envoy ext-authz sidecar** — richer but higher complexity; not warranted for this scale.
3. **Thin Go auth proxy (new service)** — replaces one homegrown component with another; undesirable.

**Recommendation:** Option 1 (Nginx auth annotation) if a Vault JWT validation endpoint is available. If not available, a minimal auth proxy that validates the JWT using Vault's `/v1/auth/token/lookup-self` endpoint and injects the OmniRoute API key is the fallback. The proxy is stateless and can be a single-binary Go service (aligns with Svc-7).

**OmniRoute API key:** generated during OmniRoute setup, stored in Vault, injected via K8s secret. Never committed to source control (Article 9).

---

## OmniRoute Kubernetes Deployment

### Namespace
`inference` (co-located with llamacpp, existing PVC storage class available)

### Deployment spec highlights
```yaml
image: diegosouzapw/omniroute:<pinned-version>   # pin explicit semver tag, not latest
port: 20128
resources:
  requests: { cpu: 200m, memory: 512Mi }
  limits:   { cpu: 1000m, memory: 1Gi }
volumeMounts:
  - name: omniroute-data
    mountPath: /app/data
volumes:
  - name: omniroute-data
    persistentVolumeClaim:
      claimName: omniroute-data-pvc
```

### PersistentVolumeClaim
1Gi ReadWriteOnce on existing storage class. Stores OmniRoute config, provider state, analytics DB.

### Service / Ingress
- ClusterIP service `omniroute.inference.svc.cluster.local:20128`
- Existing ingress `inference-gateway.trantor.internal/v1` repoints from terminus-inference-gateway to OmniRoute service
- TLS cert (existing cert-manager issuer) retained unchanged

---

## Provider Configuration

OmniRoute provider registrations (stored in persisted data volume, also exported as committed config):

| Provider ID | Type | URL | Auth | Role |
|------------|------|-----|------|------|
| `llamacpp-local` | OpenAI-compatible | `http://llamacpp.inference.svc.cluster.local:9090` | none (internal) | Primary for all combos |
| `openai-fallback` | OpenAI | `https://api.openai.com` | API key (Vault-injected secret) | Fallback for chat, interactive, triage, code-completion |

---

## Route Profile Mapping

Existing route profiles map to OmniRoute **Combos** (named provider chains with strategy):

| Profile Name | Current Provider Chain | OmniRoute Combo | Routing Strategy | Fallback | Timeout |
|-------------|----------------------|-----------------|-----------------|---------|---------|
| `chat` | `[llamacpp, openai]` | `combo-chat` | priority | openai-fallback | 300s |
| `code-completion` | `[llamacpp, openai]` | `combo-code` | priority | openai-fallback | 300s |
| `interactive` | `[llamacpp, openai]` | `combo-interactive` | priority | openai-fallback | 120s |
| `batch` | `[llamacpp]` | `combo-batch` | strict (no fallback) | none — fail explicit (Svc-6) | 0 (caller-enforced) |
| `triage` | `[llamacpp, openai]` | `combo-triage` | priority | openai-fallback | 60s |

**Route Profile Manifest** (Svc-1 compliance): a committed YAML file `docs/terminus/inference/terminus-inference-omniroute-gateway-migration/route-profiles.yaml` documents all combos with complete field values: route ID, model target/selection policy, request timeout, retry policy, escalation behavior, cost tag.

**Batch route explicit failure** (Svc-6 compliance): `combo-batch` must use strict routing with no implicit fallback. If llamacpp is unavailable, batch jobs receive an explicit 503 error and must not silently fail over to OpenAI.

---

## Guardrails (Svc-4 Compliance)

OmniRoute guardrails are configuration-driven per combo:

| Combo | Max Duration | Max Output Tokens | Max Retry | Cost Tag |
|-------|-------------|------------------|----------|---------|
| `combo-chat` | 300s | 8192 | 2 | `chat` |
| `combo-code` | 300s | 8192 | 2 | `code-completion` |
| `combo-interactive` | 120s | 4096 | 1 | `interactive` |
| `combo-batch` | 0 (caller) | 16384 | 0 | `batch` |
| `combo-triage` | 60s | 2048 | 1 | `watchdog-triage` |

All guardrail values are surfaced in the committed route-profiles manifest, not as hard-coded application logic.

---

## Controlled Rollout Plan (Svc-5 Compliance)

**Target audience tier:** dev (internal)  
**Rollout scope:** full — single deployment, no canary (only one GPU worker)  
**Success criteria:**
1. All five OmniRoute combos return correct OpenAI-shaped responses for representative test requests
2. Contract test suite passes against OmniRoute endpoint
3. llamacpp circuit breaker opens on backend shutdown and closes within 30s on recovery
4. Vault JWT auth bridge validates tokens correctly and rejects invalid tokens with 401
5. No inference traffic observed leaving the local network (tcpdump/network policy verify)

**Rollback procedure:**
1. `kubectl -n inference patch ingress inference-gateway --type=json -p '[{"op":"replace","path":"/spec/rules/0/http/paths/0/backend/service/name","value":"terminus-inference-gateway"}]'`
2. Confirm `terminus-inference-gateway` pods are still running (scale up if needed during migration window)
3. Verify via `curl https://inference-gateway.trantor.internal/v1/health` returns 200

`terminus-inference-gateway` pods are **not** deleted until cutover validation passes. Maintained in parallel for the rollback window.

---

## Architecture Documentation (Org-3 Compliance)

This tech plan serves as the architecture documentation for the migration. In addition, an `architecture.md` must be produced in the feature docs path before the dev phase begins, documenting:
- Final auth bridge implementation choice with rationale
- OmniRoute K8s deployment topology
- Route profile manifest with all combo definitions
- Observability wiring

Language Justification (Svc-7): OmniRoute itself is Node.js. Since OmniRoute is a third-party Docker image consumed as a dependency (not authored code), Go preference does not apply to OmniRoute internals. If an auth proxy service is needed and cannot be implemented via Nginx annotations, it will be written in Go per Svc-7.

---

## Observability

| Signal | Current | Target |
|--------|---------|--------|
| `LlamaCppProviderDown` Prometheus alert | kube_endpoint alert on llamacpp service | Preserve; OmniRoute circuit breaker state also surfaced |
| GPU metrics | dcgm-exporter on gpu-worker node | No change |
| Inference gateway health | `/health` on terminus-inference-gateway | `/health` on OmniRoute (`localhost:20128/health`) |
| Cost tracking | None | OmniRoute usage analytics per combo per provider |
| Auth failures | Vault audit logs | Auth proxy → Kubernetes event log + Prometheus counter |

---

## Security (Org-9 Compliance)

- OmniRoute API key stored in Vault; injected as K8s secret; never committed to source control
- OpenAI fallback API key stored in Vault; injected as K8s secret
- OmniRoute data volume contains provider config; treat as sensitive — restrict PVC access to inference namespace
- `cloud.omniroute.online` must not be used; OmniRoute endpoint is `localhost:20128` within the cluster only
- K8s NetworkPolicy restricts OmniRoute pod egress: allow llamacpp ClusterIP, allow api.openai.com:443 (for fallback), deny all other external egress

---

## Testing Strategy (Org-7, Org-8 Compliance)

| Test Type | Scope | Gate |
|-----------|-------|------|
| Contract tests (existing) | All five route profiles, request/response schema, error shape | Must pass against OmniRoute before cutover |
| BDD acceptance tests | Auth bridge validates JWT, rejects invalid token; circuit breaker fires and recovers; batch combo rejects with 503 on backend down | Must be passing before dev phase closes |
| Unit tests (auth proxy, if built) | JWT validation, header injection | TDD red-green; no production code without prior failing test |
| Integration smoke test | End-to-end request through OmniRoute to llamacpp and back | Part of cutover validation |

---

## Known Technical Risks

| Risk | Mitigation |
|------|-----------|
| OmniRoute config state is runtime-managed (dashboard), not GitOps by default | Export config as committed YAML on every change; treat data volume as ephemeral re-creatable from committed config |
| OmniRoute version pinning — upstream breaking changes | Pin explicit semver tag; no auto-update; validate on version bump |
| Auth proxy latency overhead | Measure p95 latency before and after cutover; rollback if regression > 20ms at p95 |
| llamacpp `parallel_slots=1` — OmniRoute queue behavior under concurrent requests | Validate queuing behavior; set appropriate combo timeout vs. llamacpp queue depth |
