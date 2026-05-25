---
feature: terminus-inference-omniroute-gateway-migration
story_id: OMNI-1.2
doc_type: story
status: not-started
title: llamacpp Provider Registration
depends_on: [OMNI-1.1]
updated_at: "2026-05-25"
sprint: 1
target_repo: TargetProjects/terminus/infra/terminus.infra
---

# OMNI-1.2: llamacpp Provider Registration

**Epic:** 1 — OmniRoute Deployment and Wiring  
**Sprint:** 1  
**Status:** not-started

## User Story

*As a platform operator, I can register the llamacpp backend as a custom OpenAI-compatible provider in OmniRoute so that inference requests route to the local GPU worker.*

## Acceptance Criteria

- [ ] llamacpp registered in OmniRoute as provider `llamacpp-local` at `http://llamacpp.inference.svc.cluster.local:9090`
- [ ] `POST /v1/chat/completions` through OmniRoute returns a valid OpenAI-shaped response using llamacpp
- [ ] Provider registration is exported as committed config (not dashboard-only)
- [ ] No credentials required for llamacpp-local (internal endpoint)

## Implementation Notes

- llamacpp is OpenAI-compatible (`/v1/chat/completions`, `/v1/models`)
- No auth headers required — `llamacpp.inference.svc.cluster.local` is cluster-internal
- Export config from OmniRoute data volume, validate it matches what was entered via the dashboard
- Model context: Qwen3.6-35B-A3B Q4_K_M, context 98304, `parallel_slots=1`

## Dependencies

- OMNI-1.1 (OmniRoute running and accessible)

## Definition of Done

llamacpp visible in OmniRoute provider list, test chat request returns valid response, provider config committed.
