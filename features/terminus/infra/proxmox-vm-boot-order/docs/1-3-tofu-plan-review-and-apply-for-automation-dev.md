# Story 1.3: tofu plan review and apply for automation/dev

## Status: ready-for-dev

## Story

As a platform engineer,
I want to run `tofu plan` for the `automation/dev` environment and confirm no destructive changes are planned before applying,
So that VM 9001 is imported into OpenTofu state with boot ordering declared, with zero risk of VM recreation or data loss.

## Acceptance Criteria

- **Given** Stories 1.1 and 1.2 are complete and `tofu init` has been run in `automation/dev`
- **When** `tofu plan` is executed from `TargetProjects/terminus/infra/terminus.infra/tofu/environments/automation/dev/`
- **Then** the plan output shows an import action for `proxmox_virtual_environment_vm.automation_host` (or its module-qualified address)
- **And** the plan shows zero resources to destroy (`-`) and zero forced replacements (`-/+`)
- **And** any in-place updates (`~`) are only for `on_boot`, `startup`, and config fields that differ from current VM state

- **Given** the plan output has been reviewed and confirmed non-destructive
- **When** `tofu apply` is executed
- **Then** it completes without error

- **Given** apply completes successfully
- **When** `tofu state list` is run
- **Then** it includes `module.automation_host.proxmox_virtual_environment_vm.automation_host` (or equivalent module-qualified address)

- **Given** apply completes
- **When** a follow-up `tofu plan` is run
- **Then** it shows "No changes. Your infrastructure matches the configuration." (idempotent)

## Tasks / Subtasks

- [ ] Task 1: Ensure prerequisites are met
  - [ ] Confirm Story 1.1 complete (module exists with import block)
  - [ ] Confirm Story 1.2 complete (environment root exists)
  - [ ] Confirm `tofu init` was run successfully (`.terraform/` exists)
  - [ ] Confirm Proxmox API connectivity from development environment
- [ ] Task 2: Run `tofu plan` and review output
  - [ ] Execute: `cd tofu/environments/automation/dev && tofu plan -out=automation-dev.tfplan`
  - [ ] Inspect plan output for any `-/+` or `-` lines — if any exist, **STOP** and investigate before proceeding
  - [ ] If VM config in module does not match actual VM 9001 state, update `main.tf` in the module and re-plan
- [ ] Task 3: Apply and verify
  - [ ] Execute: `tofu apply automation-dev.tfplan`
  - [ ] Confirm no error output
  - [ ] Run: `tofu state list | grep automation_host` — confirm resource is in state
- [ ] Task 4: Idempotency check
  - [ ] Run: `tofu plan` (fresh plan, no saved planfile)
  - [ ] Confirm output ends with "No changes."
- [ ] Task 5: Commit any module config corrections discovered during plan review
  - [ ] If module main.tf was updated to match VM 9001 actual config, commit those corrections to the feature branch

## Dev Notes

**Plan review is mandatory before apply.** Do not `tofu apply -auto-approve` without reviewing plan output first. The import brings an existing VM under state management — any resource attribute mismatch will cause a force-replacement, which would destroy and recreate VM 9001. This is catastrophic for the automation host.

**Common plan surprises with import:**
- Cloud-init drive settings may differ between declared and actual state
- Network interface MAC address (if declared) must match exactly
- Disk `discard` or `iothread` flags may not be reflected in the current VM config
- If any attribute causes a force-replace, comment it out of the module and re-plan until zero `-/+` entries, then investigate how to declare it correctly

**State address:** After module composition, the state address will be `module.automation_host.proxmox_virtual_environment_vm.automation_host`. Verify this matches what `tofu state list` shows.

**ADR-002 reminder:** The `import {}` block must be retained in the module after apply (NFR-04). Do not remove it — it is idempotent and documents intent.

## Dev Agent Record

### Agent Model Used: ``
### Completion Notes List

### Change List
- (any corrections to `tofu/modules/automation-host/main.tf` discovered during plan review)
