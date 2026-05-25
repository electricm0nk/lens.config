---
feature: terminus-inference-omniroute-gateway-migration
story_id: OMNI-3.3
doc_type: story
status: not-started
title: Decommission terminus-inference-gateway
depends_on: [OMNI-3.2]
updated_at: "2026-05-25"
sprint: 3
target_repo: TargetProjects/terminus/infra/terminus.infra
---

# OMNI-3.3: Decommission terminus-inference-gateway

**Epic:** 3 — Cutover and Decommission  
**Sprint:** 3  
**Status:** not-started

## Pre-Condition Gate

**This story may not begin until Story 3.2 cutover is validated AND the 24h parallel window has elapsed.**

Verify that the 24h parallel window has passed and all named consumers (Watchdog, Kaylee, Hermes) have confirmed successful operation through OmniRoute before proceeding with deletion.

## User Story

*As a platform operator, the legacy terminus-inference-gateway service is removed from the inference namespace so that maintenance surface is reduced.*

## Acceptance Criteria

- [ ] 24h parallel window has elapsed since Story 3.2 cutover
- [ ] All named consumers (Watchdog, Kaylee, Hermes) confirmed successful operation through OmniRoute
- [ ] `terminus-inference-gateway` deployment and service removed from `terminus.infra` Helm chart
- [ ] Helm release synced and ArgoCD reports Healthy
- [ ] Deployment, Service, and ingress-backend entries for `terminus-inference-gateway` removed
- [ ] `terminus-inference-gateway` source repo archived or marked deprecated (via GitHub archive or README notice)
- [ ] No CI pipeline triggers for `terminus-inference-gateway` after decommission

## Implementation Notes

- Remove from Helm values: `terminusInferenceGateway.enabled: false` (or delete section)
- Confirm ArgoCD sync before closing story
- Archive note: `terminus-inference-gateway` repo should include README: "Archived: Replaced by OmniRoute — see terminus-inference-omniroute-gateway-migration"

## Dependencies

- OMNI-3.2 (cutover validated; 24h window elapsed)
