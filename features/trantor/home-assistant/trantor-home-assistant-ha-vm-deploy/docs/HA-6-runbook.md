---
feature: trantor-home-assistant-ha-vm-deploy
story_id: HA-6
doc_type: story
status: not-started
title: Write runbook.md — HAOS VM rebuild-from-scratch
depends_on:
  - HA-2
  - HA-3
  - HA-4
  - HA-5
updated_at: "2026-05-30T00:00:00Z"
---

# Story HA-6 — Write `runbook.md` — HAOS VM rebuild-from-scratch

## Context

Feature: `trantor-home-assistant-ha-vm-deploy`  
Epic: Epic 4 — Runbook and Sign-off  
Target repo: terminus.infra (write scope: `trantor/home-assistant/`)  
Track: express

## Goal

Produce a complete, tested `runbook.md` that documents the full rebuild-from-scratch procedure for the HAOS VM. Required by Article 11.

## Background

The org constitution (Article 11) requires a runbook for any infrastructure deployment. The runbook must be accurate enough that a rebuild-from-scratch is possible by following it alone, without referring to external documentation or original session notes.

## Tasks

1. **Write `runbook.md`** in `terminus.infra/trantor/home-assistant/runbook.md` with the following sections:

   ### Prerequisites
   - Proxmox host SSH access
   - OpenTofu installed locally (match version in `versions.tf`)
   - Vault CLI + credentials with access to `secret/trantor/*`
   - Tailscale admin console access
   - Synology NAS DNS Server admin UI access

   ### HAOS Disk Image
   - Exact version used: `haos_ova-<VERSION>.qcow2` (fill in actual version from HA-2)
   - Download URL: `https://github.com/home-assistant/operating-system/releases`
   - `qm importdisk` command with actual storage name and VM ID
   - Note: Document the actual `<vmid>`, `<storage>` values used

   ### VM Provisioning
   ```bash
   cd terminus.infra/trantor/home-assistant/
   tofu init
   tofu plan    # verify expected changes
   tofu apply   # create VM
   ```
   - Record actual VM ID created
   - Disk import steps (from HA-2 procedure)
   - Attach disk and set boot order commands

   ### Static IP Configuration
   - HAOS Settings → System → Network → set static IP `10.0.0.250`, gateway, DNS server

   ### DNS Registration
   - Synology NAS DNS Server admin steps to add/re-add `ha.trantor.internal` → `10.0.0.250`
   - Validate: `nslookup ha.trantor.internal`

   ### First-Boot Onboarding
   - Navigate to `http://ha.trantor.internal:8123`
   - Complete HA onboarding wizard (admin user setup)
   - Credentials stored at `secret/trantor/home-assistant/admin` (optional)

   ### Tailscale Add-on
   - Retrieve key: `vault kv get -field=value secret/trantor/home-assistant/tailscale-auth-key`
   - Install Tailscale add-on from HA add-on store
   - Configure auth key and start add-on
   - Verify in Tailscale admin console

   ### Validation Checklist
   - [ ] `ping 10.0.0.250` succeeds
   - [ ] `nslookup ha.trantor.internal` returns `10.0.0.250`
   - [ ] `curl http://ha.trantor.internal:8123` returns HTTP 200
   - [ ] `tofu plan` shows no drift (0 to add, 0 to change, 0 to destroy)
   - [ ] HA VM visible in Tailscale admin console (best-effort)

   ### Tailscale Key Rotation
   1. Generate new reusable key at https://login.tailscale.com/admin/settings/keys
   2. `vault kv put secret/trantor/home-assistant/tailscale-auth-key value=<new-key>`
   3. In HA: Settings → Add-ons → Tailscale → Configuration → update `auth_key` → restart

   ### Limitation Note
   > Rebuild-from-scratch restores the HAOS VM infrastructure but **does NOT restore HA automations, device integrations, or dashboard configuration**. Those are lost unless HA's native backup feature is configured separately (out of scope for this feature).

2. **Peer-review the runbook:** Walk through the steps against a test (or against the actual deployment just completed) to verify all commands and steps are accurate.

3. **Commit `runbook.md`** to git with no secrets.

## Acceptance Criteria

- [ ] `runbook.md` exists at `terminus.infra/trantor/home-assistant/runbook.md`
- [ ] All sections present: prerequisites, disk image, VM provisioning, DNS, onboarding, Tailscale, validation checklist, key rotation, limitation note
- [ ] Actual values filled in (VM ID, HAOS version, storage name, Proxmox host)
- [ ] No secrets committed to git
- [ ] Runbook peer-reviewed against actual deployment

## Open Questions

None.
