# Story 1.1: Create automation-host module with import block and startup config

## Status: ready-for-dev

## Story

As a platform engineer,
I want a new `tofu/modules/automation-host` OpenTofu module that declares VM 9001 with an `import {}` block, `on_boot = true`, and `startup { order = 1, up_delay = 120 }`,
So that automation-dev is brought under state management with deterministic boot tier 1 configuration in a version-controlled, reproducible way.

## Acceptance Criteria

- **Given** `TargetProjects/terminus/infra/terminus.infra/tofu/modules/automation-host/main.tf` does not exist
- **When** the module is created
- **Then** it includes an `import {}` block with `to = proxmox_virtual_environment_vm.automation_host` and `id = "trantor/9001"` (Proxmox node/vmid format)
- **And** the `proxmox_virtual_environment_vm.automation_host` resource includes `on_boot = true`
- **And** the resource includes `startup { order = 1, up_delay = 120, down_delay = 60 }`
- **And** the resource includes `agent { enabled = true, timeout = "15m" }` matching other modules
- **And** the resource CPU, memory, disk, and network config matches the actual VM 9001 configuration (confirm via Proxmox API or `tofu plan` — zero force-replacements is the validation gate)

- **Given** `main.tf` is created
- **When** `variables.tf` is created alongside it
- **Then** it declares: `vm_name` (string), `vm_id` (number, default: 9001), `target_node` (string, default: "trantor"), `startup_up_delay` (number, default: 120), `startup_down_delay` (number, default: 60)

- **Given** the full module is created (`main.tf`, `variables.tf`, `terraform.tf`)
- **When** `tofu validate` is run from `tofu/modules/automation-host/`
- **Then** it returns no errors

## Tasks / Subtasks

- [ ] Task 1: Inspect existing VM 9001 to determine resource configuration parity
  - [ ] Query VM 9001 via Proxmox API (`pvesh get /nodes/trantor/qemu/9001/config`) or Proxmox UI
  - [ ] Note: CPU cores/sockets, memory MB, disk size/storage/format, network bridge/model/mac, SCSI controller, cloud-init drive if present
- [ ] Task 2: Create `tofu/modules/automation-host/terraform.tf`
  - [ ] Declare `bpg/proxmox` provider version constraint matching pattern in `tofu/modules/k3s-cluster/terraform.tf` or `tofu/modules/postgres-cluster/terraform.tf`
- [ ] Task 3: Create `tofu/modules/automation-host/variables.tf`
  - [ ] Variables: `vm_name`, `vm_id` (default 9001), `target_node` (default "trantor"), `startup_up_delay` (default 120), `startup_down_delay` (default 60)
  - [ ] Add any additional variables needed for CPU/memory/disk if not hardcoded
- [ ] Task 4: Create `tofu/modules/automation-host/main.tf`
  - [ ] `import {}` block: `to = proxmox_virtual_environment_vm.automation_host`, `id = "${var.target_node}/${var.vm_id}"`
  - [ ] `proxmox_virtual_environment_vm "automation_host"` resource with `on_boot = true`, `startup {}` block, `agent {}` block, and all config fields matching VM 9001 actual config
- [ ] Task 5: Run `tofu validate` from module directory — fix any syntax errors

## Dev Notes

**import {} block format:** The `bpg/proxmox` provider uses `{node}/{vmid}` as the resource ID format. The import block should be:
```hcl
import {
  to = proxmox_virtual_environment_vm.automation_host
  id = "${var.target_node}/${var.vm_id}"
}
```

**Import block idempotency (NFR-04):** Retain the `import {}` block after apply. OpenTofu is idempotent — if the resource is already in state, the import block is a no-op on subsequent applies.

**VM config parity:** The resource declaration must match VM 9001's actual configuration to avoid force-replacement on first apply. This is the most important correctness check for this story. The `tofu plan` output in Story 1.3 is the safety gate — zero `-/+` entries required.

**Startup block:** The automation-dev server runs OpenTofu, Vault CLI, SOPS, and Ansible. It must be available before other services need it for initialization. 120s up_delay is a conservative initial estimate — calibrate during acceptance testing.

**Reference modules:** Look at `tofu/modules/postgres-cluster/main.tf` or `tofu/modules/k3s-cluster/main.tf` for provider block patterns and `agent {}` block conventions.

## Dev Agent Record

### Agent Model Used: ``
### Completion Notes List

### Change List
- `TargetProjects/terminus/infra/terminus.infra/tofu/modules/automation-host/terraform.tf` — new
- `TargetProjects/terminus/infra/terminus.infra/tofu/modules/automation-host/variables.tf` — new
- `TargetProjects/terminus/infra/terminus.infra/tofu/modules/automation-host/main.tf` — new
