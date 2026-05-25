# Stories — OmniRoute Gateway Migration

**Feature:** terminus-inference-omniroute-gateway-migration  
**Domain/Service:** terminus / inference  
**Track:** express  
**Generated:** 2026-05-25

---

## Epic 1: OmniRoute Deployment and Wiring

| Story ID | Title | Sprint | Status | Dependencies |
|----------|-------|--------|--------|-------------|
| OMNI-1.1 | OmniRoute K8s Deployment | 1 | not-started | none |
| OMNI-S1.0 | Spike: Auth Bridge Implementation Path | 1 | not-started | none (parallel with 1.1) |
| OMNI-1.2 | llamacpp Provider Registration | 1 | not-started | OMNI-1.1 |
| OMNI-1.3 | Route Profile Combos (All 5) | 1 | not-started | OMNI-1.2 |
| OMNI-C1 | Chore: Article 16 .todo Approval | 1 | not-started | none |
| OMNI-C2 | Chore: OmniRoute Version Pin Automation | 1 | not-started | none |

---

## Epic 2: Auth Bridge and Hardening

| Story ID | Title | Sprint | Status | Dependencies |
|----------|-------|--------|--------|-------------|
| OMNI-2.1 | Vault JWT Auth Bridge | 2 | not-started | OMNI-1.1, OMNI-S1.0 |
| OMNI-2.2 | Contract Test Suite Port to OmniRoute | 2 | not-started | OMNI-1.3, OMNI-2.1 |
| OMNI-2.3 | Circuit Breaker + Batch Hard-Fail Validation | 2 | not-started | OMNI-1.3 |

---

## Epic 3: Cutover and Decommission

| Story ID | Title | Sprint | Status | Dependencies |
|----------|-------|--------|--------|-------------|
| OMNI-3.1 | Architecture Documentation — Full Finalization | 3 | not-started | OMNI-S1.0 resolved |
| OMNI-3.2 | Ingress Cutover + Parallel Window | 3 | not-started | OMNI-2.2, OMNI-2.3, OMNI-3.1 |
| OMNI-3.3 | Decommission terminus-inference-gateway | 3 | not-started | OMNI-3.2 (24h window elapsed) |

---

## Story Detail Index

See individual story files in `Stories/` directory:
- [OMNI-1.1](Stories/OMNI-1.1-omniroute-k8s-deployment.md)
- [OMNI-S1.0](Stories/OMNI-S1.0-auth-bridge-spike.md)
- [OMNI-1.2](Stories/OMNI-1.2-llamacpp-provider-registration.md)
- [OMNI-1.3](Stories/OMNI-1.3-route-profile-combos.md)
- [OMNI-C1](Stories/OMNI-C1-article16-todo-approval.md)
- [OMNI-C2](Stories/OMNI-C2-version-pin-automation.md)
- [OMNI-2.1](Stories/OMNI-2.1-vault-jwt-auth-bridge.md)
- [OMNI-2.2](Stories/OMNI-2.2-contract-test-suite-port.md)
- [OMNI-2.3](Stories/OMNI-2.3-circuit-breaker-batch-hardening.md)
- [OMNI-3.1](Stories/OMNI-3.1-architecture-documentation-final.md)
- [OMNI-3.2](Stories/OMNI-3.2-ingress-cutover.md)
- [OMNI-3.3](Stories/OMNI-3.3-decommission-gateway.md)
