---
feature: trantor-home-assistant-ha-vm-deploy
story_id: HA-5
doc_type: story
status: not-started
title: Configure Tailscale add-on in HA
depends_on:
  - HA-2
  - HA-4
updated_at: "2026-05-30T00:00:00Z"
---

# Story HA-5 — Configure Tailscale add-on in HA

## Context

Feature: `trantor-home-assistant-ha-vm-deploy`  
Epic: Epic 3 — Secrets and Remote Access  
Target repo: terminus.infra (write scope: `trantor/home-assistant/`)  
Track: express

## Goal

Install and configure the official HA Tailscale add-on so the HA VM is accessible via Tailscale from remote devices.

**Note:** This story is best-effort. The sprint hard DoD is satisfied by local `ha.trantor.internal` access. Tailscale access is secondary.

## Background

The HA Tailscale add-on runs a Tailscale daemon inside the HAOS Supervisor environment. It registers the HA VM as a Tailscale node using the auth key seeded in Story HA-4. Once running, the HA UI is accessible via the HA VM's Tailscale IP from any Tailscale-connected device.

## Tasks

1. **Complete HA onboarding:**
   - Navigate to `http://ha.trantor.internal:8123`
   - Follow onboarding wizard: create admin user, set home name, configure location
   - Note admin credentials (store in `secret/trantor/home-assistant/admin` if desired)

2. **Install Tailscale add-on:**
   - In HA: Settings → Add-ons → Add-on Store
   - Search "Tailscale" → Select official "Tailscale" add-on
   - Click "Install"

3. **Retrieve auth key from Vault:**
   ```bash
   vault kv get -field=value secret/trantor/home-assistant/tailscale-auth-key
   ```
   Copy the key value.

4. **Configure add-on:**
   - In add-on settings → Configuration tab
   - Set `auth_key`: paste the key from Vault
   - Save configuration

5. **Start the add-on:**
   - Click "Start"
   - Watch log — confirm `Login to admin console` message and Tailscale node registration

6. **Verify in Tailscale admin console:**
   - Open https://login.tailscale.com/admin/machines
   - Confirm HA VM appears as a named node (e.g., `home-assistant`)
   - Note the Tailscale IP assigned to the HA node

7. **Test remote access:**
   - From a Tailscale-connected device (phone or laptop on a different network)
   - Navigate to `http://<tailscale-ip>:8123`
   - Confirm HA UI loads

## Acceptance Criteria

- [ ] HA onboarding complete; admin user created
- [ ] Tailscale add-on installed and status = Running
- [ ] HA VM visible as a named node in Tailscale admin console
- [ ] HA UI accessible via Tailscale IP from a connected device

## Sprint Note

This story is **best-effort**. If Tailscale add-on setup fails (auth key issues, network policy), the sprint still passes when local `ha.trantor.internal` access (Story HA-3) is confirmed. Document the failure in the runbook and re-attempt after resolving Tailscale auth.

## Open Questions

None.
