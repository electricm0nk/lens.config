---
feature: terminus-inference-qwen-adapter
story_id: S1
title: Warmup Script
epic: E1
sprint: 1
points: 8
status: not-started
assignee: null
depends_on: []
blocks: [S3, S8]
updated_at: '2026-04-26T14:55:00Z'
---

# S1 — Warmup Script

## As a
Container operator

## I want
A Python warmup script (`warmup.py`) that runs at container startup to preload the Qwen model into GPU memory

## So that
When user requests arrive, the model is already resident in VRAM and responses are immediate (<2 sec)

## Context

The Qwen adapter pod currently suffers from JIT cold-start: when the first user request arrives, the model must be loaded from disk to VRAM (10+ minutes). This story creates the warmup script that runs automatically at container startup to eliminate this delay.

**Target file:** `warmup.py` in the container image (location TBD with team; likely `/app/warmup.py`)

## Acceptance Criteria

1. `warmup.py` exists in the container image at `/app/warmup.py`
2. Script polls `http://localhost:11434/api/tags` with `wait_for_ollama(timeout=60)` until ollama is responsive — raises exception if not ready within 60 seconds
3. Script calls `http://localhost:11434/api/pull` with `{"model": "${OLLAMA_MODEL}"}` to ensure the model is cached locally — uses the `OLLAMA_MODEL` environment variable (not a hard-coded model name)
4. Script calls `http://localhost:11434/api/generate` with the `OLLAMA_MODEL` and a test prompt (`"Hello, how are you?"`) to confirm the model is loaded into VRAM — the call must complete within 300 seconds
5. **GPU VRAM check:** Before pulling/loading, script queries `nvidia-smi --query-gpu=memory.free --format=csv,noheader,nounits` and fails fast (exit 1) if available VRAM < 24576 MB (24GB) with a clear error message
6. On success: writes `/tmp/warmup.ready` marker file and exits 0
7. On failure: writes error details to stderr and exits 1 (single attempt — no retry; pod restart handles recovery)
8. All steps log to stdout with timestamps: `[warmup] Waiting for ollama...`, `[warmup] Pulling model...`, `[warmup] Running test query...`, `[warmup] ✓ Ready in {duration}s`
9. Overall warmup timeout: 300 seconds from script start to model-loaded confirmation

## Technical Notes

```python
# Key functions:
# wait_for_ollama(timeout=60) — polls /api/tags
# pull_model(model=os.environ["OLLAMA_MODEL"]) — POST /api/pull
# test_generation(model=os.environ["OLLAMA_MODEL"]) — POST /api/generate
# check_gpu_memory(min_mb=24576) — nvidia-smi query
# main() — orchestrates sequence, writes /tmp/warmup.ready on success
```

```
OLLAMA_MODEL env var must be set in Dockerfile (e.g., qwen2.5:14b)
Confirm exact model tag with team before starting this story
```

## Definition of Done

- [ ] `warmup.py` implemented and unit-testable (S8 depends on this)
- [ ] VRAM check implemented (≥24GB threshold)
- [ ] `/tmp/warmup.ready` marker written on success
- [ ] Exits 0 on success, 1 on failure
- [ ] All steps logged with timestamps
- [ ] Code reviewed and merged to feature branch
