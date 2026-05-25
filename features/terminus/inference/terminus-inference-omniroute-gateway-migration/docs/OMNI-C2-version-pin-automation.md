---
feature: terminus-inference-omniroute-gateway-migration
story_id: OMNI-C2
doc_type: story
status: not-started
title: "Chore: OmniRoute Version Pin Automation"
depends_on: []
updated_at: "2026-05-25"
sprint: 1
target_repo: TargetProjects/terminus/infra/terminus.infra
type: chore
---

# OMNI-C2: Chore — OmniRoute Version Pin Automation

**Epic:** 1 — OmniRoute Deployment and Wiring  
**Sprint:** 1  
**Type:** Chore  
**Reviewer:** Todd Hintzmann

## Description

Add a Dependabot or Renovate rule in `terminus.infra` to watch the `diegosouzapw/omniroute` Docker image for new semver releases. Notification-only — do not auto-merge. This ensures version pin updates are evaluated consciously.

## Acceptance Criteria

- [ ] Dependabot/Renovate config updated in `terminus.infra` to watch `diegosouzapw/omniroute` Docker image
- [ ] Notification-only (no auto-merge policy)
- [ ] Committed to `terminus.infra` before Story 1.1 closes

## Dependencies

- None (run alongside Story 1.1 in Sprint 1)
