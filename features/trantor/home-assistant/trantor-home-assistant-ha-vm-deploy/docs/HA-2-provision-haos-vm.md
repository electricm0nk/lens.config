---
feature: trantor-home-assistant-ha-vm-deploy
story_id: HA-2
doc_type: story
status: not-started
title: Provision HAOS VM via OpenTofu
depends_on:
  - HA-1
updated_at: "2026-05-30T00:00:00Z"
---

# Story HA-2 — Provision HAOS VM via OpenTofu

## Context

Feature: `trantor-home-assistant-ha-vm-deploy`  
Epic: Epic 1 — HAOS VM Provisioning  
Target repo: `terminus.infra` at path `trantor/home-assistant/`  
Track: express

## Goal

Apply the OpenTofu module from HA-1 to create the Proxmox VM, import the HAOS disk image, boot HAOS, and confirm the VM is reachable at `http://10.0.0.250:8123`.

## Background

HAOS is distributed as a `.qcow2` disk image. The standard install flow is:
1. Download the HAOS qcow2 image
2. Import it to Proxmox storage using `qm importdisk` on the Proxmox host
3. Attach the imported disk as the VM boot disk
4. Start the VM — HAOS boots and initializes automatically

This is a Proxmox host-level operation for disk import. The OpenTofu module manages the VM resource; disk import is a manual step documented in the runbook (by design — avoids fragile Proxmox SSH null_resource).

## Tasks

1. **Download HAOS image:**
   ```bash
   # On local machine or Proxmox host
   curl -L -O https://github.com/home-assistant/operating-system/releases/latest/download/haos_ova-<version>.qcow2.xz
   xz -d haos_ova-<version>.qcow2.xz
   ```
   Note the exact version number for the runbook.

2. **Upload image to Proxmox storage:**
   ```bash
   scp haos_ova-<version>.qcow2 root@<proxmox-host>:/var/lib/vz/images/
   ```

3. **Get VM ID from OpenTofu plan:** Run `tofu plan` to confirm the VM ID that will be assigned.

4. **Apply OpenTofu module:**
   ```bash
   cd terminus.infra/trantor/home-assistant/
   tofu apply
   ```
   Note the created VM ID.

5. **Import disk on Proxmox host:**
   ```bash
   ssh root@<proxmox-host>
   qm importdisk <vmid> /var/lib/vz/images/haos_ova-<version>.qcow2 <storage>
   # e.g.: qm importdisk 200 /var/lib/vz/images/haos_ova-13.2.qcow2 local-lvm
   ```

6. **Attach disk as boot disk** (via Proxmox UI or `qm set`):
   ```bash
   qm set <vmid> --scsi0 <storage>:vm-<vmid>-disk-0,cache=writeback
   qm set <vmid> --boot c --bootdisk scsi0
   ```

7. **Start VM:**
   ```bash
   qm start <vmid>
   ```
   Monitor Proxmox console — HAOS boots in ~2 minutes.

8. **Verify:**
   ```bash
   ping 10.0.0.250
   curl http://10.0.0.250:8123
   ```

9. **Set static IP in HAOS** (if not auto-applied): Settings → System → Network → configure static IP `10.0.0.250`, gateway, DNS.

## Acceptance Criteria

- [ ] VM running in Proxmox (`qm status <vmid>` = running)
- [ ] `ping 10.0.0.250` succeeds
- [ ] HAOS onboarding screen accessible at `http://10.0.0.250:8123`
- [ ] VM managed by OpenTofu (`tofu plan` shows no drift after provision)
- [ ] HAOS version used recorded in runbook
- [ ] Static IP `10.0.0.250` confirmed in HAOS network settings

## Notes

- HAOS does not have QEMU guest agent installed by default. Health checking is via ping + curl, not guest agent status.
- The disk import + attachment step is manual and runs on the Proxmox host CLI. This is the standard HAOS-on-Proxmox procedure; it is not a limitation.
- If the bpg provider supports direct disk image import in future versions, this can be automated — but for this feature manual import is the approach.

## Open Questions

None.
