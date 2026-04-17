# Story 2.1: Add startup config to postgres-cluster module

## Status: ready-for-dev

## Story

As a platform engineer,
I want `on_boot = true` and `startup {}` blocks added to both `proxmox_virtual_environment_vm.postgres_primary` and `proxmox_virtual_environment_vm.postgres_replica` in the `postgres-cluster` module,
So that on hypervisor reboot, postgres-primary starts second (order=2, tier 2) and postgres-replica starts concurrently with postgres-backup in tier 3 (order=3).

## Acceptance Criteria

- **Given** `TargetProjects/terminus/infra/terminus.infra/tofu/modules/postgres-cluster/main.tf` exists with `postgres_primary` and `postgres_replica` VM resources
- **When** the `postgres_primary` resource is updated
- **Then** it includes `on_boot = true` at the resource level
- **And** it includes `startup { order = 2, up_delay = 90, down_delay = 60 }` in the resource block

- **Given** the `postgres_replica` resource exists in the same file
- **When** it is updated
- **Then** it includes `on_boot = true` at the resource level
- **And** it includes `startup { order = 3, up_delay = 60, down_delay = 30 }` in the resource block

- **Given** both resources are updated
- **When** `tofu validate` is run from the `tofu/modules/postgres-cluster/` directory
- **Then** it returns no errors

## Tasks / Subtasks

- [ ] Task 1: Read current postgres-cluster module
  - [ ] Open `TargetProjects/terminus/infra/terminus.infra/tofu/modules/postgres-cluster/main.tf`
  - [ ] Locate `proxmox_virtual_environment_vm.postgres_primary` resource block
  - [ ] Locate `proxmox_virtual_environment_vm.postgres_replica` resource block
  - [ ] Note current structure (find the right insertion point for `on_boot` and `startup {}`)
- [ ] Task 2: Update `postgres_primary` resource
  - [ ] Add `on_boot = true` (top-level attribute on the resource block)
  - [ ] Add `startup { order = 2; up_delay = 90; down_delay = 60 }` block
- [ ] Task 3: Update `postgres_replica` resource
  - [ ] Add `on_boot = true`
  - [ ] Add `startup { order = 3; up_delay = 60; down_delay = 30 }` block
- [ ] Task 4: Run `tofu validate` from module directory — fix any syntax errors

## Dev Notes

**HCL block syntax for startup:**
```hcl
on_boot = true

startup {
  order      = 2
  up_delay   = 90
  down_delay = 60
}
```

**Concurrent tier 3 (ADR-003):** postgres-replica and postgres-backup both use `order = 3`. Proxmox starts both simultaneously within the tier. The `up_delay` applies per-VM — the tier does not advance until all order=3 VMs have completed their gate.

**Attribute placement:** `on_boot` and `startup {}` are top-level attributes/blocks of `proxmox_virtual_environment_vm`. Place them alongside `name`, `node_name`, `vm_id` etc. — not inside a nested block.

**down_delay:** postgres-primary uses `down_delay = 60` (longer shutdown wait, as the primary should be the last postgres resource to stop). postgres-replica uses `down_delay = 30`.

**No variables needed:** The boot order values for postgres are hardcoded constants — they are not parameterized. Only k3s-cluster uses variables for startup order (because of `for_each` with dynamic index).

## Dev Agent Record

### Agent Model Used: ``
### Completion Notes List

### Change List
- `TargetProjects/terminus/infra/terminus.infra/tofu/modules/postgres-cluster/main.tf` — modified (on_boot + startup on postgres_primary and postgres_replica)
