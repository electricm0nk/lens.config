# Implementation Readiness — OmniRoute Gateway Migration

**Feature:** terminus-inference-omniroute-gateway-migration  
**Domain/Service:** terminus / inference  
**Track:** express  
**Assessment Date:** 2026-05-25

---

## Verdict: READY TO DEV (with gated pre-conditions)

The feature is implementation-ready for Sprint 1 to begin. Two pre-conditions must be satisfied before Sprint 2 begins.

---

## Readiness Checklist

### Planning Artifacts (Required — Express Track)

| Artifact | Status | Notes |
|---------|--------|-------|
| business-plan.md | PRESENT | Committed, reviewed |
| tech-plan.md | PRESENT | Committed, reviewed |
| sprint-plan.md | PRESENT | Updated with ADV fixes applied |
| expressplan-adversarial-review.md | PRESENT | PASS-WITH-WARNINGS verdict |
| finalizeplan-review.md | PRESENT | PASS-WITH-WARNINGS; all hard gates satisfied |
| architecture.md | PRESENT (stub) | FinalizePlan stub satisfies Org-3; finalized in Story 3.1 |

### Constitution Hard Gates

| Gate | Status | Notes |
|------|--------|-------|
| Org-3: Architecture documentation | PASS | architecture.md stub present |
| Org-4: No data exfiltration | PASS | Self-hosted OmniRoute; NetworkPolicy in design |
| Org-7: TDD Red-Green | CONDITIONAL | Required only if auth proxy path chosen in Spike S-1.0; Story 2.1 AC includes TDD requirement |
| Org-8: BDD scenarios | PASS | Stories 2.3 and 3.2 include BDD scenarios |
| Org-9: Security — no credential commits | PASS | All credentials Vault-backed; architecture.md documents paths |
| Org-16: External tool .todo | PARTIAL (gate) | .todo entry exists; must reach `approved` status before Sprint 2 begins (Chore C-1) |

### Story Completeness

| Story ID | Title | ACs Defined | Target Repo |
|----------|-------|-------------|-------------|
| OMNI-1.1 | OmniRoute K8s Deployment | Yes | terminus.infra |
| OMNI-S1.0 | Spike: Auth Bridge Path | Yes (timebox, output, gate) | docs only |
| OMNI-1.2 | llamacpp Provider Registration | Yes | terminus.infra |
| OMNI-1.3 | Route Profile Combos (All 5) | Yes (incl. queue-depth test) | terminus.infra |
| OMNI-C1 | Chore: Article 16 Approval | Yes | docs only |
| OMNI-C2 | Chore: Version Pin Automation | Yes | terminus.infra |
| OMNI-2.1 | Vault JWT Auth Bridge | Yes (both implementation paths) | terminus.infra |
| OMNI-2.2 | Contract Test Suite Port | Yes | terminus-inference-gateway |
| OMNI-2.3 | Circuit Breaker + Batch Hard-Fail | Yes (BDD scenarios) | terminus.infra |
| OMNI-3.1 | Architecture Documentation (Final) | Yes | docs only |
| OMNI-3.2 | Ingress Cutover + Parallel Window | Yes (consumer verification gate) | terminus.infra |
| OMNI-3.3 | Decommission terminus-inference-gateway | Yes | terminus.infra |

---

## Pre-Conditions Before Sprint 2 May Begin

1. **Chore C-1 complete:** `docs/terminus/.todo/2026-05-25-omniroute-external-tool-deviation.md` status updated to `approved`, reviewed by Todd Hintzmann, committed to control repo.
2. **Spike S-1.0 complete:** `auth-bridge-decision.md` committed. Auth bridge implementation path (Nginx or Go proxy) confirmed. Story 2.1 may not begin until Spike S-1.0 is complete.

---

## Open Technical Questions (Story-Level Deferrals)

| Question | Assigned To | Decision Gate |
|----------|------------|--------------|
| OmniRoute semver version pin | Story 1.1 open question | Before Story 1.1 closes |
| `gateway-vault-token` secret type (consumer-facing or service-account?) | Spike S-1.0 investigation | Spike S-1.0 output |
| Auth bridge implementation path (Nginx or Go proxy) | Spike S-1.0 | Story 2.1 gate |
| OpenAI fallback — can triage combo guarantee `json_object` format? | Story 2.3 contract test | Story 2.3 AC |

---

## Risk Summary

| Risk | Severity | Mitigation |
|------|----------|-----------|
| Auth bridge implementation unknown | Medium | Spike S-1.0 timebox in Sprint 1 |
| llamacpp `parallel_slots=1` queue behavior under OmniRoute routing | Medium | Story 1.3 concurrent test AC |
| OmniRoute config round-trip reliability | Low | Story 1.1 round-trip restore AC |
| Triage combo OpenAI json_object guarantee | Low | Story 2.3 test; disable fallback if unguaranteed |
