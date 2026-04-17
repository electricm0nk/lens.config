# Story 2.3: tofu plan review and apply for postgres/dev

## Status: ready-for-dev

## Story

As a platform engineer,
I want to run `tofu plan` for the `postgres/dev` environment and confirm only in-place updates before applying,
So that postgres-primary and replica/backup VMs have boot ordering declared in Proxmox state with zero risk of VM recreation.

## Acceptance Criteria

- **Given** Stories 2.1 and 2.2 are complete (module changes present in working tree)
- **When** `tofu plan` is executed from the `postgres/dev` environment directory
- **Then** the plan shows only `~` (in-place update) changes for `on_boot` and `startup` fields across postgres-primary (VM 200), postgres-replica (VM 201), and postgres-backup (VM 202)
- **And** the plan shows zero `-/+` (force-replace) entries and zero `-` (destroy) entries

- **Given** the plan is reviewed and confirmed non-destructive
- **When** `tofu apply` is executed
- **Then** it completes without error

- **Given** apply completes
- **When** a follow-up `tofu plan` is run
- **Then** it shows "No changes." (idempotent)

## Tasks / Subtasks

- [ ] Task 1: Locate postgres/dev environment directory
  - [ ] Find the environment root for postgres/dev (likely `tofu/environments/postgres/dev/`)
  - [ ] Confirm `tofu init` was previously run (`.terraform/` exists) or run it now
- [ ] Task 2: Run `tofu plan` and review output
  - [ ] Execute: `tofu plan -out=postgres-dev.tfplan`
  - [ ] Inspect for any `-/+` or `-` lines — if any exist, **STOP** and investigate
  - [ ] Confirm in-place updates (`~`) appear for `on_boot` and `startup` on all three postgres VMs
- [ ] Task 3: Apply and verify
  - [ ] Execute: `tofu apply postgres-dev.tfplan`
  - [ ] Confirm no error output
- [ ] Task 4: Idempotency check
  - [ ] Run: `tofu plan` (fresh, no planfile)
  - [ ] Confirm "No changes."

## Dev Notes

**Expected plan scope:** 3 VM resources should show in-place updates:
- `module.postgres_cluster.proxmox_virtual_environment_vm.postgres_primary` — `~` on_boot, startup
- `module.postgres_cluster.proxmox_virtual_environment_vm.postgres_replica` — `~` on_boot, startup
- `module.postgres_backup.proxmox_virtual_environment_vm.postgres_backup` — `~` on_boot, startup

The actual state addresses depend on how the postgres/dev environment composes the modules. Check the environment's `main.tf` to confirm module names.

**Non-destructive change confirmation (NFR-01):** `on_boot` and `startup {}` are Proxmox API-level fields applied via a `PUT` to the VM config endpoint. The `bpg/proxmox` provider applies these as in-place updates, not replacements. No VM downtime is caused by applying these changes.

**Story 2.1 and 2.2 changes are in the same apply:** Both postgres-cluster and postgres-backup are typically applied from the same environment root (postgres/dev). The plan will cover all three VMs in one shot.

## Dev Agent Record

### Agent Model Used: ``
### Completion Notes List

### Change List
- (no new files — plan and apply of existing module changes from Stories 2.1 and 2.2)
