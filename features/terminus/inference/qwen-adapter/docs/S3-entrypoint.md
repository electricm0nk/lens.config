---
feature: terminus-inference-qwen-adapter
story_id: S3
title: Container Entrypoint
epic: E1
sprint: 1
points: 5
status: not-started
assignee: null
depends_on: [S1, S2]
blocks: [S4, S5, S6, S7]
updated_at: '2026-04-26T14:55:00Z'
---

# S3 — Container Entrypoint

## As a
Container operator

## I want
The Dockerfile entrypoint to start ollama in the background, run the warmup script to completion, then start the agent service

## So that
The pod is fully warmed before accepting any traffic, and all three processes have the correct startup order

## Context

This story wires together S1 (warmup.py) and S2 (ollama config) at the container level. The correct startup sequence is:
1. Start `ollama serve` in the background
2. Run `warmup.py` (blocking — waits for ollama, pulls model, runs test query, writes ready marker)
3. Start the agent application

If warmup fails (non-zero exit), the container should exit with an error code so k8s restarts the pod.

## Acceptance Criteria

1. `Dockerfile` `ENTRYPOINT` or `CMD` starts `ollama serve &` (background)
2. Then runs `python3 /app/warmup.py` (blocking) — if warmup exits non-zero, the container exits non-zero immediately
3. Then starts the agent service process (main application)
4. The sequence is implemented in a shell script or Dockerfile `CMD` — not inline, for readability and testability
5. The entrypoint script is at `/app/entrypoint.sh` and is marked executable in the Dockerfile
6. If `OLLAMA_MODEL` env var is not set, the entrypoint exits with a clear error message before starting warmup
7. Container startup logs show the three phases: `[entrypoint] Starting ollama...`, `[entrypoint] Running warmup...`, `[entrypoint] Starting agent...`

## Technical Notes

```bash
#!/bin/bash
# /app/entrypoint.sh

set -e

if [ -z "$OLLAMA_MODEL" ]; then
  echo "[entrypoint] ERROR: OLLAMA_MODEL env var not set" >&2
  exit 1
fi

echo "[entrypoint] Starting ollama..."
ollama serve &

echo "[entrypoint] Running warmup..."
python3 /app/warmup.py

echo "[entrypoint] Starting agent..."
exec python3 /app/main.py  # or whatever the agent start command is
```

```dockerfile
COPY entrypoint.sh /app/entrypoint.sh
RUN chmod +x /app/entrypoint.sh
ENTRYPOINT ["/app/entrypoint.sh"]
```

## Definition of Done

- [ ] `/app/entrypoint.sh` implemented with three-phase sequence
- [ ] Fails fast if `OLLAMA_MODEL` not set
- [ ] Fails fast if warmup exits non-zero
- [ ] `exec` used for agent process (correct PID 1 handoff)
- [ ] Dockerfile updated to use new entrypoint
- [ ] Integration verified: container starts, logs show all three phases
- [ ] Code reviewed and merged to feature branch
