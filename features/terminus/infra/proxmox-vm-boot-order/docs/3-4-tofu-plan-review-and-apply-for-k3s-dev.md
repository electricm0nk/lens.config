# Story 3.4: tofu plan review and apply for k3s/dev

## Status: ready-for-dev

## Story

As a platform engineer,
I want to run `tofu plan` for the `k3s/dev` environment and confirm only in-place updates before applying,
So that all k3s nodes receive sequential boot ordering configuration with zero risk of VM recreation or kubelet disruption.

## Acceptance Criteria

- **Given** Stories 3.1, 3.2, and 3.3 are complete (module and environment changes present in working tree)
- **When** `tofu plan` is executed from the k3s/dev environment directory
- **Then** the plan shows only `~` (in-place update) changes for `on_boot` and `startup` fields
- **And** the plan shows zero `-/+` (force-replace) entries and zero `-` (destroy) entries
- **And** the plan output shows updates for all 8 k3s nodes (3 CP + 5 workers)

- **Given** the plan is reviewed and confirmed non-destructive
- **When** `tofu apply` is executed
- **Then** it completes without error

- **Given** apply completes
- **When** a follow-up `tofu plan` is run
- **Then** it shows "No changes." (idempotent)

- **Given** all three epics are applied (Epic 1 automation-host, Epic 2 postgres, Epic 3 k3s)
- **When** the operator performs a controlled reboot of the `trantor` hypervisor (per `.todo` item `[BOOT-ORDER-ACCEPTANCE]`)
- **Then** VMs autostart in the configured order:
  - VM 9001 starts first (order=1), gate 120s
  - VM 200 starts second (order=2), gate 90s
  - VMs 201+202 start concurrently third (order=3), gate 60s
  - VMs 300-302 start fourth through sixth (orders 4-6), gate 60s each
  - VMs 310-314 start seventh through eleventh (orders 7-11), gate 60s each
- **And** k3s control-plane quorum is established before workers register
- **And** all workers join the cluster without manual `kubectl uncordon` or service restarts
- **And** `up_delay` values are observed and calibrated; any adjustments committed as a follow-up

## Tasks / Subtasks

- [ ] Task 1: Ensure prerequisites are met
  - [ ] Confirm Stories 3.1, 3.2, 3.3 complete
  - [ ] Confirm `tofu init` was previously run in k3s/dev (`.terraform/` exists) or run it now
- [ ] Task 2: Run `tofu plan` and review output
  - [ ] Execute: `tofu plan -out=k3s-dev.tfplan` from `tofu/environments/k3s/dev/`
  - [ ] Inspect for any `-/+` or `-` lines — if any exist, **STOP** and investigate
  - [ ] Confirm `~` updates appear for all 8 k3s VMs (3 CP + 5 workers) for `on_boot` and `startup`
- [ ] Task 3: Apply and verify
  - [ ] Execute: `tofu apply k3s-dev.tfplan`
  - [ ] Confirm no error output
- [ ] Task 4: Idempotency check
  - [ ] Run: `tofu plan` (fresh, no planfile)
  - [ ] Confirm "No changes."
- [ ] Task 5: Schedule acceptance test
  - [ ] Coordinate with operator to perform controlled `trantor` hypervisor reboot per `.todo [BOOT-ORDER-ACCEPTANCE]`
  - [ ] After reboot: observe autostart sequence, confirm boot order, confirm k3s cluster recovers without intervention
  - [ ] Record observed guest-agent response times; compare to `up_delay` values (120s, 90s, 60s)
  - [ ] If adjustments needed: commit updated up_delay values as a follow-up commit to the feature branch

## Dev Notes

**Expected plan scope:** 8 VM resources should show in-place updates:
```
module.k3s_cluster.proxmox_virtual_environment_vm.nodes["k3s-cp-01"]   ~ on_boot, startup
module.k3s_cluster.proxmox_virtual_environment_vm.nodes["k3s-cp-02"]   ~ on_boot, startup
module.k3s_cluster.proxmox_virtual_environment_vm.nodes["k3s-cp-03"]   ~ on_boot, startup
module.k3s_cluster.proxmox_virtual_environment_vm.nodes["k3s-worker-01"] ~ on_boot, startup
module.k3s_cluster.proxmox_virtual_environment_vm.nodes["k3s-worker-02"] ~ on_boot, startup
module.k3s_cluster.proxmox_virtual_environment_vm.nodes["k3s-worker-03"] ~ on_boot, startup
module.k3s_cluster.proxmox_virtual_environment_vm.nodes["k3s-worker-04"] ~ on_boot, startup
module.k3s_cluster.proxmox_virtual_environment_vm.nodes["k3s-worker-05"] ~ on_boot, startup
```
Actual state addresses depend on hostname keys in the node map. The module name in the state address depends on what the k3s/dev environment's `main.tf` names the module.

**Non-destructive (NFR-01):** `on_boot` and `startup {}` are Proxmox API-level VM config fields applied via `PUT`. The `bpg/proxmox` provider applies these as in-place updates. Running k3s workloads are unaffected by applying these changes.

**Apply-time dependency note:** Epic 1 (automation-host) should be applied before this story if possible. The boot sequence has a gap at order=1 until VM 9001 is under management. However, Epics 2 and 3 can be applied independently — Proxmox will simply start all autoboot VMs without an order=1 anchor until Epic 1 is applied.

**Acceptance test scheduling:** The controlled reboot of `trantor` is an operator task requiring physical access or IPMI control. It should be performed during a maintenance window. The `.todo [BOOT-ORDER-ACCEPTANCE]` item tracks this. The acceptance test is the final validation gate for the entire feature.

## Dev Agent Record

### Agent Model Used: ``
### Completion Notes List

### Change List
- (no new files — plan and apply of existing module changes from Stories 3.1, 3.2, 3.3)
- (any up_delay calibration commits as follow-up after acceptance test)
