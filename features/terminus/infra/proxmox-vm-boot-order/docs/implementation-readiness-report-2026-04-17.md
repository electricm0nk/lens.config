---
stepsCompleted: [1, 2, 3, 4, 5, 6]
workflowType: implementation-readiness
project_name: proxmox-vm-boot-order
user_name: CrisWeber
date: "2026-04-17"
includedFiles:
  architecture:
    - docs/terminus/infra/proxmox-vm-boot-order/architecture.md
  reviews:
    - docs/terminus/infra/proxmox-vm-boot-order/techplan-adversarial-review.md
    - docs/terminus/infra/proxmox-vm-boot-order/finalizeplan-review.md
  epics:
    - docs/terminus/infra/proxmox-vm-boot-order/epics.md
missingDocuments:
  - prd (N/A — tech-change track; architecture is authoritative planning artifact)
  - ux-design (N/A — no UI component)
status: complete
readinessStatus: ready
assessor: GitHub Copilot
track: tech-change
feature: proxmox-vm-boot-order
---

# Implementation Readiness Report — proxmox-vm-boot-order

**Date:** 2026-04-17
**Feature:** `proxmox-vm-boot-order`
**Track:** tech-change
**Domain/Service:** terminus / infra

---

## Document Discovery

### Track Adjustment

This feature uses the `tech-change` track. No PRD or UX Design document is required or expected. The architecture document (`architecture.md`) is the sole authoritative planning artifact for this track.

### Artifacts Found

| Artifact | Path | Status |
|----------|------|--------|
| Architecture | `docs/terminus/infra/proxmox-vm-boot-order/architecture.md` | ✅ Complete |
| TechPlan adversarial review | `docs/terminus/infra/proxmox-vm-boot-order/techplan-adversarial-review.md` | ✅ Pass-with-warnings |
| FinalizePlan review | `docs/terminus/infra/proxmox-vm-boot-order/finalizeplan-review.md` | ✅ Pass |
| Epics and stories | `docs/terminus/infra/proxmox-vm-boot-order/epics.md` | ✅ Complete |
| Operator todo | `TargetProjects/terminus/infra/terminus.infra/.todo` | ✅ Created |

---

## Architecture Analysis

**Architecture document is complete.** Against the standard checklist:

| Criterion | Status | Notes |
|-----------|--------|-------|
| Problem statement present and clear | ✅ | Non-deterministic recovery sequence, race conditions documented |
| Solution approach justified | ✅ | `startup {}` blocks in `bpg/proxmox`, compared to alternatives |
| All ADRs resolved | ✅ | ADR-001 (sole mechanism), ADR-002 (import block), ADR-003 (concurrent tier) |
| Open questions captured | ✅ | `up_delay` calibration carried as open question in frontmatter |
| Assumptions declared | ✅ | Single node, guest agent present, recovery SLO not bounded |
| Out-of-scope items listed | ✅ | HA fencing, app-layer health checks, shutdown optimization |
| Specific boot order values provided | ✅ | Full order/up_delay table for all 12 VMs |
| Code examples for all module changes | ✅ | HCL snippets for all 4 affected modules |
| Acceptance criterion established | ✅ | k3s loads workers without human intervention post-reboot |

---

## FR Coverage Validation

All 15 functional requirements are covered by the epics:

| FR | Covered By | Status |
|----|-----------|--------|
| FR-01 (VM 9001 startup block) | Story 1.1 | ✅ |
| FR-02 (VM 9001 on_boot) | Story 1.1 | ✅ |
| FR-03 (VM 9001 import block) | Story 1.1 | ✅ |
| FR-04 (automation/dev environment) | Story 1.2 | ✅ |
| FR-05 (postgres-primary startup) | Story 2.1 | ✅ |
| FR-06 (postgres-primary on_boot) | Story 2.1 | ✅ |
| FR-07 (postgres-replica startup) | Story 2.1 | ✅ |
| FR-08 (postgres-replica on_boot) | Story 2.1 | ✅ |
| FR-09 (postgres-backup startup) | Story 2.2 | ✅ |
| FR-10 (postgres-backup on_boot) | Story 2.2 | ✅ |
| FR-11 (k3s startup_order locals) | Story 3.1 | ✅ |
| FR-12 (k3s new variables) | Story 3.1 | ✅ |
| FR-13 (k3s startup block on resource) | Story 3.2 | ✅ |
| FR-14 (k3s on_boot on resource) | Story 3.2 | ✅ |
| FR-15 (k3s/dev env vars) | Story 3.3 | ✅ |

**NFR and TA coverage:**

| Item | Covered By | Status |
|------|-----------|--------|
| NFR-01 (no replacements in plan) | Stories 1.3, 2.3, 3.4 AC | ✅ |
| NFR-02 (VCS only) | ADR-001, all stories | ✅ |
| NFR-03 (idempotent) | Stories 1.3, 2.3, 3.4 AC | ✅ |
| NFR-04 (import block retained) | Story 1.1 AC | ✅ |
| TA-01 (startup blocks sole mechanism) | Architecture ADR-001 | ✅ |
| TA-02 (import block for VM 9001) | Story 1.1, ADR-002 | ✅ |
| TA-03 (concurrent tier 3) | Stories 2.1 + 2.2 (same order=3) | ✅ |
| TA-04 (guest agent present) | Architecture Assumption 2 | ✅ |
| TA-05 (Epic 1 first) | Epic sequencing in finalizeplan-review | ✅ |
| TA-06 (acceptance test todo) | Story 3.4 AC + `.todo` | ✅ |

**Coverage: 100% — no orphaned requirements.**

---

## UX Alignment

No UI component. N/A for this track.

---

## Epic Quality Review

### Epic Value Assessment (tech-change track adaptation)

The standard requires epics to be "user-value-focused, not technical-layer-focused." For infrastructure tech-change features, the "user" is the platform operator. Assessment:

| Epic | User Value | Independent | Verdict |
|------|-----------|-------------|---------|
| Epic 1: Automation-Host Module | Operator can apply, import VM 9001 into state, and confirm tier-1 boot config | ✅ Standalone (no other epics needed to apply) | ✅ PASS |
| Epic 2: Postgres Startup Config | Operator can apply postgres boot ordering without touching k3s or automation-host afterward | ✅ Builds on Epic 1's presence in boot sequence; can be applied independently | ✅ PASS |
| Epic 3: k3s-Cluster Startup Config | Full k3s cluster recovery without intervention — this is the primary end-user acceptance criterion | ✅ Can be applied standalone; complete value delivered by Story 3.4 AC | ✅ PASS |

**Note on epic ordering:** Epic 1 (automation-host) must be applied before Epic 2 and 3 for the boot sequence to have no gaps at order=1. This is an **apply-time dependency**, not a code dependency — all three epics can be developed and reviewed independently, but should be applied in sequence. This is documented in `finalizeplan-review.md` and the `.todo` item.

### Story Dependency Audit

No forward dependencies detected. All stories within each epic are independently executable in sequence:

- Story 1.1 has no predecessor → creates module
- Story 1.2 depends on 1.1 (module exists) → creates environment
- Story 1.3 depends on 1.2 (`tofu init` run) → plan + apply
- Story 2.1 has no predecessor → modifies existing module
- Story 2.2 has no predecessor (different module) → modifies existing module
- Story 2.3 depends on 2.1 + 2.2 (changes exist) → plan + apply
- Story 3.1 has no predecessor → extends existing module locals/vars
- Story 3.2 depends on 3.1 (`startup_order` in locals) → adds resource block
- Story 3.3 has no predecessor (env layer change) → passes vars
- Story 3.4 depends on 3.1 + 3.2 + 3.3 (all module changes exist) → plan + apply

**No story requires a future story to function.** ✅

### Story Sizing

All stories are scoped to a single agent working session:
- Module file creation / modification (< 100 lines HCL each)
- `tofu validate` check
- `tofu plan` + `tofu apply` run

No story is oversized. ✅

### Acceptance Criteria Quality

All stories have:
- Given/When/Then format ✅
- Specific file paths ✅
- Specific HCL field references ✅
- Idempotency check AC (Stories 1.3, 2.3, 3.4) ✅
- No-replacement check AC (Stories 1.3, 2.3, 3.4) ✅

---

## Summary and Recommendations

### Overall Readiness Status

**READY**

### Critical Issues Requiring Immediate Action

None. All blocking items were resolved during the TechPlan and FinalizePlan phases.

### Observations Carried Forward (non-blocking)

1. **`up_delay` calibration:** Architecture Assumption 4 and `.todo [BOOT-ORDER-ACCEPTANCE]` carry this forward. The values are conservative initial estimates — they will be calibrated during acceptance testing after Story 3.4 apply. This is tracked and expected.

2. **automation-host VM configuration parity:** Story 1.1 requires the `proxmox_virtual_environment_vm.automation_host` resource declaration to match the existing VM 9001 configuration (CPU, memory, disk, network). The dev agent implementing Story 1.1 must inspect VM 9001 via the Proxmox API or GUI before writing the resource block. Acceptance criterion specifies `tofu plan` must show no replacement — this is the safety gate.

3. **import block idempotency:** NFR-04 requires the `import {}` block be retained in the module after first apply. This is explicitly captured in Story 1.1 AC. No action needed — just awareness.

### Recommended Next Steps

1. ✅ Run sprint planning to generate the sprint YAML
2. ✅ Create individual story files for all 10 stories
3. ✅ Create final PR: `proxmox-vm-boot-order` → `main`
4. ✅ Update governance `feature.yaml` and `feature-index.yaml` to `finalizeplan-complete`

**This feature is ready for sprint assignment and implementation.**
