---
feature: terminus-inference-qwen-adapter
story_id: S2
title: Ollama Persistence Config
epic: E1
sprint: 1
points: 5
status: not-started
assignee: null
depends_on: []
blocks: [S3]
updated_at: '2026-04-26T14:55:00Z'
---

# S2 — Ollama Persistence Config

## As a
Container operator

## I want
Ollama configured to never unload the model on idle, with the model cache persisted to a durable volume

## So that
Qwen stays loaded in VRAM for the entire pod lifetime on this dedicated GPU

## Context

By default, ollama unloads models after 5 minutes of inactivity. On a dedicated GPU with no resource contention, this is unnecessary and would invalidate the warmup investment. This story configures ollama to disable idle shutdown and use a persistent model cache path.

## Acceptance Criteria

1. `Dockerfile` sets `ENV OLLAMA_KEEP_ALIVE=0` — model persists indefinitely, no idle unload
2. `Dockerfile` sets `ENV OLLAMA_MODELS=/models` — model cache stored on the persistent volume mount
3. `Dockerfile` sets `ENV OLLAMA_MODEL=<confirmed-model-tag>` — e.g., `qwen2.5:14b` (confirm exact tag with team before story start)
4. The `OLLAMA_KEEP_ALIVE` value is hard-coded in the Dockerfile (`=0`), NOT configurable at runtime, to prevent accidental misconfiguration
5. A comment in the Dockerfile documents the intent: `# Dedicated GPU — disable idle model unload; model persists for pod lifetime`
6. Docker image builds successfully with these env vars set
7. Manual verification: start container, wait 6+ minutes with no requests, confirm ollama process still reports the model loaded (via `/api/tags`)

## Technical Notes

```dockerfile
# In Dockerfile
ENV OLLAMA_KEEP_ALIVE=0
ENV OLLAMA_MODELS=/models
ENV OLLAMA_MODEL=qwen2.5:14b  # Replace with confirmed tag

# Note: OLLAMA_KEEP_ALIVE=0 means "never unload"
# This is intentional — dedicated GPU, no contention
```

**Action required before starting:** Confirm the exact model tag to use for `OLLAMA_MODEL`.
The model must be pullable from the default ollama registry or an internal mirror.

## Definition of Done

- [ ] Dockerfile updated with three ENV vars
- [ ] Hard-coded KEEP_ALIVE with explanatory comment
- [ ] Image builds without errors
- [ ] Manual idle-persistence verification documented in PR
- [ ] Code reviewed and merged to feature branch
