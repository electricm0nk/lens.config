---
feature: terminus-inference-omniroute-gateway-migration
story_id: OMNI-3.2
doc_type: story
status: not-started
title: Ingress Cutover + Parallel Window
depends_on: [OMNI-2.2, OMNI-2.3, OMNI-3.1]
updated_at: "2026-05-25"
sprint: 3
target_repo: TargetProjects/terminus/infra/terminus.infra
---

# OMNI-3.2: Ingress Cutover + Parallel Window

**Epic:** 3 — Cutover and Decommission  
**Sprint:** 3  
**Status:** not-started

## User Story

*As a consumer, my requests to `https://inference-gateway.trantor.internal/v1` are served by OmniRoute so that the migration is live.*

## Acceptance Criteria

- [ ] `terminus-inference-gateway` deployment gracefully drained: existing long-polling connections allowed to complete (up to 300s timeout) before ingress is re-pointed
- [ ] `terminus-inference-gateway` deployment scaled to 0 replicas (NOT deleted — held for rollback window)
- [ ] End-to-end smoke test passes: authenticated request through ingress → OmniRoute → llamacpp → response
- [ ] **All named consumers verified:** Watchdog, Kaylee, and Hermes each execute at least one successful request through OmniRoute before the parallel window begins
- [ ] All five route-profile combos are reachable via the public ingress endpoint
- [ ] Rollback verified: scaling `terminus-inference-gateway` back to 1 and re-patching ingress restores previous behavior in < 5 minutes
- [ ] 24h parallel window starts only after all three named consumers verified

## BDD Scenario (Rollback)

```
Given OmniRoute is live behind ingress
And terminus-inference-gateway is scaled to 0
When an operator scales terminus-inference-gateway to 1
And patches the ingress backend to terminus-inference-gateway
Then a smoke test request returns a valid OpenAI response from the legacy gateway
And the rollback completes in < 5 minutes
```

## Implementation Notes

- Drain command: patch `terminus-inference-gateway` deployment `terminationGracePeriodSeconds: 300`
- Hold at 0 replicas for at least 24h after cutover before proceeding to Story 3.3
- Consumer verification: reach out to Watchdog, Kaylee, and Hermes team owners (Todd Hintzmann operates all three) to confirm at least one real inference request completes through OmniRoute

## Dependencies

- OMNI-2.2 (contract tests green against OmniRoute)
- OMNI-2.3 (circuit breaker and batch hard-fail validated)
- OMNI-3.1 (architecture documentation reviewed and approved)
