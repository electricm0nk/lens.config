# Story 3.3: Update k3s/dev environment with startup order variables

## Status: ready-for-dev

## Story

As a platform engineer,
I want the `tofu/environments/k3s/dev/` environment root to explicitly pass the startup order variables to the k3s-cluster module,
So that CP nodes receive orders 4–6 and workers receive orders 7–11, each with 60-second up_delay in the dev environment.

## Acceptance Criteria

- **Given** `TargetProjects/terminus/infra/terminus.infra/tofu/environments/k3s/dev/main.tf` exists with a call to the k3s-cluster module
- **When** the module call is updated to include startup order variables
- **Then** it passes:
  - `control_plane_startup_order_base = 4`
  - `worker_startup_order_base        = 7`
  - `startup_up_delay                 = 60`
  - `startup_down_delay               = 30`

- **Given** the environment is updated
- **When** `tofu validate` is run from `tofu/environments/k3s/dev/`
- **Then** it returns no errors

## Tasks / Subtasks

- [ ] Task 1: Read current k3s/dev environment
  - [ ] Open `TargetProjects/terminus/infra/terminus.infra/tofu/environments/k3s/dev/main.tf`
  - [ ] Locate the `module "k3s_cluster"` (or similarly named) call block
  - [ ] Note the current variable arguments being passed
- [ ] Task 2: Add the four startup order variables to the module call
  - [ ] `control_plane_startup_order_base = 4`
  - [ ] `worker_startup_order_base = 7`
  - [ ] `startup_up_delay = 60`
  - [ ] `startup_down_delay = 30`
- [ ] Task 3: Run `tofu validate` from environment directory — fix any errors
  - [ ] If validation fails because variables aren't declared in the environment's `variables.tf`, add them there

## Dev Notes

**Why explicit values:** The module defaults already match these values (4, 7, 60, 30). However, declaring them explicitly in the environment module call follows the Terminus infra convention of explicit over implicit — it makes the boot order immediately visible to anyone reading the environment configuration, and enables env-specific overrides without module changes.

**Expected module call result (3 CP nodes + 5 workers = 8 k3s VMs):**
- k3s-cp-01 (VM 300) → order 4
- k3s-cp-02 (VM 301) → order 5
- k3s-cp-03 (VM 302) → order 6
- k3s-worker-01 (VM 310) → order 7
- k3s-worker-02 (VM 311) → order 8
- k3s-worker-03 (VM 312) → order 9
- k3s-worker-04 (VM 313) → order 10
- k3s-worker-05 (VM 314) → order 11

**If the environment has a separate `variables.tf`:** Add the four variables there with the values as defaults (or without defaults if the environment always passes them explicitly). Match whatever convention the existing variables in the environment use.

**This is a pure configuration change** — no module logic changes, no new resources. The variables enable the locals and resource changes from Stories 3.1 and 3.2 to take effect with the correct values.

## Dev Agent Record

### Agent Model Used: ``
### Completion Notes List

### Change List
- `TargetProjects/terminus/infra/terminus.infra/tofu/environments/k3s/dev/main.tf` — modified (4 startup order vars added to module call)
- `TargetProjects/terminus/infra/terminus.infra/tofu/environments/k3s/dev/variables.tf` — modified if needed (new vars declared)
