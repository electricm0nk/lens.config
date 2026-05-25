---
feature: terminus-inference-omniroute-gateway-migration
story_id: OMNI-1.1
doc_type: story
status: not-started
title: OmniRoute K8s Deployment
depends_on: []
updated_at: "2026-05-25"
sprint: 1
target_repo: TargetProjects/terminus/infra/terminus.infra
---

# OMNI-1.1: OmniRoute K8s Deployment

**Epic:** 1 — OmniRoute Deployment and Wiring  
**Sprint:** 1  
**Status:** not-started

## User Story

*As a platform operator, I can access a self-hosted OmniRoute instance running inside the inference namespace so that inference traffic stays on-premises.*

## Acceptance Criteria

- [ ] OmniRoute pod runs in `inference` namespace with resource limits (cpu: 1000m, memory: 1Gi)
- [ ] PVC `omniroute-data-pvc` (1Gi, ReadWriteOnce) is provisioned and mounted at `/app/data`
- [ ] OmniRoute responds to `GET /health` from inside the cluster
- [ ] Image tag is pinned to a specific semver release (not `:latest`)
- [ ] K8s NetworkPolicy: allow llamacpp egress, allow api.openai.com:443, deny all other external egress
- [ ] OmniRoute config exported from data volume to committed YAML; round-trip restore validated (create config → export → delete volume → import → verify config matches)
- [ ] OmniRoute readiness probe configured to delay Ready until `/health` returns 200 (not just container start)

## Open Questions (must resolve before story closes)

- What OmniRoute semver version to pin? (check `docker.io/diegosouzapw/omniroute` available tags; select latest stable)
- Is `gateway-vault-token` K8s secret the consumer-facing token or a service-account token? (context for Spike S-1.0)

## Implementation Notes

- Helm chart target: `terminus.infra` chart in `TargetProjects/terminus/infra/terminus.infra`
- Namespace: `inference` (existing)
- OmniRoute API port: 20128
- Do NOT use `cloud.omniroute.online` — self-hosted only (Org-4)
- Architecture reference: `docs/terminus/inference/terminus-inference-omniroute-gateway-migration/architecture.md`

## Dependencies

- None (Sprint 1 start story)

## Definition of Done

All ACs checked, open questions answered and committed, OmniRoute responds to `GET /health` from a test pod inside `inference` namespace.
