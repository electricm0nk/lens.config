---
feature: terminus-infra-gpu-passthrough
doc_type: implementation-readiness
status: draft
goal: "Readiness assessment before implementation begins"
depends_on:
  - architecture.md
  - finalizeplan-review.md
  - epics.md
  - stories.md
updated_at: "2026-04-25T00:00:00Z"
---

# Implementation Readiness Report — terminus-infra-gpu-passthrough

Feature: terminus-infra-gpu-passthrough  
Track: tech-change  
Domain: terminus/infra  
Assessment Date: 2026-04-25

## Readiness Assessment

**Overall Status: CONDITIONAL READY**

The feature is ready for sprint planning and story-file creation. Implementation of Epic 1 and Epic 2 is blocked on resolution of five open questions (OQ-1 through OQ-BOOTSTRAP). Epic 3 is blocked on Epic 2 completion. No fatal architectural blockers remain. All findings from the TechPlan and FinalizePlan reviews are captured as stories.

### Readiness by Area

| Area | Status | Notes |
|------|--------|-------|
| Architecture | ✅ Ready | k3s GPU worker model is coherent; no open design contradictions |
| OpenTofu patterns | ✅ Ready | GPU worker module follows existing terminus.infra patterns |
| Inference gateway | ✅ Ready | Gateway and provider-routing are live; routing config update is bounded |
| Host prerequisites | ⚠️ Conditional | OQ-1 (node + PCI ID) must be resolved before Epic 1 starts |
| Worker sizing | ⚠️ Conditional | OQ-2 (static IP/FQDN), OQ-MAC, OQ-BOOTSTRAP must be resolved before Epic 2 |
| IOMMU gate | ⚠️ Conditional | Story 1.1 (IOMMU evidence artifact) is a hard gate for Epic 2 |
| Security controls | ⚠️ Conditional | NetworkPolicy (Story 3.3) is required before launch, not optional |
| Telemetry | ⚠️ Conditional | Minimum GPU telemetry (Story 3.4) required as launch gate |

## Risk Assessment

### HIGH Risks

**R-1 — IOMMU group includes critical host devices**
- Probability: Low-Medium (depends on slot assignment)
- Impact: HIGH — blocks Epic 2 entirely until resolved by hardware remap or ACS override approval
- Mitigation: Story 1.1 captures IOMMU group output early; resolution path required before Epic 2 begins
- Residual: If IOMMU group is unsafe, architecture is still valid but hardware changes are needed first

**R-2 — NVIDIA driver/k3s runtime compatibility on target kernel**
- Probability: Medium (k3s container runtime + NVIDIA toolkit compatibility matrix is version-sensitive)
- Impact: HIGH — blocks Epic 2 Story 2.3 until compatible versions identified
- Mitigation: OQ-BOOTSTRAP resolution pins specific versions before provisioning; test pod validation is a hard gate
- Residual: Version incompatibility requires driver/toolkit update cycle

### MEDIUM Risks

**R-3 — Proxmox feature dependency (terminus-infra-proxmox) diverges**
- Probability: Low (proxmox feature is in preplan)
- Impact: MEDIUM — if passthrough strategy changes in proxmox feature, Epic 1 work may need revision
- Mitigation: Flag sequence dependency; align when proxmox exits preplan; Epic 1 is self-contained enough to proceed

**R-4 — k3s worker provisioning pattern changes (terminus-infra-k3s)**
- Probability: Low (k3s feature is in preplan)
- Impact: MEDIUM — GPU worker module may need to follow updated provisioning conventions
- Mitigation: Track when k3s feature exits preplan; review module for alignment before Epic 2 Story 2.2 is marked done

**R-5 — GPU worker VM memory insufficient for Ollama model set**
- Probability: Medium (depends on model selection vs OQ-3)
- Impact: MEDIUM — pod OOMkill cycle if model exceeds VM memory minus system overhead
- Mitigation: F-3 worker sizing guidance in module README; OQ-3 resolution defines memory budget before provisioning

### LOW Risks

**R-6 — Single RTX 3090 becomes a single point of failure for local inference**
- Probability: N/A (by design — no MIG, no redundancy in scope)
- Impact: LOW — gateway fallback handles local provider down state
- Mitigation: Story 3.2 validates fallback path; no architectural change needed

**R-7 — NetworkPolicy inadvertently blocks gateway health checks**
- Probability: Low
- Impact: LOW — Ollama goes unhealthy from gateway perspective; fallback activates
- Mitigation: Story 3.3 includes explicit verification that gateway can reach Ollama after policy applied

## Pre-Implementation Checklist

The following must be confirmed before Epic 1 begins:

- [ ] OQ-1: Proxmox node name + RTX 3090 Turbo PCI address confirmed
- [ ] target_repos field in feature.yaml updated to include terminus.infra

The following must be confirmed before Epic 2 begins:

- [ ] Story 1.1 complete — IOMMU group output committed and IOMMU isolation confirmed safe
- [ ] Story 1.2 complete — RTX 3090 bound to vfio-pci; host reboot validated
- [ ] OQ-2: gpu-worker-01 static IP and FQDN assigned
- [ ] OQ-MAC: MAC address source of truth declared
- [ ] OQ-BOOTSTRAP: Canonical NVIDIA bootstrap method selected with version pins

The following must be confirmed before Epic 3 begins:

- [ ] Story 2.4 complete — Ollama running on GPU worker, nvidia.com/gpu satisfied
- [ ] OQ-3: Concurrency targets and model memory budget defined

## Launch Gate Summary

Feature is considered implementation-ready when ALL of the following pass:

1. Story 1.2 — VFIO confirmed at boot
2. Story 2.3 — NVIDIA device plugin shows allocatable GPU
3. Story 2.4 — Ollama pod running with GPU claim on GPU worker
4. Story 3.1 — Ollama provider registered in gateway
5. Story 3.2 — Healthy-path AND fallback-path routing validated
6. Story 3.3 — NetworkPolicy applied and verified
7. Story 3.4 — Minimum GPU telemetry visible

## Governance Dependency Notes

- **terminus-infra-proxmox** (preplan): Epic 1 can proceed independently; if proxmox passthrough strategy changes, revisit Epic 1 scope
- **terminus-infra-k3s** (preplan): GPU worker module follows current k3s patterns; if k3s feature changes conventions, update gpu-worker module before Epic 2 is closed
- **terminus-inference-gateway** (live): No planning-phase risk; routing config update is implementation-bounded
- **terminus-inference-provider-routing** (live): Stable API; routing YAML config extension is low-risk
