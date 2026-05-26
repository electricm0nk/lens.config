# Business Plan — OmniRoute Upgrade to 3.8.3

**Feature**: terminus-inference-omniroute-upgrade-to-3-8-3  
**Track**: quickdev  
**Domain/Service**: terminus / inference  
**Date**: 2026-05-26

---

## Problem Statement

OmniRoute is deployed at version 3.2.8, which is significantly behind the current stable release (3.8.3 as of 2026-05-26). The gap spans ~24 patch/minor versions and includes a large backlog of security fixes, resilience improvements, and combo routing enhancements that directly benefit the terminus inference stack.

Key motivations:
- **Security**: v3.8.1 contains critical security fixes — `execSync` → `spawnSync` shell injection prevention, CSP hardening, and auth gates added to previously unauthenticated API endpoints (`/api/usage/budget/bulk`, `/api/resilience/reset`).
- **Reliability**: Circuit breaker improvements (DEGRADED state, `lastFailureKind` persistence, `exhaustedProviders` reset on retry) and a Global System Prompt persistence bug fix that was silently resetting the seeded system prompt on every restart.
- **Resilience**: Credential health check + adaptive circuit breaker v2, `falloverBeforeRetry` strategy, and API key health tracking with auto-rotation improve combo routing stability.
- **Observability**: WebSocket live monitoring daemon (port 20129) for real-time dashboard logs.
- **Maintainability**: Running a 6-month-old version increases the migration surface if left longer.

---

## Scope

### In scope
- Bump the OmniRoute Docker image tag from `3.2.8` to `3.8.3` in `deployment.yaml`
- Update `app.kubernetes.io/version` labels in deployment metadata and pod template
- Add the new WebSocket port `20129` to `service.yaml` to enable in-cluster dashboard access (WebSocket monitoring sidecar)
- Validate that all five seeded combos (combo-chat, combo-code, combo-interactive, combo-triage, combo-batch) continue to function after upgrade
- Validate health endpoint (`GET /api/monitoring/health` on port 20128) and ArgoCD sync

### Out of scope
- Re-seeding providers or credentials (existing SQLite PVC data is preserved)
- UI configuration changes or new feature enablement (API key groups, relay proxies, etc.)
- Changes to Traefik ingress, ForwardAuth, or vault-auth-proxy
- Enabling the new WebSocket monitoring dashboard integration beyond the port assessment

---

## Success Criteria

1. OmniRoute pod starts healthy and passes readiness probe (`/api/monitoring/health` returns 200) at 3.8.3
2. All five seeded combos respond correctly via the `/v1/chat/completions` endpoint
3. ArgoCD shows the omniroute app as Synced and Healthy
4. No regressions in inference API traffic through the vault-auth-proxy → OmniRoute chain
5. DB migrations complete cleanly on startup (SQLite at `/app/data` on PVC)

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| SQLite migration failure across ~24 version gap | Low | High | PVC data preserved; rollback is image tag revert |
| `better-sqlite3` optionality in 3.8.1 breaks startup | Low | High | Docker image bundles native binaries; fallback chain handles it |
| Seeded combo data incompatible with schema changes | Low | Medium | Validate combos post-deploy; re-seed from configmap if needed |
| New WebSocket port 20129 causes noise in logs | Low | Low | Port is not exposed externally; assess service.yaml addition |

**Rollback plan**: Revert `deployment.yaml` image tag to `3.2.8` and push — ArgoCD restores prior state.
