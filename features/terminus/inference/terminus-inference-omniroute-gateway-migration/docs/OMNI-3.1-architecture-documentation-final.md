---
feature: terminus-inference-omniroute-gateway-migration
story_id: OMNI-3.1
doc_type: story
status: not-started
title: "Architecture Documentation — Full Finalization"
depends_on: [OMNI-S1.0]
updated_at: "2026-05-25"
sprint: 3
target_repo: docs only
---

# OMNI-3.1: Architecture Documentation — Full Finalization

**Epic:** 3 — Cutover and Decommission  
**Sprint:** 3  
**Status:** not-started

## User Story

*As a future maintainer, I can read the complete architecture documentation for the OmniRoute deployment so that all operational decisions (including the confirmed auth bridge implementation) are captured and auditable.*

## Context

An architecture stub was produced during FinalizePlan to satisfy Org-3 before dev begins. The stub marks the auth bridge section as "TBD — resolved in Spike S-1.0 / Story 3.1." This story finalizes that section once the auth bridge choice has been confirmed and implemented.

## Acceptance Criteria

- [ ] `docs/terminus/inference/terminus-inference-omniroute-gateway-migration/architecture.md` updated with confirmed auth bridge implementation + rationale
- [ ] Auth bridge section no longer marked TBD — contains confirmed implementation choice, configuration details, and rationale from `auth-bridge-decision.md`
- [ ] Covers: K8s deployment topology, route profile manifest reference, observability wiring, Article 16 deviation decision
- [ ] Reviewed and approved before cutover proceeds (Story 3.2 depends on this)

## Dependencies

- OMNI-S1.0 (auth bridge path confirmed — required to finalize the auth bridge section)

## Note

This story requires only documentation updates to the control repo docs path. No code changes are required.
