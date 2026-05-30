---
feature: trantor-home-assistant-ha-vm-deploy
story_id: HA-4
doc_type: story
status: not-started
title: Seed Vault secret — Tailscale auth key
depends_on: []
updated_at: "2026-05-30T00:00:00Z"
---

# Story HA-4 — Seed Vault secret — Tailscale auth key

## Context

Feature: `trantor-home-assistant-ha-vm-deploy`  
Epic: Epic 3 — Secrets and Remote Access  
Target repo: terminus.infra (write scope: `trantor/home-assistant/`)  
Track: express

## Goal

Generate a reusable Tailscale auth key, store it in Vault at `secret/trantor/home-assistant/tailscale-auth-key`, and document the rotation procedure.

## Background

The HA Tailscale add-on requires a Tailscale auth key to register the HA VM as a Tailscale node. Per Article 9, secrets are stored in Vault — not in git. This story creates the Vault path and seeds the key before Story HA-5 configures the add-on.

This story has no dependency on other stories — it can be executed in parallel with HA-1/HA-2.

## Tasks

1. **Verify Vault policy:**
   ```bash
   vault token capabilities secret/trantor/home-assistant/tailscale-auth-key
   # Expected: create, read, update (write access)
   ```
   If policy does not cover `secret/trantor/*`, add a policy entry before proceeding.

2. **Generate reusable Tailscale auth key:**
   - Open Tailscale admin console: https://login.tailscale.com/admin/settings/keys
   - Click "Generate auth key"
   - Select: **Reusable** (important — ephemeral keys expire and cannot re-register after expiry)
   - Description: "home-assistant-haos-vm"
   - Copy the key

3. **Write to Vault:**
   ```bash
   vault kv put secret/trantor/home-assistant/tailscale-auth-key value=<key>
   ```

4. **Verify:**
   ```bash
   vault kv get secret/trantor/home-assistant/tailscale-auth-key
   # Should return the key value (confirm field name = "value")
   ```

5. **Do NOT commit the key to git.**

6. **Document rotation procedure** (for runbook HA-6):
   - Generate new key in Tailscale admin console
   - `vault kv put secret/trantor/home-assistant/tailscale-auth-key value=<new-key>`
   - In HA: Settings → Add-ons → Tailscale → Configuration → update auth key → restart add-on

## Acceptance Criteria

- [ ] `vault kv get secret/trantor/home-assistant/tailscale-auth-key` returns a valid key
- [ ] Key type is **reusable** (not ephemeral) — confirmed in Tailscale admin console
- [ ] Vault policy covers `secret/trantor/*` (write confirmed in pre-check)
- [ ] No key committed to git
- [ ] Key rotation procedure noted for inclusion in runbook

## Notes

- Use a **reusable** key, not an ephemeral key. Ephemeral keys auto-expire and cannot re-register the same device, which would break Tailscale after a VM rebuild.
- The key is read by the operator (not by HA directly) — HA add-on is configured manually by pasting the key from Vault into the add-on config UI (see Story HA-5).

## Open Questions

None.
