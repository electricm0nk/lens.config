---
feature: trantor-home-assistant-ha-vm-deploy
story_id: HA-1
doc_type: story
status: not-started
title: Scaffold OpenTofu module for HAOS VM
depends_on: []
updated_at: "2026-05-30T00:00:00Z"
---

# Story HA-1 — Scaffold OpenTofu module for HAOS VM

## Context

Feature: `trantor-home-assistant-ha-vm-deploy`  
Epic: Epic 1 — HAOS VM Provisioning  
Target repo: `terminus.infra` at path `trantor/home-assistant/`  
Track: express

## Goal

Create the OpenTofu module in `terminus.infra` to declare the HAOS Proxmox VM resource. No `tofu apply` yet — this story only establishes the scaffolded, plan-passing IaC module.

## Background

The tech-plan specifies deploying HAOS on Proxmox as a dedicated VM. OpenTofu with the bpg/proxmox provider is the IaC tool (same as existing terminus.infra k3s VM provisioning). The module lives in `terminus.infra` under `trantor/home-assistant/`. Static IP is `10.0.0.250`. Remote state backend must match the terminus.infra backend already in use.

## Tasks

1. Add `trantor/home-assistant/` directory to `terminus.infra` repo
2. Create `versions.tf` — pin bpg/proxmox provider to the **same version** already used in terminus.infra (check existing `versions.tf` files)
3. Create `main.tf` — define `proxmox_virtual_environment_vm` resource with:
   - CPU: 2 vCPUs
   - RAM: 4096 MB
   - Disk: 32 GB, virtio or scsi depending on HAOS requirements
   - Network: VirtIO NIC on the same LAN bridge as k3s nodes
   - Static IP: `10.0.0.250` configured via HAOS network initialization or cloud-init equivalent (note: HAOS doesn't support standard cloud-init; IP may need to be set post-boot via HAOS Network UI or via Proxmox network config)
4. Create `variables.tf` — externalize: `proxmox_node`, `vm_id`, `storage`, `bridge`, `static_ip` (default `10.0.0.250`)
5. Create `outputs.tf` — output VM ID and IP
6. Configure backend: match terminus.infra remote state backend (same bucket/workspace)
7. Confirm Proxmox node target: check `tofu show` or Proxmox UI for available resources; note whether selected node has USB controller (future USB dongle passthrough consideration)
8. Run `tofu init && tofu plan` — verify no errors
9. Commit to git; verify no secrets in source

## Acceptance Criteria

- [ ] `trantor/home-assistant/` directory exists in `terminus.infra`
- [ ] `versions.tf` pins bpg/proxmox provider (same version as existing terminus.infra modules)
- [ ] `tofu plan` succeeds with no errors; plan shows creation of one VM resource
- [ ] VM spec in plan: 2 vCPU, 4 GB RAM, 32 GB disk, VirtIO NIC, static IP `10.0.0.250`
- [ ] Remote state backend matches terminus.infra pattern
- [ ] No secrets in source (`git diff --cached -- '*secret*' '*key*' '*auth*'` clean)
- [ ] Proxmox node selected and documented (in `variables.tf` default or README)

## Notes

- HAOS does not support standard cloud-init for IP assignment the same way generic Linux VMs do. Static IP is set through HAOS UI (Settings → System → Network) after first boot, or via a Proxmox VM-level network config. The OpenTofu module should declare the intended static IP in variables as documentation even if HAOS applies it on first-boot rather than via provider.
- bpg provider version should be checked in `TargetProjects/fourdogs/central/` (or wherever terminus.infra exists locally) to ensure version match.

## Open Questions

None — all pre-implementation questions resolved in FinalizePlan.
