---
feature: terminus-inference-omniroute-gateway-migration
story_id: OMNI-1.3
doc_type: story
status: not-started
title: Route Profile Combos (All 5)
depends_on: [OMNI-1.2]
updated_at: "2026-05-25"
sprint: 1
target_repo: TargetProjects/terminus/infra/terminus.infra
---

# OMNI-1.3: Route Profile Combos (All 5)

**Epic:** 1 — OmniRoute Deployment and Wiring  
**Sprint:** 1  
**Status:** not-started

## User Story

*As a consumer, my existing route profile semantics (chat, code-completion, interactive, batch, triage) are preserved through OmniRoute combos so that no consumer changes are required at cutover.*

## Acceptance Criteria

- [ ] Five combos created: `combo-chat`, `combo-code`, `combo-interactive`, `combo-batch`, `combo-triage`
- [ ] Each combo configured per route-profiles manifest (routing strategy, timeout, retry policy, fallback)
- [ ] `combo-batch` uses strict routing with no fallback provider; returns 503 explicitly if llamacpp is unavailable
- [ ] `combo-triage` is priority with 60s timeout and 2048 max tokens
- [ ] All five combos return valid OpenAI-shaped responses for representative test requests via direct OmniRoute API call
- [ ] Route-profiles manifest committed at `docs/terminus/inference/terminus-inference-omniroute-gateway-migration/route-profiles.yaml`
- [ ] OmniRoute combo config exported to committed YAML (validates round-trip restore from Story 1.1 with combo state)
- [ ] llamacpp queue-depth test: send 2 concurrent requests when `parallel_slots=1`; verify OmniRoute queues second request correctly and both complete (no dropped requests, no timeout for reasonable 60s wait)

## Route Profile Reference

| Combo | Strategy | Timeout | Max Tokens | Retry | Fallback |
|-------|----------|---------|-----------|-------|---------|
| combo-chat | priority | 300s | 8192 | 2 | openai-fallback |
| combo-code | priority | 300s | 8192 | 2 | openai-fallback |
| combo-interactive | priority | 120s | 4096 | 1 | openai-fallback |
| combo-batch | strict | 0 (caller) | 16384 | 0 | none — explicit 503 |
| combo-triage | priority | 60s | 2048 | 1 | openai-fallback |

Full manifest: `docs/terminus/inference/terminus-inference-omniroute-gateway-migration/route-profiles.yaml` (produced by this story)

## Note on Story Width

This story is intentionally wide (5 combos in one story). The dev team may split into core combos (1.3a: chat/code/interactive) and specialized combos (1.3b: batch/triage) at sprint-start if estimated hours warrant it. This is a dev-phase decision.

## Dependencies

- OMNI-1.2 (llamacpp provider registered)

## Definition of Done

All five combos respond correctly; route-profiles.yaml committed; combo config exported and validated; queue-depth test completed.
