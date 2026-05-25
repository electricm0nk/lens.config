---
feature: terminus-inference-omniroute-gateway-migration
story_id: OMNI-S1.0
doc_type: story
status: not-started
title: "Spike: Auth Bridge Implementation Path"
depends_on: []
updated_at: "2026-05-25"
sprint: 1
target_repo: docs only
type: spike
---

# OMNI-S1.0: Spike — Auth Bridge Implementation Path

**Epic:** 1 — OmniRoute Deployment and Wiring  
**Sprint:** 1 (timebox: 2 hours)  
**Status:** not-started  
**Type:** Spike

## User Story

*As a developer, I have confirmed whether Nginx ingress `auth-url` annotation can validate Vault JWT tokens and inject the OmniRoute API key so that the auth bridge implementation path is decided before Sprint 2 begins.*

## Spike Goal

Validate Nginx `auth-url` against Vault `/v1/auth/token/lookup-self`:

- **If accessible and returning 200/403 correctly:** Document Nginx annotation as confirmed implementation path.
- **If not accessible or insufficient (e.g., Vault endpoint requires service mesh, mutual TLS, or IP allowlist that Nginx ingress cannot satisfy):** Confirm Go auth proxy as implementation path.

## Timebox

**2 hours maximum.** Produce a decision document regardless of outcome. Do not continue investigation after the timebox expires.

## Output

Commit `docs/terminus/inference/terminus-inference-omniroute-gateway-migration/auth-bridge-decision.md` containing:

1. Implementation path chosen: `nginx-annotation` | `go-proxy`
2. Rationale (what was tested, what succeeded or failed)
3. Key config details for the chosen path (e.g., annotation keys for Nginx; package choices for Go proxy)
4. Answer to: Is `gateway-vault-token` K8s secret the consumer-facing Vault JWT token, or a service-account / intermediary token?

## Acceptance Criteria

- [ ] `auth-bridge-decision.md` committed to control repo before Sprint 2 begins
- [ ] Chosen path is one of: `nginx-annotation` or `go-proxy` (not TBD)
- [ ] `gateway-vault-token` secret type is identified and documented
- [ ] Story 2.1 implementation notes updated with confirmed path

## Gate

**Story 2.1 (Vault JWT Auth Bridge) may not begin until this spike output is committed and reviewed.**

## Investigation Steps

1. Check whether Nginx ingress-nginx supports `auth-url` annotation in the current cluster
2. Test whether Vault `/v1/auth/token/lookup-self` is accessible from the Nginx ingress pod's network
3. Test whether Vault returns 200 on valid token / 403 on invalid token in a way Nginx can interpret
4. Inspect `kubectl -n inference get secret gateway-vault-token -o yaml` — confirm what credential is stored and what its intended use is

## Dependencies

- None (can run in parallel with Story 1.1)
