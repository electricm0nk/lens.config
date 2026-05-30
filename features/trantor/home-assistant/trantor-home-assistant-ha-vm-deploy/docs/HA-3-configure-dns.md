---
feature: trantor-home-assistant-ha-vm-deploy
story_id: HA-3
doc_type: story
status: not-started
title: Configure DNS — ha.trantor.internal
depends_on:
  - HA-2
updated_at: "2026-05-30T00:00:00Z"
---

# Story HA-3 — Configure DNS — `ha.trantor.internal`

## Context

Feature: `trantor-home-assistant-ha-vm-deploy`  
Epic: Epic 2 — Network Access and DNS  
Target repo: `terminus.infra` at path `trantor/home-assistant/`  
Track: express

## Goal

Add `ha.trantor.internal` A record to the Synology NAS DNS server, pointing to `10.0.0.250`, so HA is accessible by hostname on the local network.

## Background

Internal DNS for `.trantor.internal` is served by a Synology NAS DNS Server package (standalone; not CoreDNS in k3s). DNS record addition is done via the Synology DNS Server admin UI. There is no ArgoCD or IaC dependency for this story — it is a manual DNS admin operation documented in the runbook.

## Tasks

1. **Open Synology DNS Server UI:**
   - Log into Synology NAS web UI → Package Center → DNS Server

2. **Add forward zone if needed:**
   - Confirm `trantor.internal` forward zone exists (should already exist for other `.trantor.internal` entries)
   - If not, create forward zone `trantor.internal`

3. **Add A record:**
   - Zone: `trantor.internal`
   - Record type: A
   - Name: `ha`
   - IP: `10.0.0.250`
   - TTL: 300 (or zone default)

4. **Save and reload DNS service**

5. **Document the steps** in `runbook.md` (Story HA-6 will finalize the runbook, but note the DNS steps now)

6. **Validate from multiple devices:**
   ```bash
   # From laptop
   nslookup ha.trantor.internal
   # Expected: Address: 10.0.0.250
   
   # From phone: use network tool app or browser
   # Navigate to http://ha.trantor.internal:8123
   ```

7. **Validate HA is reachable by hostname:**
   ```bash
   curl http://ha.trantor.internal:8123
   # Expected: HTTP 200 (HA onboarding or login page)
   ```

## Acceptance Criteria

- [ ] `ha.trantor.internal` resolves to `10.0.0.250` from laptop
- [ ] `ha.trantor.internal` resolves to `10.0.0.250` from at least one additional device (phone or tablet)
- [ ] `http://ha.trantor.internal:8123` returns HA UI (HTTP 200 or redirect)
- [ ] DNS add-record procedure documented in runbook

## Notes

- This is a manual DNS record operation. No git commit is required for the DNS record itself (Synology DNS is not IaC-managed in this homelab). The runbook documents the procedure for reproducibility.
- If Synology DNS ever migrates to IaC in the future, this would become a committed config change. For this feature, manual + runbook is the approach.

## Open Questions

None.
