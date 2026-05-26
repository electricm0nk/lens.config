# Sprint Plan — OmniRoute Upgrade to 3.8.3

**Feature**: terminus-inference-omniroute-upgrade-to-3-8-3  
**Track**: quickdev  
**Domain/Service**: terminus / inference  
**Date**: 2026-05-26

---

## Sprint Overview

**Sprint**: 1 of 1  
**Scope**: Bounded image bump upgrade with validation  
**Target**: OmniRoute 3.2.8 → 3.8.3 in inference namespace

---

## Stories

### OMNI-U1: Update OmniRoute image to 3.8.3 and validate deployment

**Type**: Implementation  
**Estimate**: S  

**Description**:  
Update the OmniRoute deployment in `terminus.infra` to use image `diegosouzapw/omniroute:3.8.3`. Update all version labels. Optionally add port 20129 to the ClusterIP service. Commit to the feature branch and verify ArgoCD sync.

**Acceptance Criteria**:
- `deployment.yaml` image updated to `diegosouzapw/omniroute:3.8.3`
- `app.kubernetes.io/version` labels updated to `"3.8.3"` in both metadata and pod template
- `readinessProbe.initialDelaySeconds` bumped to `60` for this upgrade deploy (revert after validation confirms acceptable startup time)
- ArgoCD `omniroute` app shows `Synced` and `Healthy`
- OmniRoute pod passes readiness probe (`/api/monitoring/health` returns 200)
- SQLite migrations complete without error (check pod logs)

**Files changed**:
- `platforms/k3s/k8s/omniroute/deployment.yaml`
- `platforms/k3s/k8s/omniroute/service.yaml` (add port 20129 for WebSocket monitoring sidecar)

---

### OMNI-U2: Validate combo routing and API correctness post-upgrade

**Type**: Validation  
**Estimate**: S  

**Description**:  
After OmniRoute 3.8.3 is live, verify that all seeded combos continue to route correctly through the vault-auth-proxy → OmniRoute auth chain. Confirm no regressions in the `/v1/chat/completions` API response shape.

**Acceptance Criteria**:
- combo-chat responds correctly
- combo-code responds correctly
- combo-interactive responds correctly
- combo-triage responds correctly
- combo-batch responds correctly (llamacpp backend)
- Health endpoint returns 200 with valid JSON
- No auth failures in vault-auth-proxy logs
- No 500-series errors in OmniRoute pod logs during combo test
- **GSP persistence check**: restart the OmniRoute pod after upgrade and confirm the seeded system prompt is preserved (validates 3.8.2 GSP persistence fix)
- **SQLite adapter check**: confirm startup logs show `better-sqlite3` native binary in use (not WASM fallback)

**Test method**: Direct `curl` via kubectl exec or via the OmniRoute dashboard using the internal API key.

---

### OMNI-U3: Author runbook for OmniRoute upgrade procedure

**Type**: Documentation  
**Estimate**: XS  

**Description**:  
Document the upgrade procedure so future version bumps can be executed with a standard checklist. Include the image bump pattern, migration notes, validation steps, and rollback procedure.

**Acceptance Criteria**:
- `runbook.md` created in the feature docs path
- Covers: pre-upgrade checklist, deployment steps, validation steps, rollback steps
- References the `better-sqlite3` compatibility note and WebSocket port 20129 consideration

---

## Sprint Checklist

- [ ] OMNI-U1: Image bump + deploy
- [ ] OMNI-U2: Combo validation
- [ ] OMNI-U3: Runbook authored
- [ ] Feature phase transitioned to `dev-complete`
- [ ] Governance updated

---

## Definition of Done

- OmniRoute 3.8.3 is live in the inference namespace and healthy
- All five combos validated
- Runbook authored
- No open P1/P2 issues from the upgrade
