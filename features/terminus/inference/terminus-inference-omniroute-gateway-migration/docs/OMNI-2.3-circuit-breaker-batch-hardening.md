---
feature: terminus-inference-omniroute-gateway-migration
story_id: OMNI-2.3
doc_type: story
status: not-started
title: Circuit Breaker + Batch Hard-Fail Validation
depends_on: [OMNI-1.3]
updated_at: "2026-05-25"
sprint: 2
target_repo: TargetProjects/terminus/infra/terminus.infra
---

# OMNI-2.3: Circuit Breaker + Batch Hard-Fail Validation

**Epic:** 2 — Auth Bridge and Hardening  
**Sprint:** 2  
**Status:** not-started

## User Story

*As an operator, the inference platform degrades gracefully when llamacpp is unavailable so that consumers receive explicit errors rather than silent misbehaviour.*

## Acceptance Criteria (Circuit Breaker)

- [ ] OmniRoute circuit breaker opens within the configured failure threshold
- [ ] BDD scenario: `Given llamacpp is down, When a chat request is sent via combo-chat, Then the request fails over to openai-fallback within circuit breaker timeout`
- [ ] Circuit breaker recovers (closes) within 30 seconds of llamacpp returning to healthy
- [ ] Alert `LlamaCppProviderDown` (or equivalent OmniRoute health alert) fires during downtime

## Acceptance Criteria (Batch Hard-Fail)

- [ ] BDD scenario: `Given llamacpp is down, When a batch request is sent via combo-batch, Then a 503 response is returned with {"error": "provider unavailable"}`
- [ ] Batch requests do NOT silently fall over to OpenAI under any circumstance
- [ ] This is validated by: (a) combo-batch strict=true config, and (b) an explicit integration test simulating llamacpp downtime

## Open Question

- Can the `combo-triage` route guarantee `json_object` response format from OpenAI fallback? Test during this story. If OpenAI does not guarantee structured output for the triage use case, disable triage fallback and document the decision.

## Dependencies

- OMNI-1.3 (all five combos configured)
