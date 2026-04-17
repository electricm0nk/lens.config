# Story 2.2: Add startup config to postgres-backup module

## Status: ready-for-dev

## Story

As a platform engineer,
I want `on_boot = true` and `startup { order = 3, up_delay = 60, down_delay = 30 }` added to `proxmox_virtual_environment_vm.postgres_backup` in the `postgres-backup` module,
So that postgres-backup starts concurrently with postgres-replica (same tier 3, order=3) on hypervisor reboot.

## Acceptance Criteria

- **Given** `TargetProjects/terminus/infra/terminus.infra/tofu/modules/postgres-backup/main.tf` exists with a `postgres_backup` VM resource
- **When** the `postgres_backup` resource is updated
- **Then** it includes `on_boot = true` at the resource level
- **And** it includes `startup { order = 3, up_delay = 60, down_delay = 30 }` in the resource block

- **Given** the resource is updated
- **When** `tofu validate` is run from the `tofu/modules/postgres-backup/` directory
- **Then** it returns no errors

## Tasks / Subtasks

- [ ] Task 1: Read current postgres-backup module
  - [ ] Open `TargetProjects/terminus/infra/terminus.infra/tofu/modules/postgres-backup/main.tf`
  - [ ] Locate `proxmox_virtual_environment_vm.postgres_backup` resource block
  - [ ] Note current structure (find the right insertion point for `on_boot` and `startup {}`)
- [ ] Task 2: Update `postgres_backup` resource
  - [ ] Add `on_boot = true` (top-level attribute on the resource block)
  - [ ] Add `startup { order = 3; up_delay = 60; down_delay = 30 }` block
- [ ] Task 3: Run `tofu validate` from module directory — fix any syntax errors

## Dev Notes

**Same order=3 as postgres_replica (ADR-003):** postgres-backup and postgres-replica share `order = 3` to start concurrently in tier 3. Neither depends on the other at boot time. Forcing sequential start of replica+backup adds unnecessary recovery delay.

**HCL block syntax:**
```hcl
on_boot = true

startup {
  order      = 3
  up_delay   = 60
  down_delay = 30
}
```

**This is a single-resource, single-module change.** It is intentionally separated from Story 2.1 (postgres-cluster) because postgres-backup is a distinct module with its own main.tf. The separation keeps each story scoped to one atomic change.

## Dev Agent Record

### Agent Model Used: ``
### Completion Notes List

### Change List
- `TargetProjects/terminus/infra/terminus.infra/tofu/modules/postgres-backup/main.tf` — modified (on_boot + startup on postgres_backup)
