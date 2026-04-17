# Story 3.2: Add startup block and on_boot to k3s node resource

## Status: ready-for-dev

## Story

As a platform engineer,
I want `on_boot = true` and a `startup {}` block using `each.value.startup_order` added to the `proxmox_virtual_environment_vm.nodes` resource in the k3s-cluster module,
So that each k3s VM is configured for Proxmox autostart at its assigned sequential boot order when the hypervisor starts.

## Acceptance Criteria

- **Given** Story 3.1 is complete and `startup_order` is present in `local.all_nodes` entries
- **When** the `proxmox_virtual_environment_vm.nodes` resource in `main.tf` is updated
- **Then** it includes `on_boot = true` at the resource level (top-level attribute)
- **And** it includes a `startup {}` block:
  ```hcl
  startup {
    order      = each.value.startup_order
    up_delay   = var.startup_up_delay
    down_delay = var.startup_down_delay
  }
  ```

- **Given** the resource is updated
- **When** `tofu validate` is run from `tofu/modules/k3s-cluster/`
- **Then** it returns no errors

## Tasks / Subtasks

- [ ] Task 1: Confirm Story 3.1 is complete
  - [ ] Verify `startup_order` is present in `local.control_plane_nodes` and `local.worker_nodes`
  - [ ] Verify `startup_up_delay` and `startup_down_delay` variables exist in `variables.tf`
- [ ] Task 2: Locate the k3s node resource
  - [ ] Find `resource "proxmox_virtual_environment_vm" "nodes"` in `main.tf`
  - [ ] Note current resource structure — find appropriate insertion points for `on_boot` and `startup {}`
- [ ] Task 3: Add `on_boot = true` to the resource
  - [ ] Place at top-level of the resource block (alongside `name`, `node_name`, `vm_id`)
- [ ] Task 4: Add `startup {}` block to the resource
  - [ ] Reference `each.value.startup_order`, `var.startup_up_delay`, `var.startup_down_delay`
- [ ] Task 5: Run `tofu validate` from module directory — fix any syntax errors

## Dev Notes

**Full resource context:**
```hcl
resource "proxmox_virtual_environment_vm" "nodes" {
  for_each = local.all_nodes

  name      = each.key
  node_name = var.target_node
  vm_id     = each.value.vm_id

  on_boot = true

  startup {
    order      = each.value.startup_order
    up_delay   = var.startup_up_delay
    down_delay = var.startup_down_delay
  }

  # ... existing config unchanged ...
}
```

**`each.value.startup_order` access:** This works because Story 3.1 added `startup_order` to both `control_plane_nodes` and `worker_nodes` maps, and `all_nodes` is the merge of those maps. Every iteration of `for_each = local.all_nodes` will have `each.value.startup_order` available.

**No hardcoded order values:** The `startup {}` block references variables and the node-specific `startup_order` field — no actual integers are hardcoded in the resource. This is intentional — order values are set at the environment level (Story 3.3).

**`on_boot` is required (see architecture section "Proxmox Autostart Prerequisite"):** Without `on_boot = true`, the `startup {}` block has no effect — Proxmox does not autostart the VM at all.

## Dev Agent Record

### Agent Model Used: ``
### Completion Notes List

### Change List
- `TargetProjects/terminus/infra/terminus.infra/tofu/modules/k3s-cluster/main.tf` — modified (on_boot + startup block on nodes resource)
