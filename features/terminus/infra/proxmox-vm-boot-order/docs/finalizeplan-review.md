---
feature: proxmox-vm-boot-order
doc_type: finalizeplan-review
status: approved
goal: "Add deterministic VM boot ordering to the Proxmox cluster so the system recovers gracefully after a hypervisor power cycle or host restart"
key_decisions:
  - tofu apply is non-destructive for startup{} and on_boot=true changes — no maintenance window required
  - Acceptance criterion: k3s starts gracefully and loads workers without human intervention after cold reboot
  - automation-host (Epic 1) must be applied before other module changes to avoid order=1 gap
open_questions: []
depends_on: []
blocks: []
updated_at: "2026-04-17"
---

# FinalizePlan Review — proxmox-vm-boot-order

**Verdict:** pass  
**Phase:** finalizeplan  
**Source:** phase-complete  

---

## Combined Planning Set Assessment

**Artifacts reviewed:**
- `architecture.md` — complete, all ADRs resolved, assumptions stated
- `techplan-adversarial-review.md` — pass-with-warnings; both warnings carried to epics as acceptance criteria

**Track:** `tech-change` — no BusinessPlan/PrePlan predecessors required. Architecture is the sole planning artifact and is sufficient.

---

## Cross-Artifact Coherence

✅ Architecture goal, scope, and implementation approach are internally consistent  
✅ All critical findings from TechPlan adversarial review are resolved  
✅ Open warnings (up_delay calibration, Patroni cold-start window) are acknowledged and tracked  
✅ `.todo` item `[BOOT-ORDER-ACCEPTANCE]` provides durable operator tracking  

---

## Findings Resolved in This Phase

### M-01 — `tofu apply` safety for live VMs
**Resolution:** Apply is non-destructive. `startup {}` and `on_boot = true` are API-level fields applied via Proxmox `PUT` — no VM replacement is triggered. Story acceptance criteria will include "verify `tofu plan` shows no replacement or destruction before apply."

### M-02 — Story sequencing: automation-host first
**Resolution:** Epic structure will sequence automation-host (new module + import block) as Epic 1. All subsequent module changes depend on it being applied first.

---

## Acceptance Criterion: Controlled Reboot Test

**Observable evidence:** After a controlled hypervisor reboot of `trantor`, the k3s cluster starts gracefully — control-plane nodes form quorum and workers join without human intervention.

**Operator validation steps (added to `.todo` item `[BOOT-ORDER-ACCEPTANCE]`):**
1. Cold reboot `trantor`
2. Observe autostart sequence — confirm VMs come up in configured order
3. Confirm k3s control-plane quorum is established before workers register
4. Confirm all workers join without manual `kubectl uncordon` or service restarts
5. Calibrate `up_delay` values based on observed guest-agent response times; commit updated values

---

## Governance Cross-Check

**Impacted services identified:**
- `terminus/infra` — directly implemented (postgres-cluster, postgres-backup, k3s-cluster modules)
- `terminus/platform` — k3s workloads depend on worker readiness; no code change required, but recovery behavior improves
- `terminus/inference` — inference gateway and openaiadapter run on k3s workers; improvement is passive

**No blocking governance dependencies.** Domain constitution (terminus) has no rules that gate this change.

---

## Implementation Sequencing

```
Epic 1: automation-host module (new — import VM 9001)
  └─ Story 1-1: Create automation-host module with import{} block and startup configuration
  └─ Story 1-2: Add automation-host to tofu/environments/automation/dev
  └─ Story 1-3: tofu plan review + apply (non-destructive validation)

Epic 2: postgres-cluster and postgres-backup startup configuration
  └─ Story 2-1: Add startup{} + on_boot=true to postgres-cluster module (primary order=2, replica order=3)
  └─ Story 2-2: Add startup{} + on_boot=true to postgres-backup module (order=3)
  └─ Story 2-3: tofu plan review + apply for postgres/dev environment

Epic 3: k3s-cluster startup configuration
  └─ Story 3-1: Add startup_order locals and new variables to k3s-cluster module
  └─ Story 3-2: Add startup{} + on_boot=true to k3s-cluster resource
  └─ Story 3-3: Update k3s/dev environment variables (control_plane_startup_order_base=4, worker_startup_order_base=7)
  └─ Story 3-4: tofu plan review + apply for k3s/dev environment
```

---

## Plan Branch Status

Artifacts staged on `proxmox-vm-boot-order-plan`:
- ✅ `docs/terminus/infra/proxmox-vm-boot-order/architecture.md`
- ✅ `docs/terminus/infra/proxmox-vm-boot-order/techplan-adversarial-review.md`
- ✅ `docs/terminus/infra/proxmox-vm-boot-order/finalizeplan-review.md` (this file)

**Ready for:** Plan PR from `proxmox-vm-boot-order-plan` → `proxmox-vm-boot-order`
