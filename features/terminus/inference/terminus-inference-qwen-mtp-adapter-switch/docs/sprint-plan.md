# Sprint Plan: qwen-mtp-adapter-switch

**Feature:** terminus-inference-qwen-mtp-adapter-switch  
**Domain/Service:** terminus / inference  
**Track:** express | **Phase:** expressplan  
**Date:** 2026-05-23

---

## Sprint Goal

Deliver the MTP-enabled llama.cpp direct inference path for Qwen3-35B-A22B as the primary provider in the terminus inference gateway, with the Ollama path preserved and tested as a fallback, and all acceptance criteria verified with passing BDD tests.

---

## Stories

### Story 1: llama-server systemd service and GGUF provisioning
**ID:** QMTP-1  
**Points:** 3  
**Owner:** infra (todd)

**Goal:** `llama-server` is running on k3s-worker-07 as a supervised systemd service with MTP GGUFs loaded, health endpoint reachable at `llamacpp.trantor.internal:9090`.

**Tasks:**
0. Pre-sprint: Verify `llama-server --version` or binary build date on k3s-worker-07 confirms llama.cpp ≥ b9253 with MTP support (`--spec-type draft-mtp` must be a recognised flag). Record commit/tag in the infra runbook. **Do not proceed to GGUF download until confirmed.**
1. Download bartowski Q4_K_M MTP-enabled GGUF and MTP draft GGUF to PVC path `/llamacpp-models/` via Semaphore imperative job (Art. 19 — this is an imperative download, not GitOps). Pre-download PVC free-space check required — abort and alert if available space < 30 GB.
2. Write `llamacpp.service` systemd unit with `--spec-type draft-mtp --spec-draft-n-max 3` flags, full GPU offload (`-ngl 99`), flash attention (`-fa 1`), `--port 9090`, `Restart=always`, `RestartSec=5s`
3. Enable and start the unit; verify `GET /health` returns 200
4. Create DNS A record `llamacpp.trantor.internal → 10.0.0.133` in internal bind9 nameserver
5. Record GGUF checksums in infra runbook for reproducibility (Art. 9 — operational integrity)

**Acceptance Criteria:**
```
Given llama-server is started via systemd on k3s-worker-07
When GET http://llamacpp.trantor.internal:9090/health is called from the k8s cluster
Then the response is HTTP 200
And the model is loaded and serving
```

**Rollback:** `systemctl stop llamacpp` — Ollama unaffected.

---

### Story 2: `llamacpp` adapter in terminus-inference-gateway (TDD)
**ID:** QMTP-2  
**Points:** 5  
**Owner:** dev (todd)  
**Depends on:** QMTP-1 (health endpoint required for integration test)

**Goal:** A `llamacpp` provider adapter is implemented in Go following the `adapter.Provider` interface, registered in the gateway, and fully covered by TDD unit and integration tests.

**Tasks (TDD — red before green):**
1. Write failing unit test for `LlamaCppAdapter.Name()` → `"llamacpp"` [RED]
2. Implement `llamacpp/adapter.go` with `New()`, `Name()`, `Chat()`, `Health()` [GREEN]
3. Write failing unit tests for `Chat` happy path, context cancellation, non-200 error [RED]
4. Implement error handling and response normalization [GREEN]
5. Write integration test targeting live llama-server stub or mock [RED → GREEN]
6. Add `LlamaCppBaseURL`, `LlamaCppTimeoutSeconds`, `LlamaCppEnabled` to `GatewayConfig` [RED → GREEN]
7. Wire `LlamaCppAdapter` registration in `main.go`/cmd behind `LlamaCppEnabled` flag [RED → GREEN]

**Acceptance Criteria:**
```
Given the llamacpp adapter is registered and llama-server is healthy at its configured URL
When a ChatRequest is dispatched to the llamacpp adapter
Then a normalized ChatResponse is returned
And finish_reason is "stop" or "length"
And no internal error details are forwarded to the caller

Given llama-server returns a non-200 response
When Chat is called
Then an error is returned that maps to a gateway error type
And no raw provider error detail is forwarded

Given the request context is cancelled before llama-server responds
When Chat is called
Then the in-flight HTTP call is cancelled
And an appropriate context error is returned
```

---

### Story 3: Helm values update — `llamacpp` as primary provider
**ID:** QMTP-3  
**Points:** 2  
**Owner:** dev (todd)  
**Depends on:** QMTP-2

**Goal:** `values.yaml` routes `interactive` and `batch` profiles to `llamacpp` as primary with `ollama` as fallback. Deployed to dev cluster via ArgoCD sync.

**Tasks:**
1. Add `llamacpp` provider block to `values.yaml` (baseURL, timeoutSeconds, enabled: true)
2. Update `routing.config` in `values.yaml`: `interactive` chain = `[llamacpp, ollama, openai]`, `batch` chain = `[llamacpp, ollama]`
3. Update route profile manifest with `llamacpp` backend entries for all qwen routes (Inference Art. 1)
4. Deploy to dev via `helm upgrade` + ArgoCD sync
5. Validate `GET /healthz` reports `llamacpp` healthy
6. Run benchmark: confirm ≥ 1.3× throughput vs prior Ollama baseline (1.5× expected; 1.3× acceptance gate). If result is below 1.3×: adjust `--spec-draft-n-max` (try 4 or 5) and re-bench before making a promotion decision. Do not ship below 1.3× without a documented re-bench attempt. If re-bench still < 1.3×, file a `.todo` entry and revert sprint outcome to Ollama primary.

**Acceptance Criteria:**
```
Given llamacpp is the first entry in the provider_chain for "interactive" and "batch" profiles
When a chat request arrives at the gateway
Then the request is dispatched to the llamacpp provider
And the response is returned to the caller within the route timeout

Given llamacpp is unhealthy (server down)
When a chat request arrives
Then the gateway falls back to the ollama provider
And the caller receives a valid response

Given the llamacpp provider is removed from provider_chain in values.yaml and helm upgrade is run
When a chat request arrives
Then the request routes to ollama only
And the rollback takes under 5 minutes from upgrade trigger to healthy ollama response
```

---

### Story 4: qwen-warmup update — probe llamacpp endpoint
**ID:** QMTP-4  
**Points:** 1  
**Owner:** dev (todd)  
**Depends on:** QMTP-1, QMTP-3

**Goal:** The qwen-warmup service probes the `llamacpp` endpoint on startup alongside or in place of the Ollama endpoint, ensuring the MTP model is hot-loaded into GPU memory before the first real request.

**Tasks:**
1. Add `LLAMACPP_BASE_URL` env var support to `warmup.py`
2. Add a warmup probe for `POST /v1/chat/completions` against `llamacpp.trantor.internal:9090`
3. Update `deploy/` manifest to supply the new env var
4. Verify warmup log confirms `llamacpp` is ready after startup

**Acceptance Criteria:**
```
Given LLAMACPP_BASE_URL is configured in the warmup deployment
When the warmup service starts
Then it sends a probe request to the llamacpp endpoint
And logs confirm the llamacpp model responded within the warmup timeout
```

---

## Sprint Sequence

```
QMTP-1  ──────────────────────────────────────► (infra, parallel start)
                             QMTP-2  ──────────►
                                       QMTP-3  ►
                                         QMTP-4►
```

Stories 1 and 2 can begin in parallel (infra + dev). QMTP-3 depends on QMTP-2 and can start once the adapter passes unit tests (integration test can be done against QMTP-1). QMTP-4 is a small tail task after QMTP-3 is live in dev.

---

## Unresolved Risks for FinalizePlan

1. **Benchmark gate:** Sprint includes a throughput task (QMTP-3, task 6). If measured speedup < 1.3×, FinalizePlan should review `--spec-draft-n-max` value and whether the MoE architecture benefits from MTP as expected.
2. **systemd supervision gap:** `llama-server` is not managed by k8s. If the process dies, the gateway health check will surface it, but restart is manual. FinalizePlan should decide whether to wrap `llama-server` in a k8s pod for automatic restart (and capture as a `.todo` entry if deferred).
3. **GGUF download verification:** Bartowski checksums must be verified post-download. Semaphore job for the download must include a sha256 verification step.

---

## Definition of Done

- All four stories are implemented with passing TDD unit tests (red-green commit history) and BDD scenario tests (Art. 7, Art. 8)
- llama.cpp commit/tag confirmed ≥ b9253 and recorded in infra runbook before QMTP-1 begins
- `llamacpp.trantor.internal:9090` responds healthy in dev cluster
- Gateway routes to `llamacpp` primary; falls back to `ollama` on provider failure
- Rollback to `ollama`-only confirmed under 5 minutes
- Throughput ≥ 1.3× baseline measured and recorded
- ArgoCD reports Synced and Healthy
- Route profile manifest updated with `llamacpp` backend entries
- No secrets committed to source control (Art. 9)
- `.todo` entry filed for systemd supervision gap if deferred to follow-on
