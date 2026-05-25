---
feature: terminus-inference-omniroute-gateway-migration
story_id: OMNI-2.2
doc_type: story
status: not-started
title: Contract Test Suite Port to OmniRoute
depends_on: [OMNI-1.3, OMNI-2.1]
updated_at: "2026-05-25"
sprint: 2
target_repo: TargetProjects/terminus/inference/terminus-inference-gateway
---

# OMNI-2.2: Contract Test Suite Port to OmniRoute

**Epic:** 2 — Auth Bridge and Hardening  
**Sprint:** 2  
**Status:** not-started

## User Story

*As a developer, the existing contract test suite passes against the OmniRoute endpoint so that I have confidence the API contract is preserved before cutover.*

## Acceptance Criteria

- [ ] All existing contract tests in `terminus-inference-gateway` adapter suite pass when run against `http://omniroute.inference.svc.cluster.local:20128/v1`
- [ ] Test runner updated to target OmniRoute endpoint via environment variable (e.g., `INFERENCE_TARGET_URL`)
- [ ] No tests skipped or modified to pass — all existing assertions hold
- [ ] CI pipeline can execute tests against OmniRoute (not just terminus-inference-gateway)

## Implementation Notes

- Tests should be runnable against either endpoint via env var — do not hardcode OmniRoute URL
- The contract tests from `terminus-inference-gateway` test the OpenAI API shape; these should pass unchanged against OmniRoute
- If any tests fail: document the gap and create a follow-up issue (do not skip tests silently)

## Dependencies

- OMNI-1.3 (all five combos working via OmniRoute)
- OMNI-2.1 (auth bridge working — tests run through the full auth path)
