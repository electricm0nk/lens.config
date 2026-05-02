---
feature: terminus-inference-qwen-adapter
doc_type: business-plan
status: draft
goal: "Eliminate JIT cold-start delays by preloading Qwen into memory and persisting indefinitely"
key_decisions:
  - Run warmup exercise script automatically on container startup
  - Disable ollama idle shutdown (persist indefinitely on dedicated GPU)
  - Load Qwen 3.6 model into vram before accepting requests
open_questions:
  - What's the optimal warmup strategy (lightweight query vs full model validation)?
  - How to monitor and log warmup completion status?
depends_on: []
blocks: []
updated_at: '2026-04-26T14:00:00Z'
---

## Overview

The Qwen adapter currently operates in **just-in-time (JIT) compilation mode**: models load only when first requested, causing 10-minute delays for users. On a dedicated GPU with no resource contention, this cold-start tax is unnecessary and degrades user experience.

**Business case:** Preload the Qwen 3.6 model into GPU memory at container startup, ensure ollama stays resident (no idle shutdown), and expose exercise endpoints for verification. Users experience instant responsiveness instead of JIT load delays.

## Problem Statement

- **Current behavior:** Container starts; agent stays idle; user requests arrive; 10-minute wait while Qwen loads from disk → GPU
- **Dedicated resource:** GPU is exclusive to this agent — no other workloads compete for it
- **User impact:** Unusable latency for interactive requests; cold-start anxiety for agent operators
- **Operational cost:** Repeated model loads waste time and energy

## Success Criteria

1. ✅ Agent pod starts, automatically runs warmup script
2. ✅ Qwen 3.6 fully loaded into GPU memory within 5 minutes of startup
3. ✅ Ollama process persists indefinitely (no idle shutdown)
4. ✅ First user request completes in <2 seconds (vs. 10 minutes cold-start)
5. ✅ Warmup completion can be verified via health/readiness endpoints
6. ✅ No resource cleanup on idle; model stays in vram for session lifetime

## Key Design Decisions

### 1. Warmup Strategy
- Run exercise script (`warmup.py` / shell script) automatically during container init
- Script loads Qwen model via ollama, validates with test query
- On success: model stays resident; pod marked ready
- On failure: retry with backoff, surface errors in logs

### 2. Ollama Persistence Configuration
- Set `OLLAMA_KEEP_ALIVE=0` or disable idle timeout entirely
- Ensure ollama process doesn't enter shutdown/cleanup cycles
- Model persists for the container lifetime

### 3. Readiness / Liveness Probes
- Readiness: return 200 only after warmup completes and model is loaded
- Liveness: verify ollama process is running and model is still in memory
- Prevents traffic routing during warmup phase

### 4. Model Validation
- Warmup script validates model with lightweight test (e.g., single token generation)
- Confirms CUDA availability and vram allocation
- Logs model size, memory usage, and load time for troubleshooting

## Non-Functional Requirements

- **Startup time:** Warmup should complete within 5 minutes for Qwen 3.6 (GPU-dependent)
- **Memory:** Qwen 3.6 ~20-40 GB vram (dedicated GPU must have sufficient capacity)
- **Observability:** Warmup logs, model load times, and memory state visible in pod logs
- **Graceful shutdown:** Model stays loaded until container terminates; no forced unload

## Dependencies & Risks

### Dependencies
- Ollama running and configured for model persistence
- GPU driver / CUDA compatible with Qwen inference
- Sufficient GPU memory for Qwen 3.6 (verify during pod init)

### Risks
1. **OOM on startup** — If GPU memory insufficient, warmup fails; detect early and fail pod startup
2. **Slow warmup** — Large models may take 5+ minutes; ensure readiness timeout accommodates
3. **Ollama config drift** — Idle shutdown timeout could re-enable after restart; hard-code in image
4. **Model mismatch** — Wrong model version or corrupted cache; validate checksums

## Implementation Approach

### Phase 1: Warmup Script
- Create `warmup.sh` / `warmup.py` that:
  - Waits for ollama to be ready
  - Pulls/validates Qwen 3.6 model
  - Runs test query to confirm loaded
  - Exits 0 on success, non-zero on failure

### Phase 2: Ollama Configuration
- Update `OLLAMA_KEEP_ALIVE` environment variable (set to 0 or high value)
- Ensure ollama service runs without idle shutdown
- Document persistent memory requirements

### Phase 3: Init Integration
- Add warmup script to container image (Dockerfile `RUN` or init system)
- Update k8s deployment readiness probe to poll for warmup completion
- Document expected startup time and memory requirements

### Phase 4: Verification & Monitoring
- Health endpoint reports model loaded status
- Logs include warmup duration, model size, vram utilization
- Metrics: startup time, model load time, readiness delay

## Success Metrics

| Metric | Target | Validation |
|--------|--------|-----------|
| Model load time | < 5 min | Warmup script logs |
| First request latency (post-warmup) | < 2 sec | Request timing from agent |
| Ollama uptime | Entire session | Process monitor / logs |
| Memory stability | Constant after warmup | GPU memory query |
| Startup readiness delay | < 5 min | k8s event timestamp |

## Open Questions

1. What's the optimal warmup query? (lightweight sentiment analysis, token generation, etc.)
2. Should warmup retry on failure, or fail fast?
3. How long to wait for ollama readiness before timing out?
4. Should readiness probe expose warmup metrics for observability?

## Rollout & Validation

- Deploy warmup script to dev environment
- Observe startup time, memory behavior, first-request latency
- Compare against baseline (current JIT behavior)
- Verify no resource leaks or unintended shutdowns
- Document in runbook for operators
