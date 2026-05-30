---
feature: trantor-home-assistant-ha-vm-deploy
story_id: HA-7
doc_type: story
status: not-started
title: DoD validation and sign-off
depends_on:
  - HA-6
updated_at: "2026-05-30T00:00:00Z"
---

# Story HA-7 — DoD validation and sign-off

## Context

Feature: `trantor-home-assistant-ha-vm-deploy`  
Epic: Epic 4 — Runbook and Sign-off  
Target repo: terminus.infra (write scope: `trantor/home-assistant/`)  
Track: express

## Goal

Confirm all hard Definition of Done criteria are met and sign off the feature as complete.

## Background

This is the final story in the sprint. All implementation stories (HA-1 through HA-6) must be complete before this story begins. The DoD is defined in the business plan and implementation readiness document. Hard DoD items block sign-off; Tailscale access is best-effort.

## Tasks

1. **Verify local access:**
   ```bash
   # From laptop on local network
   curl -I http://ha.trantor.internal:8123
   # Expected: HTTP/1.1 200 OK or redirect
   
   # From phone browser — navigate to http://ha.trantor.internal:8123
   # Expected: HA login/onboarding page loads
   ```

2. **Verify static IP stability across reboot:**
   ```bash
   # Reboot VM via Proxmox UI or HAOS Settings → System → Restart
   # Wait ~2-3 minutes for HAOS to boot
   ping 10.0.0.250      # confirm same IP
   nslookup ha.trantor.internal  # confirm DNS still resolves
   curl http://ha.trantor.internal:8123  # confirm HA UI loads
   ```

3. **Verify no secrets in git:**
   ```bash
   cd terminus.infra
   git log --all --diff-filter=A --name-only --pretty="" -- '*secret*' '*auth*' '*key*' '*tailscale*'
   # Expected: no output (no files matching secret patterns ever committed)
   
   # Also check staged changes
   git diff --cached -- '*secret*' '*key*' '*auth*'
   # Expected: no output
   ```

4. **Verify OpenTofu no-drift:**
   ```bash
   cd terminus.infra/trantor/home-assistant/
   tofu plan
   # Expected: "No changes. Your infrastructure matches the configuration."
   ```

5. **Verify runbook completeness:**
   - Read `terminus.infra/trantor/home-assistant/runbook.md`
   - Confirm all sections are present and actual values (VM ID, HAOS version, storage) are filled in
   - Confirm limitation note is present

6. **Verify Tailscale (best-effort):**
   - From a Tailscale-connected device on a different network
   - Navigate to `http://<tailscale-ip>:8123`
   - If HA loads: Tailscale pass. If not: document failure, sprint still passes on hard DoD.

7. **Update feature status** to complete via `/lens complete` or lifecycle state update.

## Acceptance Criteria

### Hard DoD (all required)

- [ ] `http://ha.trantor.internal:8123` accessible on local network ✓
- [ ] Static IP `10.0.0.250` stable across full VM reboot cycle ✓
- [ ] No secrets in git (`git log` scan clean, `git diff --cached` clean) ✓
- [ ] `tofu plan` shows no drift ("No changes") ✓
- [ ] `runbook.md` complete with all required sections and actual values ✓

### Best-Effort (sprint passes without these)

- [ ] Tailscale access functional from remote Tailscale-connected device

## Notes

- **TLS/HTTPS is explicitly deferred.** `http://` on LAN is the accepted protocol for this feature. HTTPS requires a future feature (reverse proxy or HA Nabu Casa / Let's Encrypt configuration).
- If Tailscale fails the best-effort check, document the exact failure in the runbook and create a follow-up task or feature for resolution.

## Open Questions

None.
