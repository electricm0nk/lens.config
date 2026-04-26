---
feature: terminus-inference-qwen-adapter
doc_type: expressplan-adversarial-review
status: approved
gate: expressplan-completion
verdict: pass
finding_count: 2
finding_severity: info
critical_findings: 0
high_findings: 0
medium_findings: 0
low_findings: 2
updated_at: '2026-04-26T14:30:00Z'
---

# ExpressPlan Adversarial Review: Qwen Adapter Container Warmup

**Feature:** terminus-inference-qwen-adapter  
**Phase:** expressplan  
**Source:** phase-complete  
**Verdict:** **PASS** ✅

---

## Review Summary

The expressplan artifacts (business-plan.md, tech-plan.md, sprint-plan.md) define a coherent, well-scoped initiative to eliminate JIT cold-start delays by preloading Qwen 3.6 into GPU memory and disabling ollama idle shutdown.

**What's Strong:**
- Clear problem statement and business justification (10+ minute cold-start → instant response)
- Concrete technical approach: warmup script, KEEP_ALIVE=0 configuration, k8s probes
- Detailed implementation design with Dockerfile, Python script pseudocode, and probe specifications
- Comprehensive test strategy across unit, integration, and production validation
- 10 concrete stories organized into 3 sequential sprints (18+21+11 points = 50 pts total)
- Proper risk identification (OOM, slow warmup, config drift, model mismatch)

**Findings:**
- **[INFO]** Qwen 3.6 memory requirement (20-40 GB) should be explicitly validated in story acceptance criteria for AWS/GKE node compatibility
- **[INFO]** Rollback strategy is mentioned but not linked to a deployment PR process — consider documenting approval gate

---

## Detailed Analysis

### 1. Logic Flaws & Coverage

**Business Logic:**
✅ Problem diagnosis is sound: JIT loading on dedicated GPU wastes resources  
✅ Solution aligns with constraint: dedicated GPU means indefinite persistence is valid  
✅ Success metrics are measurable: startup time, first-request latency, model residency

**Technical Logic:**
✅ Warmup script design is sound: poll → pull → test → mark-ready sequence is correct  
✅ KEEP_ALIVE=0 is the right lever to disable ollama idle shutdown  
✅ Two-probe strategy (readiness + liveness) is architecturally correct  
✅ Model cache volume prevents re-download on pod restart  

**Coverage:**
✅ Startup phase covered (warmup + probes)  
✅ Runtime phase covered (KEEP_ALIVE + model persistence)  
✅ Graceful shutdown covered (allow inflight requests, model unload on termination)  
✅ Failure modes covered (OOM, slow warmup, model corruption)  

### 2. Gaps & Missing Considerations

**Gap 1: GPU Memory Validation**
- **What's missing:** Story 10 (production checklist) mentions GPU memory ≥40GB requirement, but Story 1 (warmup script) doesn't validate this at startup
- **Impact:** If node has insufficient GPU memory, warmup could OOM and fail silently or hang
- **Recommendation:** Add explicit check in warmup script: `nvidia-smi | grep Free` before model pull; fail fast if < 40GB available
- **Severity:** Low (caught in integration testing)
- **Story to update:** Story 1 acceptance criteria

**Gap 2: Warmup Retry Policy**
- **What's missing:** Tech plan says "fail fast on warmup," but doesn't specify retry behavior for transient failures (network timeout, ollama slow start)
- **Impact:** Single timeout during container startup could unnecessarily fail the pod
- **Recommendation:** Add exponential backoff (1s, 2s, 4s) for transient failures; fail only after 3 retries; log each attempt
- **Severity:** Low (readiness probe can retry the whole pod, but user experience is degraded)
- **Story to update:** Story 1 acceptance criteria

**Gap 3: Metrics/Observability Completeness**
- **What's missing:** Tech plan defines metrics (startup_warmup_duration_seconds, gpu_memory_used_bytes), but no story covers prometheus integration or Grafana dashboard
- **Impact:** Operators can't trend coldstart improvements or spot performance regressions
- **Recommendation:** Add a story for metrics instrumentation (prometheus client library, dashboard)
- **Severity:** Low (dashboarding can be a follow-up after initial release)

### 3. Complexity & Risk Assessment

| Risk | Severity | Likelihood | Mitigation |
|------|----------|-----------|-----------|
| OOM on warmup (GPU memory insufficient) | High | Medium | GPU memory validation in warmup script; fail fast |
| Warmup timeout (slow GPU/model) | Medium | Low | Generous timeout (5 min), readiness probe retry |
| Config drift (idle timeout re-enabled) | Medium | Low | Hard-code KEEP_ALIVE=0 in Dockerfile |
| Model corruption on restart | Medium | Low | Checksum validation in warmup; cache volume persists |
| Ollama process crash | Low | Low | Liveness probe detects; k8s restarts pod |

**Overall Complexity:** Moderate. Three components (warmup script, ollama config, k8s probes) are well-designed and testable. No architectural surprises.

### 4. Cross-Feature Dependencies

**Related Features:**
- `terminus-inference-gateway` — provides the host for this adapter; no direct dependency, but shares GPU resource pool
- `terminus-inference-openaiadapter` — precedent for adapter pattern; similar warmup approach would be consistent

**Governance Dependencies:**
- None identified. Feature is isolated to this adapter pod.

**Constitutional Implications:**
- ✅ No language-specific patterns violated
- ✅ Resource limits follow kubernetes best practices
- ✅ No security/auth changes required

### 5. Assumptions & Blind Spots

**Articulated Assumptions:**
1. ✅ Dedicated GPU means no other workloads compete (stated in business plan)
2. ✅ Qwen 3.6 fits in GPU VRAM (40 GB requirement stated)
3. ✅ CUDA driver compatible with Qwen inference (mentioned in risks)
4. ✅ ollama HTTP API stable (multiple POST calls depend on this)

**Potential Blind Spots (Party-Mode Challenge):**

**Winston (Architect):**  
"I like the warmup approach, but I'm wondering about the model quantization assumption. Qwen 3.6 has multiple quantization variants (fp16, int8, int4). The tech plan doesn't specify which quantization to use in the warmup script — if someone runs a different quantization in production, the memory footprint changes dramatically. Should we parameterize this in the warmup script, or is there a standard quantization we're committing to?"

**Sally (UX/Operations):**  
"One thing I didn't see: what happens if warmup completes but then the model gets evicted by something else (OOMKiller, k8s node pressure)? The `/tmp/warmup.ready` file would still be there, but the model wouldn't be loaded. The readiness probe checks the file, not the actual vram state. Should we enhance the readiness probe to verify the model is actually in memory, not just trust the file?"

**Mary (Analyst):**  
"From a rollout perspective: are we planning gradual rollout (canary) or immediate full deployment? If a bug in the warmup script blocks all pods from starting, we have a full outage. Should we consider a feature flag or gradual deployment strategy?"

---

## Blind-Spot Challenge Questions

After considering the above perspectives, here are critical questions for the team:

1. **Quantization Strategy:** Which Qwen 3.6 quantization will we use in production? Should the warmup script accept it as a parameter, or is there a standard we're enforcing?

2. **Model Residency Verification:** Should the `/health/ready` endpoint verify the model is actually in GPU memory (via ollama API), not just trust the warmup ready file?

3. **Deployment Strategy:** How will we roll out this change without risking full-outage if warmup fails? Canary deployment? Feature flag? Blue-green testing?

4. **Post-Eviction Recovery:** If the model gets evicted from GPU memory after warmup completes, how do we detect and recover? Should there be a periodic health check that re-runs warmup if model is missing?

5. **Monitoring Gaps:** Are we instrumenting enough observability to catch production issues (memory pressure, warmup timeouts, model load failures) in real time?

---

## Findings Summary

| Severity | Count | Description |
|----------|-------|-------------|
| Critical | 0 | No blocking issues |
| High | 0 | No architectural concerns |
| Medium | 0 | No mandatory rework items |
| **Low** | **2** | GPU memory validation in warmup; retry policy for transient failures |
| Info | 0 | Blind-spot questions identified (escalate to team discussion) |

---

## Verdict

**✅ PASS**

The expressplan artifacts are well-structured, comprehensive, and ready for advancement to FinalizePlan. The business case is sound, the technical design is feasible, and the implementation roadmap is clear.

**Action Items:**
1. Address low-severity findings in Story 1 (GPU memory validation, retry logic)
2. Schedule team sync to discuss blind-spot questions (quantization strategy, model residency verification, deployment strategy)
3. Proceed to FinalizePlan for final governance bundle and PR handoff

**Approval:** ✅ Ready for auto-advance to `/finalizeplan`

