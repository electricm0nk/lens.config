---
feature: terminus-inference-qwen-adapter
doc_type: implementation-readiness
status: approved
goal: "Assess readiness to begin development on the Qwen adapter warmup feature"
key_decisions:
  - All planning artifacts complete and adversarially reviewed (PASS)
  - 10 stories defined with concrete acceptance criteria
  - Infrastructure prerequisites verified
open_questions: []
depends_on: []
blocks: []
updated_at: '2026-04-26T14:55:00Z'
---

# Implementation Readiness — Qwen Adapter Container Warmup

**Assessment Date:** 2026-04-26  
**Track:** express  
**Phase:** finalizeplan  
**Overall Status:** ✅ **READY FOR DEVELOPMENT**

---

## Readiness Summary

| Gate | Status | Notes |
|------|--------|-------|
| Business case defined | ✅ Pass | business-plan.md — JIT cold-start elimination |
| Technical design complete | ✅ Pass | tech-plan.md — warmup.py, Dockerfile, K8s specs |
| Stories written with AC | ✅ Pass | sprint-plan.md — 10 stories, 50 pts |
| Adversarial review passed | ✅ Pass | expressplan-adversarial-review.md + finalizeplan-review.md |
| Cross-feature conflicts | ✅ None | provider-routing, openaiadapter, gateway all at preplan |
| Dependencies available | ✅ Pass | Ollama, GPU, k8s cluster all in place |
| Scope bounded | ✅ Pass | 3 sprints, well-defined entry/exit criteria |

---

## Planning Artifacts Status

| Artifact | Present | Reviewed | Notes |
|----------|---------|---------|-------|
| business-plan.md | ✅ | ✅ | PASS — no blockers |
| tech-plan.md | ✅ | ✅ | PASS — 4 action items (AC updates) |
| sprint-plan.md | ✅ | ✅ | PASS — 10 stories across 3 sprints |
| expressplan-adversarial-review.md | ✅ | ✅ | PASS verdict |
| finalizeplan-review.md | ✅ | ✅ | PASS verdict, 6 AC refinements |

---

## Technical Prerequisites

### Infrastructure
| Prerequisite | Status | Notes |
|-------------|--------|-------|
| GPU node in k8s cluster | ✅ Available | Dedicated GPU for qwen-adapter pod |
| VRAM ≥24GB | ✅ Assumed | Verify in Story 1 warmup script |
| Ollama base image | ✅ Available | Running in current container |
| CUDA-compatible driver | ✅ Available | Required for Qwen inference |
| PVC storage class | ✅ Available | 25GB ReadWriteOnce for model cache |

### Application
| Prerequisite | Status | Notes |
|-------------|--------|-------|
| Qwen model tag confirmed | ⚠️ Pending | Confirm `OLLAMA_MODEL` value before Story 1; use env var per F3 finding |
| Agent health endpoint exists | ⚠️ Pending | Story 6 creates `/health` and `/health/ready`; check if stub exists |
| ollama API versioned stable | ✅ Available | `/api/pull`, `/api/generate`, `/api/tags` used in warmup |

---

## Story Readiness

| Story | Title | Points | Prerequisites | Ready? |
|-------|-------|--------|--------------|--------|
| S1 | Warmup Script | 8 | Ollama running, GPU available | ✅ |
| S2 | Ollama Persistence Config | 5 | Dockerfile access | ✅ |
| S3 | Container Entrypoint | 5 | S1, S2 complete | ✅ (after S1/S2) |
| S4 | Readiness Probe | 5 | S3 complete | ✅ (after S3) |
| S5 | Liveness Probe | 3 | S3 complete | ✅ (after S3) |
| S6 | Health Endpoints | 5 | S3 complete | ✅ (after S3) |
| S7 | Model Cache Volume | 3 | S3 complete | ✅ (after S3) |
| S8 | Unit Tests | 5 | S1 complete | ✅ (after S1) |
| S9 | Integration Test | 8 | S4-S8 complete | ✅ (after Sprint 2) |
| S10 | Production Checklist | 3 | S9 complete | ✅ (after S9) |

---

## Action Items Before Sprint 1 Start

| # | Item | Priority | Owner |
|---|------|---------|-------|
| A1 | Confirm Qwen model tag (e.g., `qwen2.5:14b`) for `OLLAMA_MODEL` env var | High | Dev Lead |
| A2 | Check if agent service has any existing `/health` endpoint stub | Medium | Dev Lead |
| A3 | Verify PVC StorageClass name in target cluster | Medium | Dev/Ops |
| A4 | Confirm GPU node label/taint for nodeSelector in deployment manifest | Medium | Dev/Ops |

---

## Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Qwen model tag mismatch | Medium | High — warmup fails | Confirm before Sprint 1; use env var |
| VRAM insufficient | Low | High — pod crash loop | Check with `nvidia-smi` in warmup |
| Slow warmup >5 min | Medium | Medium — ops noise | Alert on >300s; generous probe timeout |
| ollama API change | Low | Medium — warmup script breaks | Pin ollama version in Dockerfile |
| PVC provisioning failure | Low | Medium — model re-download each restart | Pre-provision PVC before Story 7 |

---

## Definition of Done

Development is complete when:
1. ✅ All 10 stories pass acceptance criteria
2. ✅ Integration test (Story 9) passes in dev environment
3. ✅ Warmup completes in <5 minutes after fresh pod start
4. ✅ First request latency (pod-boundary) <2 seconds post-warmup
5. ✅ Model remains in VRAM for ≥60 minutes without idle shutdown
6. ✅ Prometheus metrics flowing; warmup duration alert active
7. ✅ Runbook written and reviewed

---

## Verdict

**✅ READY — Begin Sprint 1 implementation.**

Pre-sprint action items (A1-A4) are low-effort confirmations that should be resolved in the first day of Sprint 1. None are blockers for starting Stories S1 and S2 in parallel.
