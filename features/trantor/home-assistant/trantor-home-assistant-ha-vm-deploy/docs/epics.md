---
feature: trantor-home-assistant-ha-vm-deploy
doc_type: epics
status: ready
updated_at: 2026-05-30T00:00:00Z
---

# Epics — Home Assistant VM Deployment

## Epic 1: HAOS VM Provisioning

**Goal:** Provision the Home Assistant OS Proxmox VM via OpenTofu IaC so it is running, accessible on the local network, and reproducible from committed infrastructure code.

**Scope:**
- OpenTofu module scaffold in `terminus.infra` (`trantor/home-assistant/` directory)
- Proxmox VM creation: 2 vCPU / 4 GB RAM / 32 GB disk, VirtIO NIC on LAN bridge
- HAOS disk image import via `qm importdisk` and boot disk attachment
- Static IP `10.0.0.250` assigned in HAOS network settings
- VM accessible at `http://10.0.0.250:8123` on local network

**Stories:** HA-1, HA-2

**Acceptance Criteria:**
- `tofu plan` shows no drift against committed module
- VM running in Proxmox at `10.0.0.250`
- HAOS onboarding screen reachable from local network

---

## Epic 2: Network Access and DNS

**Goal:** Make Home Assistant reliably accessible by hostname on the local network.

**Scope:**
- `ha.trantor.internal` A record → `10.0.0.250` added to Synology NAS DNS
- DNS resolution verified from all local devices
- DNS addition procedure documented in runbook

**Stories:** HA-3

**Acceptance Criteria:**
- `nslookup ha.trantor.internal` resolves to `10.0.0.250` on all local network devices
- DNS registration is documented and reproducible

---

## Epic 3: Secrets and Remote Access

**Goal:** Provision Tailscale auth key in Vault and configure the HA Tailscale add-on for remote access.

**Scope:**
- Vault path `secret/trantor/home-assistant/tailscale-auth-key` seeded with reusable auth key
- HA Tailscale add-on installed and configured
- HA accessible via Tailscale network (best-effort)

**Stories:** HA-4, HA-5

**Acceptance Criteria:**
- Vault path exists with valid key; no secret in git
- HA Tailscale add-on running; HA VM visible in Tailscale console
- HA UI reachable via Tailscale from a connected device (best-effort)

---

## Epic 4: Runbook and Sign-off

**Goal:** Produce a complete, tested rebuild-from-scratch runbook and confirm all hard DoD criteria are met.

**Scope:**
- `runbook.md` covering full rebuild procedure (Article 11 requirement)
- Final DoD validation: `ha.trantor.internal` accessible, static IP stable, no secrets in git, OpenTofu no-drift

**Stories:** HA-6, HA-7

**Acceptance Criteria:**
- `runbook.md` complete and peer-reviewed against a tested rebuild
- All hard DoD criteria satisfied
