---
feature: terminus-inference-qwen-adapter
doc_type: finalizeplan-review
status: approved
goal: "Final cross-artifact review and governance impact check before FinalizePlan bundle handoff"
key_decisions:
  - Warmup script approach is sound; proceed to implementation
  - OLLAMA_KEEP_ALIVE=0 is correct for dedicated GPU
  - K8s readiness/liveness probe split is appropriate
  - PVC for /models cache is required for warm restarts
open_questions: []
depends_on: []
blocks: []
updated_at: '2026-04-26T14:45:00Z'
---

# FinalizePlan Review — Qwen Adapter Container Warmup

**Review Date:** 2026-04-26  
**Phase:** FinalizePlan (express track)  
**Verdict:** ✅ **PASS — Proceed to Bundle**  
**Reviewers:** Adversarial panel + party-mode challenge (Winston, Sally, Mary)

---

## Review Scope

This review covers the full expressplan artifact set:

| Artifact | Status | Verdict |
|----------|--------|---------|
| `business-plan.md` | Present | ✅ Pass |
| `tech-plan.md` | Present | ✅ Pass |
| `sprint-plan.md` | Present | ✅ Pass |
| `expressplan-adversarial-review.md` | Present | ✅ Pass (prior gate) |

---

## Summary of Prior Review (expressplan gate)

The expressplan adversarial review (PASS, 0 critical, 0 high, 0 medium, 2 low) established:

- Clear, concrete business problem (JIT cold-start on dedicated GPU)
- Sound technical approach (warmup script + KEEP_ALIVE=0 + readiness probes)
- 10 stories across 3 sprints, well-defined acceptance criteria
- Two open items (low severity): GPU memory validation in Story 1, retry policy for warmup

---

## Final Adversarial Review

### Re-Evaluation: Prior Low-Severity Findings

**Finding 1 — GPU Memory Validation (Story 1)**
- Assessment: Sprint-plan Story 1 includes basic GPU CUDA check but doesn't specify minimum memory threshold
- Resolution: Story 1 AC should explicitly require `nvidia-smi --query-gpu=memory.free,memory.total` check and fail-fast if available VRAM < 24GB (floor for Qwen 3.6 Q4)
- Status: **Low — add to Story 1 AC update at bundle time**

**Finding 2 — Warmup Retry Policy**
- Assessment: Business plan mentions retry with backoff; tech plan says "fail fast"; sprint plan is silent
- Resolution: Align to single-attempt with clear timeout (300s), no retry. Rationale: if warmup fails, that's a configuration/hardware problem that retrying won't fix. Pod restart (via liveness) handles recovery.
- Status: **Resolved — fail-fast is correct; document in Story 1 AC**

---

## New Findings — Final Review Pass

### No New Critical or High Findings

The artifact set is internally consistent. The business-plan ↔ tech-plan alignment is clean. Sprint-plan stories map 1:1 to tech-plan implementation steps.

### Low Findings (Final Pass)

**F3 — Model Name Parameterization**
- tech-plan hard-codes `qwen:3.6` in warmup script but Qwen naming in ollama varies by quantization (e.g., `qwen2.5:7b`, `qwen2.5:14b`, `qwen3:8b`)
- Recommendation: Use `OLLAMA_MODEL` env var in Dockerfile, defaulting to `qwen2.5:14b` or the confirmed model tag
- Severity: Low — simple to resolve in Story 2 (Dockerfile env vars)

**F4 — Readiness Probe Initial Delay**
- tech-plan specifies readiness probe with `initialDelaySeconds: 30` but warmup may take up to 300 seconds
- Recommendation: Set `initialDelaySeconds: 60`, `periodSeconds: 10`, `failureThreshold: 30` to allow up to 360s total warmup window
- Severity: Low — update Story 4 AC

---

## Party-Mode Blind-Spot Challenge

**Winston (Architect):**
> "The warmup architecture assumes ollama is already running when warmup.py starts. Is there a race condition risk between `ollama serve` launching in the background and warmup.py polling? Story 3 (Entrypoint) should explicitly handle the `wait_for_ollama` timeout before any pull/test steps. The tech-plan already has this, but Story 3 AC should call it out explicitly."

**Sally (Operations):**
> "What's the operator story if warmup fails during production? The runbook is in Story 10 but there's no alert defined. I'd add a Prometheus alert rule on `startup_warmup_duration_seconds > 300` to detect silent warmup failures. Flag it for Story 9 integration test."

**Mary (Analyst):**
> "The success criteria say 'first request <2 sec' — but is that measured at the API gateway, inside the pod, or at the client? This matters for SLA definitions. I'd recommend measuring at the pod boundary (inside k8s, gateway-to-pod) to separate network latency from inference latency. Document the measurement point in Story 9 AC."

**Blind-Spot Summary:** Three constructive items, none blocking. All can be addressed in story AC updates during the bundle phase.

---

## Governance Impact Assessment

### Cross-Feature Scan — Terminus Inference Domain

| Feature | Phase | Impact |
|---------|-------|--------|
| `provider-routing` | preplan | None — different routing layer, no shared warmup infra |
| `openaiadapter` | preplan | None — separate adapter pod; qwen-specific pattern could be template-reused later |
| `gateway` | preplan | None — gateway is upstream; qwen pod is leaf node |

**Verdict:** No cross-feature conflicts or shared-dependency risks identified. The qwen-adapter changes are fully isolated to the Qwen container image and its k8s deployment manifest.

### Potential Future Reuse (informational)
- The warmup pattern (init script + KEEP_ALIVE config + PVC model cache) could become a reusable adapter template if/when `openaiadapter` or future adapter features need similar warm-start behavior
- No action required in this feature, but the pattern is worth noting in docs

---

## Action Items for Bundle Phase

The following should be addressed in story AC during bundle creation (no blockers):

| # | Item | Target Story |
|---|------|-------------|
| A1 | Add explicit VRAM minimum check (≥24GB) to warmup.py | Story 1 |
| A2 | Document fail-fast (single-attempt, 300s timeout) in warmup retry policy | Story 1 |
| A3 | Replace hard-coded `qwen:3.6` with `OLLAMA_MODEL` env var | Story 2 |
| A4 | Update readiness probe: `initialDelaySeconds: 60`, `failureThreshold: 30` | Story 4 |
| A5 | Add Prometheus alert on `startup_warmup_duration_seconds > 300` | Story 9 |
| A6 | Define first-request latency measurement point (pod-boundary) in integration test | Story 9 |

---

## Final Verdict

| Gate | Result |
|------|--------|
| Artifact completeness | ✅ All 3 required artifacts present |
| Prior review gate | ✅ PASS (0 critical/high/medium, 2 low resolved) |
| New critical/high findings | ✅ None |
| Cross-feature conflicts | ✅ None identified |
| Party-mode blind-spots | ✅ 3 constructive items — addressable in bundle |
| Ready for downstream bundle | ✅ **YES** |

**Overall Verdict: ✅ PASS — Proceed to epics, stories, implementation-readiness, sprint-status, and story files.**
