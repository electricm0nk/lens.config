---
stepsCompleted: [step-01, step-02, step-03, step-04]
inputDocuments:
  - docs/terminus/infra/proxmox-vm-boot-order/architecture.md
  - docs/terminus/infra/proxmox-vm-boot-order/finalizeplan-review.md
  - docs/terminus/infra/proxmox-vm-boot-order/techplan-adversarial-review.md
feature: proxmox-vm-boot-order
track: tech-change
domain: terminus
service: infra
status: ready-for-dev
updated_at: "2026-04-17"
---

# proxmox-vm-boot-order — Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for the `proxmox-vm-boot-order` tech-change feature. The goal is to add deterministic VM boot ordering to the Proxmox cluster so the system recovers gracefully after a hypervisor power cycle or host restart.

All changes are non-destructive `tofu apply` operations — no maintenance window is required. The acceptance criterion is: **k3s starts gracefully and loads all workers without human intervention after a cold reboot of the `trantor` hypervisor.**

---

## Requirements Inventory

### Functional Requirements

| FR | Requirement |
|----|-------------|
| FR-01 | `startup {}` block with `order` and `up_delay` fields declared on automation-dev (VM 9001) with order=1, up_delay=120 |
| FR-02 | `on_boot = true` declared on automation-dev (VM 9001) |
| FR-03 | automation-dev (VM 9001) brought under OpenTofu management via `import {}` block in a new `automation-host` module |
| FR-04 | New environment root `tofu/environments/automation/dev/` composing the automation-host module |
| FR-05 | `startup {}` block declared on postgres-primary (VM 200) with order=2, up_delay=90 |
| FR-06 | `on_boot = true` declared on postgres-primary (VM 200) |
| FR-07 | `startup {}` block declared on postgres-replica (VM 201) with order=3, up_delay=60 |
| FR-08 | `on_boot = true` declared on postgres-replica (VM 201) |
| FR-09 | `startup {}` block declared on postgres-backup (VM 202) with order=3, up_delay=60 |
| FR-10 | `on_boot = true` declared on postgres-backup (VM 202) |
| FR-11 | `startup_order` field added to k3s-cluster module `all_nodes` locals map, derived from base+index |
| FR-12 | New variables added to k3s-cluster module: `control_plane_startup_order_base`, `worker_startup_order_base`, `startup_up_delay`, `startup_down_delay` |
| FR-13 | `startup {}` block declared on k3s nodes resource using `each.value.startup_order` |
| FR-14 | `on_boot = true` declared on k3s nodes resource |
| FR-15 | k3s/dev environment variables set: control_plane_startup_order_base=4, worker_startup_order_base=7, startup_up_delay=60, startup_down_delay=30 |

### Non-Functional Requirements

| NFR | Requirement |
|-----|-------------|
| NFR-01 | `tofu plan` must show no VM replacements or destructions before any `tofu apply` — all changes are in-place API field updates |
| NFR-02 | All startup ordering configuration declared in version control only — no `qm set`, Proxmox GUI, or `datacenter.cfg` changes |
| NFR-03 | Boot ordering idempotent: repeated `tofu apply` must produce no drift |
| NFR-04 | `import {}` block retained in automation-host module for state idempotency |

### Technical Architecture Requirements

| TA | Requirement |
|----|-------------|
| TA-01 | ADR-001: `startup {}` blocks are the sole mechanism — no alternative ordering mechanisms |
| TA-02 | ADR-002: `import {}` block for VM 9001 — on first apply absorbs existing unmanaged VM into state |
| TA-03 | ADR-003: postgres-replica and postgres-backup share order=3 for concurrent tier start |
| TA-04 | Guest agent (`agent { enabled = true }`) already present on all resources — `up_delay` fires against liveness signal |
| TA-05 | Epic 1 (automation-host) must be applied before Epic 2 and Epic 3 to fill the order=1 gap |
| TA-06 | Acceptance test `[BOOT-ORDER-ACCEPTANCE]` in `.todo` — controlled hypervisor reboot to observe and calibrate |

### FR Coverage Map

| Epic | Stories | FRs Covered |
|------|---------|-------------|
| Epic 1: Automation-Host Module | 1-1, 1-2, 1-3 | FR-01, FR-02, FR-03, FR-04, NFR-01, NFR-02, NFR-04, TA-01, TA-02 |
| Epic 2: Postgres Startup Config | 2-1, 2-2, 2-3 | FR-05, FR-06, FR-07, FR-08, FR-09, FR-10, NFR-01, NFR-02, NFR-03, TA-01, TA-03 |
| Epic 3: k3s-Cluster Startup Config | 3-1, 3-2, 3-3, 3-4 | FR-11, FR-12, FR-13, FR-14, FR-15, NFR-01, NFR-02, NFR-03, TA-01, TA-04, TA-06 |

---

## Epic List

### Epic 1: Automation-Host Module
Bring automation-dev (VM 9001) under OpenTofu management with boot order tier 1 configuration. Operators can apply the new module safely, importing the existing VM into state without disruption, and confirm that VM 9001 is the first VM to autostart on hypervisor reboot.
**FRs covered:** FR-01, FR-02, FR-03, FR-04

### Epic 2: Postgres Startup Configuration
Add deterministic boot ordering to the postgres-cluster and postgres-backup modules so postgres-primary starts second (order=2) and both postgres-replica and postgres-backup start concurrently third (order=3). Operators apply non-destructive changes to the postgres/dev environment and verify no VM replacements are triggered.
**FRs covered:** FR-05, FR-06, FR-07, FR-08, FR-09, FR-10

### Epic 3: k3s-Cluster Startup Configuration
Extend the k3s-cluster module to assign sequential `startup_order` values to each node (CP nodes 4–6, workers 7–11), add `on_boot = true`, and wire the environment variables. Operators apply non-destructive changes to k3s/dev and verify the cluster recovers without human intervention after a controlled hypervisor reboot.
**FRs covered:** FR-11, FR-12, FR-13, FR-14, FR-15

---

## Epic 1: Automation-Host Module

**Goal:** Bring VM 9001 (automation-dev) under OpenTofu management via a new `tofu/modules/automation-host` module and a new `tofu/environments/automation/dev` environment root. On first apply, the `import {}` block absorbs the existing VM into state. The module declares `on_boot = true` and `startup { order = 1, up_delay = 120 }`, making automation-dev the first VM to boot on any hypervisor restart.

**Operator outcome:** `tofu plan` in `automation/dev` shows import of VM 9001 and zero replacement or destruction. `tofu apply` succeeds — VM 9001 is now in OpenTofu state with boot ordering declared.

---

### Story 1.1: Create automation-host module with import block and startup config

As a platform engineer,
I want a new `tofu/modules/automation-host` OpenTofu module that declares VM 9001 with `import {}`, `on_boot = true`, and `startup { order = 1, up_delay = 120 }`,
So that automation-dev is brought under state management with deterministic boot tier 1 configuration in a version-controlled, reproducible way.

**Acceptance Criteria:**

**Given** the file `TargetProjects/terminus/infra/terminus.infra/tofu/modules/automation-host/main.tf` does not exist
**When** the module is created with the `import {}` block referencing VM 9001 and the `proxmox_virtual_environment_vm.automation_host` resource with `on_boot = true` and `startup { order = 1, up_delay = 120, down_delay = 60 }`
**Then** the file exists with correct HCL syntax (passes `tofu validate`)
**And** the `import {}` block uses `to = proxmox_virtual_environment_vm.automation_host` and `id = "trantor/9001"` (node/vmid format)
**And** the resource includes `agent { enabled = true, timeout = "15m" }` consistent with other modules

**Given** the module is created
**When** `variables.tf` is created alongside it
**Then** variables are declared for: `vm_name`, `vm_id` (default: 9001), `target_node` (default: "trantor"), `startup_up_delay` (default: 120), `startup_down_delay` (default: 60)

**Given** the module is created
**When** `terraform.tf` (providers block) is created alongside it
**Then** it declares the `bpg/proxmox` provider version constraint matching the pattern used in other modules in the repo

---

### Story 1.2: Add automation-host to tofu/environments/automation/dev

As a platform engineer,
I want a new `tofu/environments/automation/dev/` environment root that composes the `automation-host` module,
So that the automation-dev VM is managed within the standard environment layout and `tofu init` / `tofu plan` / `tofu apply` can be run without special pre-steps.

**Acceptance Criteria:**

**Given** `TargetProjects/terminus/infra/terminus.infra/tofu/environments/automation/dev/` does not exist
**When** the environment root is created with `main.tf`, `variables.tf`, and `terraform.tf`
**Then** `main.tf` calls the automation-host module using a relative path (`../../../modules/automation-host`)
**And** all required module variables are passed through from environment-level variables
**And** `terraform.tf` declares the `bpg/proxmox` provider with backend configuration matching the pattern of `tofu/environments/k3s/dev/terraform.tf` (state backend, provider version)

**Given** the environment root is created
**When** `tofu init` is run from `tofu/environments/automation/dev/`
**Then** it completes without error and `.terraform/` is populated with provider plugins

---

### Story 1.3: tofu plan review and apply for automation/dev

As a platform engineer,
I want to run `tofu plan` for the `automation/dev` environment and confirm no destructive changes are planned before applying,
So that VM 9001 is imported into state with boot ordering declared, with zero risk of VM recreation or data loss.

**Acceptance Criteria:**

**Given** Story 1.1 and Story 1.2 are complete and `tofu init` has been run in `automation/dev`
**When** `tofu plan` is executed
**Then** the plan output shows an `import` for `proxmox_virtual_environment_vm.automation_host` (or its module-qualified address)
**And** the plan shows zero resources to destroy and zero forced replacements (`-/+`)
**And** any in-place updates (`~`) are only for `on_boot`, `startup`, and any other fields that differ from current VM state

**Given** `tofu plan` output has been reviewed and confirmed non-destructive
**When** `tofu apply` is executed with `-auto-approve` or interactive approval
**Then** it completes without error
**And** `tofu state list` includes `module.automation_host.proxmox_virtual_environment_vm.automation_host` (or equivalent)
**And** a follow-up `tofu plan` shows no changes (idempotent)

---

## Epic 2: Postgres Startup Configuration

**Goal:** Add `on_boot = true` and `startup {}` blocks to the `postgres-cluster` module (primary at order=2/up_delay=90, replica at order=3/up_delay=60) and the `postgres-backup` module (backup at order=3/up_delay=60). Apply to the postgres/dev environment non-destructively.

**Operator outcome:** `tofu plan` for postgres/dev shows only in-place `~` updates to `on_boot` and `startup` fields — no VM replacements. After apply, postgres-primary comes up second in the boot sequence and replica+backup start concurrently in tier 3.

---

### Story 2.1: Add startup config to postgres-cluster module

As a platform engineer,
I want `on_boot = true` and `startup {}` blocks added to both `proxmox_virtual_environment_vm.postgres_primary` and `proxmox_virtual_environment_vm.postgres_replica` in `tofu/modules/postgres-cluster/main.tf`,
So that on hypervisor reboot, postgres-primary starts second (after automation-dev) and postgres-replica starts concurrently with postgres-backup in tier 3.

**Acceptance Criteria:**

**Given** `TargetProjects/terminus/infra/terminus.infra/tofu/modules/postgres-cluster/main.tf` exists
**When** the `postgres_primary` resource is updated
**Then** it includes `on_boot = true` and `startup { order = 2, up_delay = 90, down_delay = 60 }` in the resource block

**Given** the `postgres_replica` resource exists in the same file
**When** it is updated
**Then** it includes `on_boot = true` and `startup { order = 3, up_delay = 60, down_delay = 30 }` in the resource block

**Given** both resources are updated
**When** `tofu validate` is run from the module directory
**Then** it returns no errors

---

### Story 2.2: Add startup config to postgres-backup module

As a platform engineer,
I want `on_boot = true` and `startup { order = 3, up_delay = 60, down_delay = 30 }` added to `proxmox_virtual_environment_vm.postgres_backup` in `tofu/modules/postgres-backup/main.tf`,
So that postgres-backup starts concurrently with postgres-replica (same tier 3) on hypervisor reboot.

**Acceptance Criteria:**

**Given** `TargetProjects/terminus/infra/terminus.infra/tofu/modules/postgres-backup/main.tf` exists
**When** the `postgres_backup` resource is updated
**Then** it includes `on_boot = true` and `startup { order = 3, up_delay = 60, down_delay = 30 }` in the resource block

**Given** the resource is updated
**When** `tofu validate` is run from the module directory
**Then** it returns no errors

---

### Story 2.3: tofu plan review and apply for postgres/dev

As a platform engineer,
I want to run `tofu plan` for the `postgres/dev` environment and confirm only in-place updates before applying,
So that postgres-primary and replica/backup VMs have boot ordering declared in Proxmox state with zero risk of VM recreation.

**Acceptance Criteria:**

**Given** Stories 2.1 and 2.2 are complete
**When** `tofu plan` is executed from `TargetProjects/terminus/infra/terminus.infra/tofu/environments/postgres/dev/`
**Then** the plan shows only `~` (in-place update) changes — zero `-/+` (force-replace) or `-` (destroy) entries

**Given** the plan is reviewed and confirmed non-destructive
**When** `tofu apply` is executed
**Then** it completes without error

**Given** apply completes
**When** a follow-up `tofu plan` is run
**Then** it shows no changes (idempotent)

---

## Epic 3: k3s-Cluster Startup Configuration

**Goal:** Extend `tofu/modules/k3s-cluster` to add `startup_order` to the `all_nodes` locals map, introduce four new variables (`control_plane_startup_order_base`, `worker_startup_order_base`, `startup_up_delay`, `startup_down_delay`), add `on_boot = true` and `startup {}` to the node resource, and pass environment-specific values from `tofu/environments/k3s/dev/`. Apply non-destructively. The post-apply system satisfies the acceptance criterion: after a controlled hypervisor reboot of `trantor`, k3s loads workers without human intervention.

**Operator outcome:** k3s control-plane nodes start at orders 4–6; workers start at orders 7–11, each 60 seconds apart. The cluster recovers autonomously.

---

### Story 3.1: Add startup_order locals and new variables to k3s-cluster module

As a platform engineer,
I want the `k3s-cluster` module locals and variables extended to carry `startup_order` per node,
So that each k3s VM gets a distinct boot order value derived from its index without hardcoding order values per VM.

**Acceptance Criteria:**

**Given** `TargetProjects/terminus/infra/terminus.infra/tofu/modules/k3s-cluster/main.tf` has `control_plane_nodes` and `worker_nodes` locals
**When** the locals are updated
**Then** each `control_plane_nodes` entry includes `startup_order = var.control_plane_startup_order_base + index`
**And** each `worker_nodes` entry includes `startup_order = var.worker_startup_order_base + index`

**Given** `TargetProjects/terminus/infra/terminus.infra/tofu/modules/k3s-cluster/variables.tf` exists
**When** the variables file is updated
**Then** it declares four new variables:
  - `control_plane_startup_order_base` — type: number, default: 4, description: "Starting Proxmox autostart order value for control-plane nodes. Each CP gets base+index."
  - `worker_startup_order_base` — type: number, default: 7, description: "Starting Proxmox autostart order value for worker nodes. Each worker gets base+index."
  - `startup_up_delay` — type: number, default: 60, description: "Seconds to wait after QEMU agent responds before starting the next-order VM."
  - `startup_down_delay` — type: number, default: 30, description: "Seconds to wait before sending shutdown signal to next VM during Proxmox autostart shutdown."

**Given** locals and variables are updated
**When** `tofu validate` is run from the module directory
**Then** it returns no errors

---

### Story 3.2: Add startup block and on_boot to k3s node resource

As a platform engineer,
I want `on_boot = true` and a `startup {}` block using `each.value.startup_order` added to the `proxmox_virtual_environment_vm.nodes` resource in the k3s-cluster module,
So that each k3s VM is configured for autostart at its assigned sequential boot order.

**Acceptance Criteria:**

**Given** Story 3.1 is complete and `startup_order` is present in `local.all_nodes`
**When** the `proxmox_virtual_environment_vm.nodes` resource in `main.tf` is updated
**Then** it includes `on_boot = true` at the resource level
**And** it includes a `startup {}` block:
  ```hcl
  startup {
    order      = each.value.startup_order
    up_delay   = var.startup_up_delay
    down_delay = var.startup_down_delay
  }
  ```

**Given** the resource is updated
**When** `tofu validate` is run from the module directory
**Then** it returns no errors

---

### Story 3.3: Update k3s/dev environment with startup order variables

As a platform engineer,
I want the `tofu/environments/k3s/dev/` environment root to explicitly pass startup order variables to the k3s-cluster module,
So that CP nodes get orders 4–6 and workers get orders 7–11 with 60-second up_delay in the dev environment.

**Acceptance Criteria:**

**Given** `TargetProjects/terminus/infra/terminus.infra/tofu/environments/k3s/dev/main.tf` exists and calls the k3s-cluster module
**When** the module call is updated
**Then** it passes:
  - `control_plane_startup_order_base = 4`
  - `worker_startup_order_base        = 7`
  - `startup_up_delay                 = 60`
  - `startup_down_delay               = 30`

**Given** the environment is updated
**When** `tofu validate` is run from `tofu/environments/k3s/dev/`
**Then** it returns no errors

---

### Story 3.4: tofu plan review and apply for k3s/dev

As a platform engineer,
I want to run `tofu plan` for the `k3s/dev` environment and confirm only in-place updates before applying,
So that all k3s nodes receive boot ordering configuration with zero risk of VM recreation or kubelet disruption.

**Acceptance Criteria:**

**Given** Stories 3.1, 3.2, and 3.3 are complete
**When** `tofu plan` is executed from `TargetProjects/terminus/infra/terminus.infra/tofu/environments/k3s/dev/`
**Then** the plan shows only `~` (in-place update) changes — zero `-/+` or `-` entries
**And** the plan output shows `on_boot` and `startup` fields updating for each of the k3s nodes (5 CP + 5 workers = 10 VMs if 3 CP + 5 workers)

**Given** the plan is reviewed and confirmed non-destructive
**When** `tofu apply` is executed
**Then** it completes without error

**Given** apply completes
**When** a follow-up `tofu plan` is run
**Then** it shows no changes (idempotent)

**Given** all three epics are applied
**When** the operator performs a controlled reboot of the `trantor` hypervisor (per `.todo` item `[BOOT-ORDER-ACCEPTANCE]`)
**Then** VMs come up in the configured order (9001 → 200 → 201+202 concurrent → 300-302 sequential → 310-314 sequential)
**And** k3s control-plane quorum is established before workers register
**And** all workers join the cluster without manual `kubectl uncordon` or service restarts
**And** up_delay values are observed and calibrated; updated values committed if adjustments are needed
