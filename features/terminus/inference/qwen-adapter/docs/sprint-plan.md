---
feature: qwen-adapter
doc_type: sprint-plan
status: draft
goal: "Organize warmup implementation and persistence hardening into delivery sprints"
key_decisions:
  - Sprint 1: Warmup script + ollama config
  - Sprint 2: K8s integration (probes, volumes, env)
  - Sprint 3: Testing and validation
open_questions:
  - What's the optimal story point estimation for each story?
  - Should we have a pre-deployment validation sprint?
depends_on: []
blocks: []
updated_at: '2026-04-26T14:00:00Z'
---

## Story Breakdown

### Story 1: Implement Warmup Script (8 points)

**Title:** Create warmup.py script for model preload on container startup

**Description:**
Implement a standalone Python script that runs during container init to ensure Qwen 3.6 is loaded into GPU memory before the agent accepts traffic.

**Acceptance Criteria:**
- [ ] Script waits for ollama:11434 to be responsive (timeout 60s)
- [ ] Script pulls/validates Qwen 3.6 model via `/api/pull` endpoint
- [ ] Script runs lightweight test query to confirm model is loaded (token generation)
- [ ] Script writes `/tmp/warmup.ready` on success
- [ ] Script logs each step (ollama ready, model pull, test query, completion)
- [ ] Script exits 0 on success, 1 on any failure
- [ ] Script handles missing/unreachable ollama gracefully (clear error messages)
- [ ] Script timeout is 300s (5 minutes) configurable via env var

**Technical Notes:**
- Use `requests` library for HTTP calls to ollama
- Poll `/api/tags` to check ready state
- POST to `/api/pull` with model name (qwen:3.6)
- POST to `/api/generate` with simple prompt ("Hello, how are you?")
- Log to stdout/stderr (k8s captures in pod logs)

**Story Points:** 8

---

### Story 2: Configure Ollama for Persistence (5 points)

**Title:** Harden ollama configuration to prevent idle model shutdown

**Description:**
Update Dockerfile and environment to ensure ollama keeps the Qwen model loaded indefinitely (no idle timeout).

**Acceptance Criteria:**
- [ ] Set `OLLAMA_KEEP_ALIVE=0` in Dockerfile (hard-coded, not env var override)
- [ ] Document why KEEP_ALIVE=0 is required (dedicated GPU, no resource contention)
- [ ] Set `OLLAMA_MODELS=/models` to persist cache in named volume
- [ ] Verify CUDA_VISIBLE_DEVICES is set correctly for GPU binding
- [ ] Test that ollama doesn't shutdown after model load completes
- [ ] Confirm model remains in GPU memory after 10 minutes of inactivity

**Technical Notes:**
- Update Dockerfile ENV section before FROM clause
- Add comment explaining KEEP_ALIVE rationale
- Link to ollama documentation for KEEP_ALIVE parameter
- Verify GPU driver compatibility in image build

**Story Points:** 5

---

### Story 3: Update Container Entrypoint (5 points)

**Title:** Integrate warmup script into container startup sequence

**Description:**
Modify Dockerfile ENTRYPOINT to run warmup script before starting the agent, ensuring model is loaded when agent begins accepting requests.

**Acceptance Criteria:**
- [ ] Dockerfile ENTRYPOINT starts ollama in background
- [ ] Warmup script runs immediately after (blocking)
- [ ] Warmup failure prevents agent startup (exit non-zero)
- [ ] Agent only starts after warmup completes
- [ ] Startup logs show: ollama started → warmup running → agent ready
- [ ] No race conditions between ollama and warmup processes

**Technical Notes:**
- Use shell form: `ENTRYPOINT ["/bin/bash", "-c", "..."]`
- Start ollama with `ollama serve &` in background
- Add `sleep 2` to allow ollama time to bind port
- Run warmup script synchronously (blocking)
- Exec agent to preserve PID 1 (proper signal handling)

**Story Points:** 5

---

### Story 4: Implement Readiness Probe (5 points)

**Title:** Add k8s readiness probe to confirm warmup + model loaded

**Description:**
Update k8s Pod spec with readiness probe that verifies warmup completion and model is resident before marking pod as Ready.

**Acceptance Criteria:**
- [ ] Readiness probe checks for `/tmp/warmup.ready` file existence
- [ ] Readiness probe calls `/health/ready` endpoint (agent-provided)
- [ ] Probe uses `exec` with shell command (no port-based probing)
- [ ] `initialDelaySeconds` set to 10 (allow time for ollama startup)
- [ ] `periodSeconds` set to 5 (frequent checks during warmup)
- [ ] `timeoutSeconds` set to 5 (reasonable timeout for HTTP call)
- [ ] `failureThreshold` set to 3 (allow transient failures)
- [ ] Pod transitions to Ready only after both checks pass

**Technical Notes:**
- Probe command: `test -f /tmp/warmup.ready && curl -f http://localhost:8000/health/ready`
- Use `failureThreshold=3` for resilience (not too strict)
- Document why both file + HTTP check are needed

**Story Points:** 5

---

### Story 5: Implement Liveness Probe (3 points)

**Title:** Add k8s liveness probe to detect ollama/agent crashes

**Description:**
Update k8s Pod spec with liveness probe to verify ollama process is running and responding.

**Acceptance Criteria:**
- [ ] Liveness probe calls `/api/tags` on ollama (http://localhost:11434/api/tags)
- [ ] Probe uses `exec` with curl command
- [ ] `initialDelaySeconds` set to 30 (allow warmup to complete)
- [ ] `periodSeconds` set to 30 (reasonable check interval)
- [ ] `timeoutSeconds` set to 5
- [ ] `failureThreshold` set to 3
- [ ] Probe failure triggers pod restart

**Technical Notes:**
- Probe command: `curl -f http://localhost:11434/api/tags`
- Separate from readiness probe (readiness gates traffic entry; liveness detects runtime crashes)
- k8s restarts pod on liveness failure (triggers warmup again)

**Story Points:** 3

---

### Story 6: Add Health Endpoints to Agent (5 points)

**Title:** Implement /health and /health/ready endpoints for probe feedback

**Description:**
Update agent HTTP server to expose readiness and liveness health check endpoints that readiness/liveness probes can query.

**Acceptance Criteria:**
- [ ] `GET /health` returns 200 with `{"status": "alive"}` (liveness check)
- [ ] `GET /health/ready` returns 200 with `{"status": "ready"}` only after warmup
- [ ] `/health/ready` checks `/tmp/warmup.ready` file existence
- [ ] `/health/ready` optionally verifies model loaded via ollama API
- [ ] `/health/ready` returns 503 if warmup not complete
- [ ] `/health/ready` returns 503 if model not found in ollama
- [ ] Both endpoints respond within 100ms (no long processing)
- [ ] Endpoints return proper JSON content-type header

**Technical Notes:**
- Use FastAPI/Flask async handlers for low latency
- `/health` is simple — no dependencies
- `/health/ready` checks: file existence + optional ollama verification
- Return 503 Service Unavailable (not 404) for "not ready" state

**Story Points:** 5

---

### Story 7: Configure Model Cache Volume (3 points)

**Title:** Set up k8s persistent volume for ollama model cache

**Description:**
Create PersistentVolumeClaim and update Pod spec to mount model cache volume, allowing model to persist across pod restarts.

**Acceptance Criteria:**
- [ ] PersistentVolumeClaim created for model cache (10 GB initial, expandable)
- [ ] Pod spec mounts PVC at `/models`
- [ ] OLLAMA_MODELS env var set to `/models`
- [ ] Model cache persists after pod restart
- [ ] No re-download of Qwen 3.6 after pod restart (use existing cache)
- [ ] Volume access mode allows single pod read-write

**Technical Notes:**
- Use `accessModes: [ReadWriteOnce]` (single pod only)
- StorageClassName: use default or specify GPU-local storage
- Mount path: `/models` (matches OLLAMA_MODELS)
- Qwen 3.6 ~18-20 GB; allocate 25 GB to avoid resize

**Story Points:** 3

---

### Story 8: Test Warmup Script Locally (5 points)

**Title:** Unit test warmup script with mock ollama responses

**Description:**
Write unit tests for warmup.py to validate logic without requiring real ollama/GPU.

**Acceptance Criteria:**
- [ ] Mock `requests.get` for ollama health checks
- [ ] Mock `requests.post` for model pull and test query
- [ ] Test: wait_for_ollama() succeeds after retry
- [ ] Test: wait_for_ollama() times out correctly
- [ ] Test: pull_model() succeeds with valid response
- [ ] Test: pull_model() fails with HTTP error
- [ ] Test: test_generation() succeeds with valid response
- [ ] Test: test_generation() fails gracefully
- [ ] Test: main() writes `/tmp/warmup.ready` on success
- [ ] Test: main() exits 0 on success, 1 on failure
- [ ] Tests run in CI without GPU

**Technical Notes:**
- Use `pytest` + `unittest.mock`
- Mock `os.path.exists()` for `/tmp/warmup.ready` checks
- Test both happy path (success) and error paths
- Mock timeouts and network errors

**Story Points:** 5

---

### Story 9: Integration Test in Dev Environment (8 points)

**Title:** Deploy warmup to dev pod and validate startup behavior

**Description:**
Deploy the warmup script + ollama config + probes to dev environment and validate:
1. Pod starts, warmup runs, model loads
2. First request arrives in <2 seconds
3. Model stays resident (no unload after idle time)
4. Pod logs show expected sequence
5. Metrics/events flow correctly

**Acceptance Criteria:**
- [ ] Dev pod deploys successfully with all config
- [ ] Warmup completes within 5 minutes (log verified)
- [ ] Readiness probe succeeds (pod marked Ready)
- [ ] Service routes traffic successfully
- [ ] First inference request completes in <2 seconds
- [ ] Request latency stable after 10 minutes (no JIT reloads)
- [ ] Startup events visible in k8s describe pod
- [ ] Pod logs show all warmup steps
- [ ] GPU memory usage stable after warmup
- [ ] Metrics (prometheus) show startup duration, model size

**Technical Notes:**
- Deploy to dev namespace
- Monitor pod events: `kubectl describe pod <name>`
- Check pod logs: `kubectl logs <name> -f`
- Monitor metrics: query prometheus for startup_warmup_duration_seconds
- Test: send requests at t=0s, t=5min, t=30min; verify latency

**Story Points:** 8

---

### Story 10: Production Readiness Checklist (3 points)

**Title:** Document and verify production deployment requirements

**Description:**
Verify all components are ready for production deployment, including: GPU resource limits, monitoring, runbook, rollback plan.

**Acceptance Criteria:**
- [ ] Resource limits documented: GPU=1, memory=48Gi
- [ ] Monitoring metrics defined and exposed
- [ ] Health probe thresholds validated in dev
- [ ] Runbook written: startup troubleshooting, known issues, recovery steps
- [ ] Rollback plan documented (previous image, cache recovery)
- [ ] WARN: Verify GPU memory capacity meets Qwen 3.6 requirements
- [ ] WARN: Verify CUDA driver compatibility in target node
- [ ] Load test: 50 concurrent requests during/after warmup
- [ ] Documentation complete: architecture, configuration, troubleshooting

**Technical Notes:**
- GPU memory for Qwen 3.6: minimum 40 GB (quantization-dependent)
- CUDA compute capability must support model
- Runbook includes: log inspection, prometheus queries, recovery procedures
- Rollback: previous image tag + cache volume retention

**Story Points:** 3

---

## Sprint Planning

### Sprint 1: Warmup Core (Start: Week 1)
**Objective:** Implement warmup script and ollama persistence config

- **Story 1** (8 pts): Warmup script
- **Story 2** (5 pts): Ollama persistence config
- **Story 3** (5 pts): Container entrypoint integration

**Total:** 18 points  
**Duration:** 1 week  
**Key outcome:** Container starts and loads model

---

### Sprint 2: K8s Integration (Start: Week 2)
**Objective:** Add k8s probes, health endpoints, and volume config

- **Story 4** (5 pts): Readiness probe
- **Story 5** (3 pts): Liveness probe
- **Story 6** (5 pts): Health endpoints
- **Story 7** (3 pts): Model cache volume
- **Story 8** (5 pts): Unit tests for warmup

**Total:** 21 points  
**Duration:** 1 week  
**Key outcome:** Pod correctly signals ready state; traffic routes after warmup

---

### Sprint 3: Validation & Production (Start: Week 3)
**Objective:** Validate in dev environment, prepare for production

- **Story 9** (8 pts): Dev environment integration test
- **Story 10** (3 pts): Production readiness checklist

**Total:** 11 points  
**Duration:** 1 week  
**Key outcome:** Production-ready deployment validated

---

## Dependency Graph

```
Story 1 (warmup script)  ──→  Story 3 (entrypoint)  ──→  Story 9 (integration test)
                ↓
Story 2 (ollama config)  ──→  Story 3
                        ↓
                    Story 8 (unit tests)

Story 3  ──→  Story 4 (readiness probe)  ──→  Story 9
         ├──→ Story 5 (liveness probe)   ──→  Story 9
         ├──→ Story 6 (health endpoints) ──→  Story 9
         └──→ Story 7 (cache volume)     ──→  Story 9

Story 9  ──→  Story 10 (prod checklist)
```

## High-Level Timeline

| Week | Sprint | Stories | Outcome |
|------|--------|---------|---------|
| 1 | Warmup Core | S1-S3 | Model loads at startup |
| 2 | K8s Integration | S4-S8 | Pod readiness signals, health checks working |
| 3 | Validation | S9-S10 | Dev validation complete, prod checklist verified |

## Success Criteria for Release

✅ All 10 stories completed and tested  
✅ Integration test passes in dev environment  
✅ Warmup completes within 5 minutes  
✅ First request latency <2 seconds (vs 10+ min cold-start)  
✅ Model persists indefinitely (no idle shutdown)  
✅ k8s probes correctly signal readiness/liveness  
✅ Production checklist items completed  
✅ Monitoring/metrics exposed for observability  

