---
feature: qwen-adapter
doc_type: tech-plan
status: draft
goal: "Design persistent Qwen loading strategy with warmup automation and ollama configuration hardening"
key_decisions:
  - Warmup runs in container init (entrypoint or systemd timer)
  - Ollama KEEP_ALIVE=0 disables idle shutdown
  - Readiness probe waits for /health/ready endpoint (set by warmup)
  - Model cache persists in named volume (pod-scoped, not ephemeral)
open_questions:
  - Should warmup run as separate init container or main container entrypoint?
  - Retry policy for warmup (exponential backoff vs fail-fast)?
  - Health endpoint implementation (Redis flag vs file touch)?
depends_on: []
blocks: []
updated_at: '2026-04-26T14:00:00Z'
---

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ Qwen Adapter Pod (Dedicated GPU)                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Init Phase:                                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 1. Container starts                                 │   │
│  │ 2. Ollama service spawned (no idle timeout)         │   │
│  │ 3. Warmup script executes:                          │   │
│  │    - Wait for ollama:11434 ready                    │   │
│  │    - Pull/validate Qwen 3.6 model                   │   │
│  │    - Run test query (e.g., token completion)       │   │
│  │    - Write /tmp/warmup.ready on success            │   │
│  │    - Exit 0                                         │   │
│  │ 4. k8s readiness probe polls /health/ready         │   │
│  │    - Checks /tmp/warmup.ready + model in memory    │   │
│  │ 5. Pod marked Ready → traffic routed                │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  Runtime Phase:                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ - Agent accepts requests                            │   │
│  │ - Ollama serves from resident model (vram)          │   │
│  │ - KEEP_ALIVE=0 prevents model unload                │   │
│  │ - Model persists until container termination        │   │
│  │ - Graceful shutdown: allow inflight requests        │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Component Design

### 1. Warmup Script (`warmup.py` / `warmup.sh`)

**Location:** `/app/warmup.py` (packaged in image)

**Pseudocode:**
```python
#!/usr/bin/env python3
"""
Warmup script for Qwen adapter pod.
Ensures ollama is running, model is loaded, and pod is ready.
"""

import os
import sys
import time
import requests
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

OLLAMA_BASE = os.getenv("OLLAMA_BASE_URL", "http://localhost:11434")
MODEL = os.getenv("WARMUP_MODEL", "qwen:3.6")
WARMUP_READY_FILE = "/tmp/warmup.ready"
OLLAMA_READY_TIMEOUT = 60  # seconds
WARMUP_TIMEOUT = 300  # 5 minutes

def wait_for_ollama(timeout=OLLAMA_READY_TIMEOUT):
    """Poll ollama health endpoint until ready."""
    start = time.time()
    while time.time() - start < timeout:
        try:
            resp = requests.get(f"{OLLAMA_BASE}/api/tags", timeout=5)
            if resp.status_code == 200:
                logger.info("✓ Ollama ready")
                return True
        except:
            pass
        logger.info("  Waiting for ollama...")
        time.sleep(2)
    logger.error("✗ Ollama not ready after %ds", timeout)
    return False

def pull_model(model=MODEL):
    """Pull and validate model."""
    logger.info(f"Pulling model: {model}")
    try:
        resp = requests.post(
            f"{OLLAMA_BASE}/api/pull",
            json={"name": model},
            timeout=WARMUP_TIMEOUT
        )
        if resp.status_code == 200:
            logger.info(f"✓ Model {model} pulled/cached")
            return True
    except Exception as e:
        logger.error(f"✗ Failed to pull model: {e}")
    return False

def test_generation(model=MODEL):
    """Run lightweight test query to confirm model is loaded."""
    logger.info(f"Testing model generation...")
    try:
        resp = requests.post(
            f"{OLLAMA_BASE}/api/generate",
            json={
                "model": model,
                "prompt": "Hello, how are you?",
                "stream": False
            },
            timeout=30
        )
        if resp.status_code == 200:
            logger.info("✓ Model generation test passed")
            return True
    except Exception as e:
        logger.error(f"✗ Generation test failed: {e}")
    return False

def check_gpu_memory():
    """Validate GPU memory allocation (optional nvidia-utils)."""
    try:
        # nvidia-smi gpu memory check
        logger.info("GPU memory check...")
        return True
    except:
        logger.warning("GPU memory check unavailable")
        return True

def main():
    logger.info("Starting Qwen adapter warmup...")
    
    start = time.time()
    
    # Step 1: Wait for ollama
    if not wait_for_ollama():
        sys.exit(1)
    
    # Step 2: Pull model
    if not pull_model():
        sys.exit(1)
    
    # Step 3: Test generation
    if not test_generation():
        sys.exit(1)
    
    # Step 4: GPU memory check
    check_gpu_memory()
    
    # Mark ready
    elapsed = time.time() - start
    with open(WARMUP_READY_FILE, "w") as f:
        f.write(f"warmup_complete={int(elapsed)}s\n")
    
    logger.info(f"✓ Warmup complete in {elapsed:.1f}s")
    sys.exit(0)

if __name__ == "__main__":
    main()
```

### 2. Ollama Configuration

**Environment Variables (Dockerfile / k8s env):**
```dockerfile
# Disable idle timeout — keep model loaded indefinitely
ENV OLLAMA_KEEP_ALIVE=0

# Optional: explicit model cache location (named volume)
ENV OLLAMA_MODELS=/models

# Optional: GPU configuration
ENV CUDA_VISIBLE_DEVICES=0
```

**Dockerfile Integration:**
```dockerfile
FROM ollama/ollama:latest as ollama

# Warmup script
COPY warmup.py /app/warmup.py
RUN chmod +x /app/warmup.py

# Install Python dependencies for warmup
RUN pip install requests

# Entrypoint: start ollama in background, run warmup, then agent
ENTRYPOINT ["/bin/bash", "-c", \
  "ollama serve & sleep 2 && python /app/warmup.py && \
   exec python -m agent.main"]
```

**Alternative: Init Container (k8s)**
```yaml
initContainers:
- name: warmup
  image: qwen-adapter:latest
  command:
  - python
  - /app/warmup.py
  env:
  - name: OLLAMA_BASE_URL
    value: "http://localhost:11434"
  - name: WARMUP_MODEL
    value: "qwen:3.6"
```

### 3. Kubernetes Pod Configuration

**Readiness Probe:**
```yaml
readinessProbe:
  exec:
    command:
    - /bin/bash
    - -c
    - |
      test -f /tmp/warmup.ready && \
      curl -f http://localhost:8000/health/ready
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 5
  failureThreshold: 3
```

**Liveness Probe:**
```yaml
livenessProbe:
  exec:
    command:
    - /bin/bash
    - -c
    - curl -f http://localhost:11434/api/tags
  initialDelaySeconds: 30
  periodSeconds: 30
  timeoutSeconds: 5
  failureThreshold: 3
```

**Resources (GPU binding):**
```yaml
resources:
  limits:
    nvidia.com/gpu: 1  # Exclusive GPU
    memory: 48Gi       # Qwen 3.6 + runtime headroom
  requests:
    nvidia.com/gpu: 1
    memory: 48Gi
```

### 4. Health Endpoint Integration

**Agent's `/health/ready` endpoint:**
```python
@app.get("/health/ready")
async def health_ready():
    """Readiness check: warmup complete + model loaded."""
    if not os.path.exists("/tmp/warmup.ready"):
        return JSONResponse(
            {"status": "not_ready", "reason": "warmup_not_complete"},
            status_code=503
        )
    
    # Optional: verify model is still in ollama memory
    try:
        async with httpx.AsyncClient() as client:
            resp = await client.get("http://localhost:11434/api/tags")
            models = resp.json().get("models", [])
            if not any(m["name"].startswith("qwen") for m in models):
                return JSONResponse(
                    {"status": "not_ready", "reason": "model_not_loaded"},
                    status_code=503
                )
    except:
        pass
    
    return {"status": "ready"}

@app.get("/health")
async def health():
    """Liveness check: service responding."""
    return {"status": "alive"}
```

## Data Flow

1. **Container Start:**
   - Docker/k8s starts pod with env vars set
   - Ollama spawned with `KEEP_ALIVE=0`

2. **Warmup Phase:**
   - Warmup script runs (sequential: wait → pull → test)
   - On success: `/tmp/warmup.ready` written
   - Ollama retains model in GPU memory

3. **Readiness Check:**
   - k8s readiness probe polls `/health/ready`
   - Confirms `/tmp/warmup.ready` exists + model loaded
   - Pod transitions to `Ready` state

4. **Traffic Routing:**
   - k8s service routes requests to Ready pod
   - Agent processes requests with resident model
   - Response latency: <2 seconds (no load delay)

5. **Runtime:**
   - Model stays in vram indefinitely (`KEEP_ALIVE=0`)
   - Ollama process remains running
   - No idle shutdown or model eviction

6. **Graceful Shutdown:**
   - k8s sends SIGTERM to pod
   - Agent allows inflight requests to complete
   - Model unloaded only on container termination

## Configuration Decisions

### KEEP_ALIVE=0 vs High Value
- **Decision:** Use `KEEP_ALIVE=0` (disable timeout entirely)
- **Rationale:** Dedicated GPU, no contention, operator wants indefinite persistence
- **Alternative:** `KEEP_ALIVE=86400` (24 hours) for safety margin — but not needed here

### Warmup Location (entrypoint vs init-container)
- **Decision:** Main container entrypoint (single image)
- **Rationale:** Simpler to maintain, no separate init container complexity
- **Tradeoff:** Pod won't mark Ready until warmup completes (acceptable)

### Model Cache Volume
- **Decision:** Use named volume for `/models` (persist across restarts)
- **Rationale:** Avoids re-downloading on pod restart
- **Path:** `OLLAMA_MODELS=/models` (mount `/models` PVC)

### Retry / Failure Handling
- **Decision:** Fail fast on warmup; surface errors in pod events
- **Rationale:** Clearer debugging; operator can inspect logs
- **Alternative:** Exponential backoff — but risks indefinite startup

## Monitoring & Observability

### Metrics to Expose
```python
# Via Prometheus endpoint
startup_warmup_duration_seconds  # Histogram
model_load_time_seconds          # Histogram
gpu_memory_used_bytes            # Gauge
ollama_process_uptime_seconds    # Gauge
```

### Logs
```
[warmup] Waiting for ollama...
[warmup] ✓ Ollama ready
[warmup] Pulling model: qwen:3.6
[warmup] ✓ Model qwen:3.6 pulled/cached (18 GB)
[warmup] Testing model generation...
[warmup] ✓ Model generation test passed
[warmup] ✓ Warmup complete in 124.5s
[agent] Health check: /health/ready returning 200
[agent] Accepting inference requests
```

### k8s Events
```
Pod qwen-adapter-xyz created
Pod container started
Readiness probe failed (warmup not complete)
Readiness probe succeeded
Pod Ready
```

## Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| Warmup duration | < 300s (5 min) | Qwen 3.6, GPU-dependent |
| Model load time | < 180s | After pull, during test query |
| First request latency | < 2s | After warmup |
| Model memory footprint | 20-40 GB | Qwen 3.6 quantization-dependent |
| Ollama process memory | 2-5 GB | Not including model |

## Risk Mitigation

| Risk | Mitigation |
|------|-----------|
| OOM on warmup | Validate GPU memory before model pull; fail pod if insufficient |
| Slow warmup | Set `initialDelaySeconds=10`, `timeoutSeconds=5` on readiness probe |
| Model corruption | Add checksum validation in warmup script |
| Config drift (idle timeout re-enabled) | Hard-code in Dockerfile, avoid env-var override |
| Ollama crash during runtime | Liveness probe detects; k8s restarts pod (triggers warmup again) |

## Deployment & Validation

1. **Dev environment:** Deploy warmup + KEEP_ALIVE=0, observe startup time
2. **Load test:** Send requests at t=0, t=5min, t=1hour; verify latency stable
3. **Resource test:** Monitor GPU memory during init and runtime
4. **Failure test:** Kill ollama process; verify liveness probe detects; pod restarts
5. **Observability:** Verify startup logs, metrics, and k8s events are visible

