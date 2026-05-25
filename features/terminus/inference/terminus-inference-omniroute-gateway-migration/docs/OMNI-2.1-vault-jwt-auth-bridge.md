---
feature: terminus-inference-omniroute-gateway-migration
story_id: OMNI-2.1
doc_type: story
status: not-started
title: Vault JWT Auth Bridge
depends_on: [OMNI-1.1, OMNI-S1.0]
updated_at: "2026-05-25"
sprint: 2
target_repo: TargetProjects/terminus/infra/terminus.infra
---

# OMNI-2.1: Vault JWT Auth Bridge

**Epic:** 2 — Auth Bridge and Hardening  
**Sprint:** 2  
**Status:** not-started

## Pre-Condition Gate

**This story may not begin until Spike S-1.0 (`auth-bridge-decision.md`) is committed and reviewed.**

Check `docs/terminus/inference/terminus-inference-omniroute-gateway-migration/auth-bridge-decision.md` before starting. Implement per the confirmed path.

## User Story

*As an existing consumer, I can continue to use my Vault-issued JWT token to call the inference gateway endpoint so that I require no credential changes after the migration.*

## Acceptance Criteria

- [ ] Auth bridge validates incoming Vault JWT tokens using Vault's `/v1/auth/token/lookup-self` endpoint
- [ ] Valid JWT: request forwarded to OmniRoute with `Authorization: Bearer <omniroute-api-key>` header injected
- [ ] Invalid/expired JWT: bridge returns HTTP 401 with `{"error": "unauthorized"}` body; request does not reach OmniRoute
- [ ] OmniRoute API key is stored in Vault, injected as K8s secret; not committed to source control (Org-9)

**If Nginx annotation path (from Spike S-1.0):**
- [ ] Nginx `auth-url` annotation config committed to `terminus.infra` chart; no new service required

**If Go auth proxy path (from Spike S-1.0):**
- [ ] Proxy written in Go (Svc-7)
- [ ] TDD red-green discipline applied (Org-7): failing tests committed before implementation
- [ ] Unit test coverage ≥ 90% for JWT validation and header injection paths

## Implementation Notes

- Vault endpoint: `/v1/auth/token/lookup-self`
- OmniRoute API key: sourced from Vault, injected as K8s secret (key: `OMNIROUTE_API_KEY`)
- Do not store OmniRoute API key as plaintext in any committed file
- Architecture reference: `docs/terminus/inference/terminus-inference-omniroute-gateway-migration/architecture.md`

## Dependencies

- OMNI-1.1 (OmniRoute running)
- OMNI-S1.0 (auth bridge implementation path confirmed)
