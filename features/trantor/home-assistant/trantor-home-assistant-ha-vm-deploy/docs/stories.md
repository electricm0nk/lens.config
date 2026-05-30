---
feature: trantor-home-assistant-ha-vm-deploy
doc_type: stories
status: ready
updated_at: 2026-05-30T00:00:00Z
---

# Stories — Home Assistant VM Deployment

## Story Index

| Story ID | Epic | Title | Depends On |
|---|---|---|---|
| HA-1 | Epic 1 | Scaffold OpenTofu module for HAOS VM | — |
| HA-2 | Epic 1 | Provision HAOS VM via OpenTofu | HA-1 |
| HA-3 | Epic 2 | Configure DNS — `ha.trantor.internal` | HA-2 |
| HA-4 | Epic 3 | Seed Vault secret — Tailscale auth key | — |
| HA-5 | Epic 3 | Configure Tailscale add-on in HA | HA-2, HA-4 |
| HA-6 | Epic 4 | Write `runbook.md` — HAOS VM rebuild-from-scratch | HA-2, HA-3, HA-4, HA-5 |
| HA-7 | Epic 4 | DoD validation and sign-off | HA-6 |

---

## Story HA-1 — Scaffold OpenTofu module for HAOS VM

**Epic:** Epic 1  
**Goal:** Create the OpenTofu module to provision the HAOS Proxmox VM. No apply yet.

**Tasks:**
1. Add `trantor/home-assistant/` module directory to `terminus.infra` target repo
2. Initialize OpenTofu module: `versions.tf` with pinned bpg/proxmox provider (match existing terminus.infra version)
3. Define VM resource: 2 vCPU, 4 GB RAM, 32 GB disk, VirtIO NIC on LAN bridge, static IP `10.0.0.250`
4. Configure remote state backend matching terminus.infra backend (same bucket/workspace or equivalent)
5. Confirm Proxmox node target (check available resource headroom; note USB controller position for future dongle passthrough)
6. Commit module with no secrets in source
7. Run `tofu plan` to verify module is syntactically valid

**Acceptance Criteria:**
- `tofu plan` succeeds with no errors against the defined Proxmox VM resource
- VM spec matches: 2 vCPU, 4 GB RAM, 32 GB disk, VirtIO NIC, static IP `10.0.0.250`
- bpg/proxmox provider version pinned in `versions.tf`
- Remote state backend configured matching terminus.infra pattern
- Module committed to git with no secrets in source

**Depends on:** —

---

## Story HA-2 — Provision HAOS VM via OpenTofu

**Epic:** Epic 1  
**Goal:** Apply the OpenTofu module to create the Proxmox VM and boot HAOS.

**Tasks:**
1. Download latest stable HAOS `.qcow2` disk image for x86_64 from https://github.com/home-assistant/operating-system/releases — note the specific version in the runbook for reproducibility
2. Import disk image to Proxmox via `qm importdisk` on the Proxmox host CLI:
   ```bash
   qm importdisk <vmid> /path/to/haos_ova-<version>.qcow2 <storage>
   ```
   Document this command in runbook (manual step; not wrapped in OpenTofu null_resource)
3. Attach imported disk as VM boot disk via OpenTofu disk attachment or Proxmox UI
4. Run `tofu apply` to create the VM
5. Start the VM and confirm HAOS boots (monitor Proxmox console)
6. Verify HAOS onboarding screen is reachable at `http://10.0.0.250:8123`
7. Verify VM responds to ping at `10.0.0.250`

**Acceptance Criteria:**
- VM running in Proxmox at `10.0.0.250`
- `ping 10.0.0.250` succeeds
- HAOS onboarding screen reachable at `http://10.0.0.250:8123`
- VM state managed by OpenTofu (`tofu plan` shows no drift after provision)
- HAOS version used documented in runbook

**Depends on:** HA-1

---

## Story HA-3 — Configure DNS — `ha.trantor.internal`

**Epic:** Epic 2  
**Goal:** Register `ha.trantor.internal` A record in Synology NAS DNS so HA is accessible by hostname.

**Tasks:**
1. Log into Synology NAS DNS Server admin UI
2. Add A record: `ha.trantor.internal` → `10.0.0.250`
3. Save and confirm DNS service reload
4. Document the DNS add-record steps in runbook
5. Validate from multiple devices: `nslookup ha.trantor.internal` returns `10.0.0.250`
6. Validate HA UI reachable at `http://ha.trantor.internal:8123`

**Acceptance Criteria:**
- `ha.trantor.internal` resolves to `10.0.0.250` on all local network devices (laptop, phone, tablet)
- HA UI accessible at `http://ha.trantor.internal:8123`
- DNS record addition steps documented in runbook

**Depends on:** HA-2

---

## Story HA-4 — Seed Vault secret — Tailscale auth key

**Epic:** Epic 3  
**Goal:** Store the Tailscale auth key in Vault before configuring the HA add-on.

**Tasks:**
1. Verify Vault policy covers `secret/trantor/*` (confirm write access from operator credentials)
2. Generate a **reusable** (non-ephemeral) Tailscale auth key in the Tailscale admin console (https://login.tailscale.com/admin/settings/keys)
3. Write key to Vault:
   ```bash
   vault kv put secret/trantor/home-assistant/tailscale-auth-key value=<key>
   ```
4. Confirm Vault path is readable: `vault kv get secret/trantor/home-assistant/tailscale-auth-key`
5. Document key rotation procedure in runbook (generate new key → write to Vault path → reconfigure HA add-on)

**Acceptance Criteria:**
- `vault kv get secret/trantor/home-assistant/tailscale-auth-key` returns a valid key
- No Tailscale key committed to git
- Key rotation procedure documented in runbook

**Depends on:** — (Vault access is pre-existing; verify policy before writing)

---

## Story HA-5 — Configure Tailscale add-on in HA

**Epic:** Epic 3  
**Goal:** Install and configure the HA Tailscale add-on so HA is reachable via Tailscale network.

**Tasks:**
1. Complete HA onboarding (create admin user) via `http://ha.trantor.internal:8123`
2. In HA: Settings → Add-ons → Add-on Store → search "Tailscale" → Install official add-on
3. Read Tailscale auth key from Vault: `vault kv get -field=value secret/trantor/home-assistant/tailscale-auth-key`
4. Configure add-on with retrieved auth key; start the add-on
5. Confirm HA VM appears as a named node in Tailscale admin console
6. From a Tailscale-connected device (phone or laptop on remote network), access HA via the Tailscale IP

**Acceptance Criteria:**
- HA Tailscale add-on installed and running (add-on status = Running)
- HA VM visible as a named node in Tailscale admin console
- HA web UI reachable via Tailscale IP from a Tailscale-connected device

**Note:** This story is best-effort — the sprint passes on hard DoD (local `ha.trantor.internal` access) even if Tailscale is not working. Tailscale access is secondary.

**Depends on:** HA-2 (HA running), HA-4 (Vault key available)

---

## Story HA-6 — Write `runbook.md` — HAOS VM rebuild-from-scratch

**Epic:** Epic 4  
**Goal:** Document a complete, tested rebuild procedure. Required by Article 11.

**Tasks:**
1. Write `runbook.md` covering:
   - **Prerequisites:** Proxmox host access, OpenTofu installed locally, Vault CLI + credentials, Tailscale admin access
   - **HAOS disk image:** exact version used, download URL, `qm importdisk` command with storage target
   - **VM provisioning:** `tofu init`, `tofu plan`, `tofu apply` sequence; confirm VM ID and static IP
   - **DNS registration:** Synology NAS DNS add-record steps for `ha.trantor.internal` → `10.0.0.250`
   - **First-boot onboarding:** HA onboarding flow steps
   - **Tailscale add-on:** install from store, retrieve key from Vault, configure, confirm node in Tailscale console
   - **Validation checklist:** `ping 10.0.0.250`, `nslookup ha.trantor.internal`, `curl http://ha.trantor.internal:8123`, Tailscale IP access
   - **Key rotation:** steps for rotating Tailscale auth key in Vault and HA add-on
   - **Limitation note:** HA backup is not configured; rebuild-from-scratch does NOT restore automations/integrations (those must be reconfigured)
2. Peer-review runbook steps against a clean test (verify commands are accurate)

**Acceptance Criteria:**
- `runbook.md` exists in `docs/trantor/home-assistant/trantor-home-assistant-ha-vm-deploy/`
- All rebuild-from-scratch steps are testable and accurate
- Tailscale key rotation procedure documented
- Runbook limitation (no HA backup = no automation restore) explicitly noted

**Depends on:** HA-2, HA-3, HA-4, HA-5

---

## Story HA-7 — DoD validation and sign-off

**Epic:** Epic 4  
**Goal:** Confirm all hard DoD acceptance criteria are met and the feature is complete.

**Tasks:**
1. Verify `http://ha.trantor.internal:8123` accessible from local network on multiple devices (laptop, phone, tablet)
2. Verify static IP `10.0.0.250` is stable across a full VM reboot cycle (`reboot` via Proxmox → verify ping resumes at same IP)
3. Verify no secrets in git: `git log --all -- '*secret*' '*auth*' '*key*' '*tailscale*'` — expect no matches in terminus.infra
4. Confirm `tofu plan` shows no drift: run from `terminus.infra/trantor/home-assistant/` and verify output is "No changes"
5. Confirm `runbook.md` is complete and accurate
6. Mark feature as Done

**Acceptance Criteria (DoD — Hard):**
- `http://ha.trantor.internal:8123` accessible on local network ✓
- Static IP `10.0.0.250` stable across reboot ✓
- No secrets in git ✓
- `tofu plan` no-drift ✓
- `runbook.md` complete ✓

**Acceptance Criteria (Best-Effort):**
- Tailscale access functional from remote device ✓ (sprint passes without this)

**Depends on:** HA-6
