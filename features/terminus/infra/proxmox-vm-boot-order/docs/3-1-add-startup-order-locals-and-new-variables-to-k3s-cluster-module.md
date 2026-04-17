# Story 3.1: Add startup_order locals and new variables to k3s-cluster module

## Status: ready-for-dev

## Story

As a platform engineer,
I want the `k3s-cluster` module's locals and variables extended to carry a `startup_order` field per node,
So that each k3s VM gets a distinct boot order value derived from its position index — without hardcoding order values per-VM.

## Acceptance Criteria

- **Given** `TargetProjects/terminus/infra/terminus.infra/tofu/modules/k3s-cluster/main.tf` has `control_plane_nodes` and `worker_nodes` locals using `for` expressions
- **When** the locals are updated
- **Then** each `control_plane_nodes` entry includes `startup_order = var.control_plane_startup_order_base + index`
- **And** each `worker_nodes` entry includes `startup_order = var.worker_startup_order_base + index`
- **And** `local.all_nodes` (if it merges both maps) continues to work correctly with the new field present

- **Given** `TargetProjects/terminus/infra/terminus.infra/tofu/modules/k3s-cluster/variables.tf` exists
- **When** the variables file is updated
- **Then** it declares four new variables:
  - `control_plane_startup_order_base` — type: number, default: 4
  - `worker_startup_order_base` — type: number, default: 7
  - `startup_up_delay` — type: number, default: 60
  - `startup_down_delay` — type: number, default: 30

- **Given** locals and variables are updated
- **When** `tofu validate` is run from `tofu/modules/k3s-cluster/`
- **Then** it returns no errors

## Tasks / Subtasks

- [ ] Task 1: Read current k3s-cluster module locals
  - [ ] Open `TargetProjects/terminus/infra/terminus.infra/tofu/modules/k3s-cluster/main.tf`
  - [ ] Find `locals {}` block — locate `control_plane_nodes` and `worker_nodes` for-expressions
  - [ ] Note the current structure (e.g., `for index, hostname in local.control_plane_names : hostname => { vm_id = ..., role = ..., ... }`)
  - [ ] Note whether `all_nodes` is a merge of both maps or defined separately
- [ ] Task 2: Update `control_plane_nodes` local
  - [ ] Add `startup_order = var.control_plane_startup_order_base + index` to each map entry
- [ ] Task 3: Update `worker_nodes` local
  - [ ] Add `startup_order = var.worker_startup_order_base + index` to each map entry
- [ ] Task 4: Update `variables.tf`
  - [ ] Add `control_plane_startup_order_base` — number, default 4, with description
  - [ ] Add `worker_startup_order_base` — number, default 7, with description
  - [ ] Add `startup_up_delay` — number, default 60, with description
  - [ ] Add `startup_down_delay` — number, default 30, with description
- [ ] Task 5: Run `tofu validate` from module directory — fix any syntax errors

## Dev Notes

**Locals for-expression pattern:** The current locals likely look like:
```hcl
control_plane_nodes = {
  for index, hostname in local.control_plane_names : hostname => {
    vm_id     = var.control_plane_vm_id_base + index
    role      = "control-plane"
    cpu       = var.control_plane_cpu
    memory_mb = var.control_plane_memory_mb
  }
}
```

Adding `startup_order`:
```hcl
control_plane_nodes = {
  for index, hostname in local.control_plane_names : hostname => {
    vm_id         = var.control_plane_vm_id_base + index
    role          = "control-plane"
    cpu           = var.control_plane_cpu
    memory_mb     = var.control_plane_memory_mb
    startup_order = var.control_plane_startup_order_base + index
  }
}
```

**Order value mapping for dev environment (3 CP nodes, 5 workers):**
- CP nodes: base=4 → VM 300=order 4, VM 301=order 5, VM 302=order 6
- Workers: base=7 → VM 310=order 7, VM 311=order 8, VM 312=order 9, VM 313=order 10, VM 314=order 11

**all_nodes merge:** If `local.all_nodes` is `merge(local.control_plane_nodes, local.worker_nodes)`, it will automatically include `startup_order` from both maps. No change needed to the merge expression.

**Variable descriptions (use these exactly):**
- `control_plane_startup_order_base`: "Starting Proxmox autostart order value for control-plane nodes. Each CP node gets base+index."
- `worker_startup_order_base`: "Starting Proxmox autostart order value for worker nodes. Each worker node gets base+index."
- `startup_up_delay`: "Seconds to wait after QEMU guest agent responds before starting the next-order VM."
- `startup_down_delay`: "Seconds to wait before sending shutdown signal to next VM during Proxmox autostart shutdown."

## Dev Agent Record

### Agent Model Used: ``
### Completion Notes List

### Change List
- `TargetProjects/terminus/infra/terminus.infra/tofu/modules/k3s-cluster/main.tf` — modified (startup_order in locals)
- `TargetProjects/terminus/infra/terminus.infra/tofu/modules/k3s-cluster/variables.tf` — modified (4 new variables)
