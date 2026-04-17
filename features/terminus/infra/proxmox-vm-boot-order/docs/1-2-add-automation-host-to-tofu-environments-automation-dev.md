# Story 1.2: Add automation-host to tofu/environments/automation/dev

## Status: ready-for-dev

## Story

As a platform engineer,
I want a new `tofu/environments/automation/dev/` environment root that composes the `automation-host` module,
So that the automation-dev VM is managed within the standard environment layout and `tofu init` / `tofu plan` / `tofu apply` can be run from a consistent directory without special pre-steps.

## Acceptance Criteria

- **Given** `TargetProjects/terminus/infra/terminus.infra/tofu/environments/automation/dev/` does not exist
- **When** the environment root is created with `main.tf`, `variables.tf`, and `terraform.tf`
- **Then** `main.tf` calls the automation-host module using a relative path (`../../../modules/automation-host`)
- **And** all required module input variables are passed from environment-level variables
- **And** `terraform.tf` declares the `bpg/proxmox` provider with a state backend configuration matching the pattern from `tofu/environments/k3s/dev/terraform.tf` (Consul backend, workspace path convention)

- **Given** the environment root is created
- **When** `tofu init` is run from `tofu/environments/automation/dev/`
- **Then** it completes without error and `.terraform/` is populated with provider plugins

## Tasks / Subtasks

- [ ] Task 1: Inspect existing environment structure
  - [ ] Read `tofu/environments/k3s/dev/terraform.tf` — note Consul backend config, workspace naming pattern
  - [ ] Read `tofu/environments/k3s/dev/main.tf` — note how module is called and variables passed
  - [ ] Read `tofu/environments/postgres/dev/terraform.tf` and `main.tf` for additional pattern confirmation
- [ ] Task 2: Create `tofu/environments/automation/dev/terraform.tf`
  - [ ] Provider block: `bpg/proxmox` with same version constraint as other environments
  - [ ] Backend: Consul backend, workspace path following convention (e.g., `terminus/infra/automation/dev`)
- [ ] Task 3: Create `tofu/environments/automation/dev/variables.tf`
  - [ ] Declare all variables needed to pass to the automation-host module (vm_name, vm_id, target_node, startup_up_delay, startup_down_delay, plus any Proxmox provider connection vars)
  - [ ] Default values should be appropriate for the dev environment
- [ ] Task 4: Create `tofu/environments/automation/dev/main.tf`
  - [ ] Module call: `module "automation_host" { source = "../../../modules/automation-host"; ... }`
  - [ ] Pass all required variables
- [ ] Task 5: Run `tofu init` from `tofu/environments/automation/dev/` — confirm clean initialization

## Dev Notes

**Environment structure pattern:** Follow the same directory and file naming conventions as `tofu/environments/k3s/dev/` — this is the canonical pattern for Terminus infra environments.

**Consul backend workspace:** The workspace path convention for Consul backends is typically `{domain}/{service}/{component}/{environment}`. For this environment it should be something like `terminus/infra/automation/dev`. Match the exact pattern from existing environments.

**Provider connection variables:** The `bpg/proxmox` provider requires endpoint, API token (or username/password), and TLS settings. These are typically passed via environment variables (`PROXMOX_VE_ENDPOINT`, etc.) or via variables declared in the environment. Check how other environments handle this — do not hardcode credentials.

**tofu init prerequisite:** The Consul backend must be reachable for `tofu init` to succeed in environments where remote state is configured. If running in a context without Consul access, `tofu init -backend=false` can be used to verify provider initialization only.

## Dev Agent Record

### Agent Model Used: ``
### Completion Notes List

### Change List
- `TargetProjects/terminus/infra/terminus.infra/tofu/environments/automation/dev/terraform.tf` — new
- `TargetProjects/terminus/infra/terminus.infra/tofu/environments/automation/dev/variables.tf` — new
- `TargetProjects/terminus/infra/terminus.infra/tofu/environments/automation/dev/main.tf` — new
