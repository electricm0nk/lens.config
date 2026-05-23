# Tech Plan: qwen-mtp-adapter-switch

**Feature:** terminus-inference-qwen-mtp-adapter-switch  
**Domain/Service:** terminus / inference  
**Track:** express | **Phase:** expressplan  
**Date:** 2026-05-23

---

## Overview

This plan describes the technical design for switching the Qwen3-35B-A22B (MoE) inference path from Ollama to a locally running `llama-server` (llama.cpp ≥ b9253) with MTP speculative decoding enabled. The Ollama path is preserved as a documented fallback in the routing config.

---

## Current Architecture

```
[Gateway pod] → adapter.Registry → OllamaAdapter → http://ollama.trantor.internal:11434
                                                    (Ollama serves Qwen3-35B MoE via GGUF blob)
```

**Relevant files:**
- `internal/adapter/ollama/adapter.go` — Ollama HTTP adapter
- `internal/adapter/openai/adapter.go` — OpenAI-compatible HTTP adapter (exists, reusable pattern)
- `internal/config/config.go` — `GatewayConfig` struct with `OllamaBaseURL`, `OpenAIBaseURL`
- `deploy/helm/values.yaml` — provider config, routing profiles
- `internal/adapter/registry.go` — data-driven registry; provider name from `adapter.Name()`

**Current routing config (from Helm values):**
```yaml
profiles:
  - name: interactive
    provider_chain: [ollama, openai]
  - name: batch
    provider_chain: [ollama]
```

---

## Target Architecture

```
[Gateway pod] → adapter.Registry → LlamaCppAdapter → http://llamacpp.trantor.internal:9090
                                 → OllamaAdapter   → http://ollama.trantor.internal:11434 (fallback)

[k3s-worker-07] llama-server process:
  --model <Q4_K_M-MTP.gguf>
  --draft <MTP-draft.gguf>
  --spec-type draft-mtp
  --spec-draft-n-max 3
  --port 9090
  --host 0.0.0.0
  -ngl 99                   (full GPU offload)
  -fa 1                     (flash attention)
```

The `llamacpp` provider speaks the llama.cpp built-in HTTP server's OpenAI-compatible API (`/v1/chat/completions`). This is architecturally equivalent to the `openai` adapter pointing at a local endpoint — a thin new adapter wrapping the same HTTP call pattern but:
- Named `llamacpp` for unambiguous registry lookup and routing config legibility
- No API key required (local endpoint)
- Configurable timeout defaults appropriate to long Qwen generation runs (300 s, same as Ollama)
- Returns `Name() = "llamacpp"` for routing resolution

### llama-server process management

`llama-server` will run as a **systemd service** on `k3s-worker-07`, analogous to how Ollama runs as a system service on the same host.

**systemd unit key settings:**
```
Restart=always
RestartSec=5s
```

This ensures automatic restart on OOM kill or process failure. The gateway health check will reflect the live state within one health poll cycle.

**DNS provisioning:** A DNS A record `llamacpp.trantor.internal → 10.0.0.133` will be added to the internal bind9 nameserver (same mechanism used for `ollama.trantor.internal`). No CoreDNS ConfigMap or hostAliases needed.

**MTP GGUF placement:**
```
<pvc-mount>/llamacpp-models/
  qwen3-35b-a22b-q4_k_m-mtp.gguf        # main model (~22 GB, bartowski)
  qwen3-35b-a22b-q4_k_m-mtp-draft.gguf  # MTP draft model (~1.2 GB)
```

These are on the same PVC used by Ollama (`pvc-c14d913e-...`), avoiding duplicate storage provisioning.

---

## Implementation Boundaries

### Repo: `terminus-inference-gateway`

| Change | Type | Location |
|---|---|---|
| New `llamacpp` adapter package | New | `internal/adapter/llamacpp/` |
| `GatewayConfig` — add `LlamaCppBaseURL`, `LlamaCppTimeoutSeconds`, `LlamaCppEnabled` | Modify | `internal/config/config.go` |
| Register `llamacpp` adapter in wiring | Modify | `cmd/` or `main.go` wire-up |
| Helm `values.yaml` — add `llamacpp` provider block, update routing profiles | Modify | `deploy/helm/values.yaml` |
| Route profile manifest — add `llamacpp` entries for qwen routes | Modify | `deploy/` or `internal/routing/` |

### Infra: `terminus.infra` (k3s cluster)

| Change | Type | Notes |
|---|---|---|
| systemd unit file for `llama-server` on `k3s-worker-07` | New | `llamacpp.service` — starts with MTP flags |
| DNS/hostfile entry `llamacpp.trantor.internal` | New | Points to k3s-worker-07 (10.0.0.133) |
| GGUF download script/task for bartowski Q4_K_M MTP | New | Semaphore job (imperative, not ArgoCD) |

### Repo: `terminus-inference-qwen-warmup`

| Change | Type | Notes |
|---|---|---|
| Add warmup target for `llamacpp` endpoint | Modify | Existing warmup hits Ollama; add parallel MTP endpoint probe |

---

## `llamacpp` Adapter Design

```go
// Package llamacpp implements the adapter.Provider interface for a local llama-server
// instance exposing an OpenAI-compatible /v1/chat/completions API.
package llamacpp

type LlamaCppAdapter struct {
    baseURL    string
    httpClient *http.Client
}

func New(baseURL string, timeoutSeconds int) *LlamaCppAdapter { ... }
func (a *LlamaCppAdapter) Name() string { return "llamacpp" }
func (a *LlamaCppAdapter) Chat(ctx context.Context, req *adapter.ChatRequest) (*adapter.ChatResponse, error) { ... }
func (a *LlamaCppAdapter) Health(ctx context.Context) error { ... }
```

The `Chat` implementation is structurally identical to the OpenAI adapter's HTTP call but:
- No `Authorization` header
- Target endpoint is `{baseURL}/v1/chat/completions`
- No API key required
- Model name passed as-is (no prefix stripping needed; llama-server accepts any model name)
- `Stream: false` hardcoded (consistent with the existing pipeline gate — streaming is rejected at the gateway layer, not the adapter layer)
- Rationale for a separate package rather than reusing the OpenAI adapter: explicit registry identity (name = "llamacpp" unambiguous in routing config), future telemetry hooks for MTP acceptance rate, no API key wiring, and explicit MTP-specific defaults. Code is not duplicated — both adapters share the same OpenAI-compat HTTP contract, but the `llamacpp` package provides a named, independently testable provider unit.

---

## Updated Routing Config

```yaml
providers:
  - name: llamacpp
    api_key_env: ""
    daily_budget_usd: 0
    price_per_1k_tokens: 0.00005  # internal cost tag (lower than ollama)
  - name: ollama
    api_key_env: ""
    daily_budget_usd: 0
    price_per_1k_tokens: 0.0001
  - name: openai
    api_key_env: OPENAI_API_KEY
    daily_budget_usd: 5.0
    price_per_1k_tokens: 0.002

profiles:
  - name: interactive
    provider_chain: [llamacpp, ollama, openai]   # MTP primary; Ollama fallback; OpenAI last-resort
  - name: batch
    provider_chain: [llamacpp, ollama]             # MTP primary; Ollama fallback
```

---

## Data / Artifact Contracts

- **API contract unchanged:** The gateway's `/v1/chat/completions` response shape is normalized by the adapter layer. Consumers see no difference between Ollama and llama.cpp responses.
- **Route profile manifest:** All qwen routes must declare `llamacpp` as backend once the adapter is wired. Inference Art. 1 compliance.
- **No confidential data:** All inference data remains on-cluster. The only new external fetch is the GGUF download (one-time, pull-only, no user data transmitted). Org Art. 4 satisfied.

---

## Testing Strategy

Following **Art. 7 (TDD)** and **Art. 8 (BDD)**:

### Unit tests (TDD — red-green discipline)
- `llamacpp/adapter_test.go` — Chat request/response translation, error handling, context cancellation
- `llamacpp/adapter_test.go` — Health check: 200 → healthy, non-200 → error
- `config_test.go` — LlamaCppBaseURL, LlamaCppEnabled config parsing
- Registry test — `llamacpp` registers and resolves correctly

### Integration test
- `adapter_integration_test.go` — live call against a local `llama-server` stub or dockerized mock

### BDD acceptance criteria (story-level)
Each story AC must have a fully implemented BDD scenario. Format: Given/When/Then. Example:
```
Given the llamacpp adapter is registered and llama-server is healthy
When a chat request is routed to the "llamacpp" provider
Then a normalized ChatResponse is returned within the route timeout
And the finish_reason is "stop" or "length"
```

### Rollback smoke test (Inference Art. 5)
```
Given the llamacpp provider is primary in the routing config
When the llamacpp provider is removed from the provider_chain in values.yaml
And the gateway Helm release is upgraded
Then all requests route to ollama with correct responses
And no service restart of Ollama is required
```

---

## Rollout Plan (Inference Art. 5)

| Step | Action | Audience |
|---|---|---|
| 1 | Download MTP GGUFs to PVC | Infrastructure |
| 2 | Start llama-server systemd unit, validate health endpoint | k3s-worker-07 |
| 3 | Deploy gateway with `llamacpp` registered but NOT in routing profiles | dev |
| 4 | Validate health check passes via gateway `/healthz` | dev |
| 5 | Enable `llamacpp` as primary in routing profiles | dev |
| 6 | Run smoke test and compare throughput | dev |
| 7 | Promote to prod if throughput ≥ 1.5× baseline | prod |

**Rollback procedure:** Remove `llamacpp` from `provider_chain` in `values.yaml`, run `helm upgrade`. Ollama resumes as primary. No pod restart of Ollama required. ETA: < 5 minutes.

---

## AI Safety Notes (Art. 15)

- All inference is local; no user data leaves the cluster network boundary
- MTP draft speculation happens entirely within llama.cpp process on the worker GPU; no external calls
- The draft model is used only to propose tokens that the main model then accepts or rejects — the main model's output remains authoritative
- No new trust boundaries introduced

---

## Language Notes (Inference Art. 7)

New adapter is Go — consistent with the existing gateway codebase. No language justification required.

---

## Known Technical Risks

| Risk | Mitigation |
|---|---|
| llama-server HTTP API surface differs from expected OpenAI schema | Test against actual llama-server ≥ b9253 in integration tests |
| MTP draft acceptance rate low on MoE architecture → no speedup | Sprint includes a benchmark task; if < 1.3× gain, revisit spec-draft-n-max |
| VRAM headroom insufficient with main + draft loaded | Monitor with `nvidia-smi`; draft Q4 is ~1.2 GB so worst-case is 23.2/24 GB used |
| systemd unit not supervised by k8s → process die undetected | Gateway health check propagates via `/healthz`; future work: wrap in k8s pod |
