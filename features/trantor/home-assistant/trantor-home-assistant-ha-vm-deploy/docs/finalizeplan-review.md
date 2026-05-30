---
feature: trantor-home-assistant-ha-vm-deploy
doc_type: finalizeplan-review
phase: finalizeplan
source: phase-complete
verdict: pass-with-warnings
updated_at: 2026-05-30T00:00:00Z
---

# FinalizePlan Adversarial Review — Home Assistant VM Deployment

## Review Scope

| Item | Value |
|---|---|
| Feature | trantor-home-assistant-ha-vm-deploy |
| Track | express |
| Phase | finalizeplan |
| Source | phase-complete |
| Reviewed artifacts | business-plan.md, tech-plan.md, sprint-plan.md |
| Predecessor review | ExpressPlan adversarial review (conditional pass, carried findings) |

---

## Adversarial Analysis

### Layer 1 — Logic Flaws

| # | Finding | Severity | Disposition |
|---|---|---|---|
| L1-01 | tech-plan.md says target repo decision is deferred to "story time" — but Story 1 lists this as its first task ("Confirm target repo"). A story cannot begin without knowing its write-scope target repo; this creates a chicken-and-egg gate that blocks Story 1 from ever starting. The target repo must be declared before dev starts. | HIGH | **Must resolve before dev** |
| L1-02 | Sprint-plan dependency chain shows Story 4 (Vault seed) depends on Story 1 ("Vault path namespace established per tech plan"). But Story 1 is an OpenTofu scaffold — it does not create any Vault paths. Vault path creation is entirely separate from VM provisioning. Story 4's dependency annotation is misleading. | MEDIUM | Clarify dependency in sprint-plan; Story 4 depends on Vault access (pre-existing), not Story 1 |
| L1-03 | HAOS disk image download/import is described as a task inside Story 2 — but `qm importdisk` is a Proxmox CLI command that runs on the Proxmox host, not a thing OpenTofu does natively. If this is wrapped in a `null_resource`, it must be tested as such. The plan does not articulate the exact import mechanism well enough for Story 2 to be implementation-ready. | MEDIUM | Clarify in story file: document the exact import path (manual `qm importdisk` with runbook entry, OR OpenTofu `null_resource` with explicit command) |

### Layer 2 — Coverage Gaps

| # | Finding | Severity | Disposition |
|---|---|---|---|
| L2-01 | No OpenTofu state backend is specified. Local state is fine for a single operator but loses state on machine failure. The terminus.infra pattern should be checked — does it use remote state (S3/minio)? If so, this feature should follow the same pattern. | MEDIUM | Add story task to confirm/configure state backend |
| L2-02 | Tailscale auth key in HA add-on — the add-on does NOT read from Vault at runtime. The key is entered manually in the add-on config UI or via HA CLI. The Vault path is a safe storage location for the operator (so it doesn't get lost), but the plan doesn't explicitly describe the manual step to retrieve and enter it. This gap could confuse a future operator following the runbook. | MEDIUM | Runbook must explicitly state: "read key from Vault → paste into HA Tailscale add-on config" |
| L2-03 | No HAOS version pinning strategy. "Latest stable" is a valid starting point but means the qcow2 download URL changes with each release, breaking idempotent re-runs. The runbook should capture the specific version used and the OpenTofu module should parameterize the image URL or checksum. | LOW | Document in runbook; add image_url/version variable to OpenTofu module |
| L2-04 | HA HTTP vs HTTPS access is mentioned in tech-plan but the DoD says "HA is deployed and accessible via ha.trantor.internal" — no port is specified. Port 8123 (HTTP) vs 443 (HTTPS) needs to be explicit in the success criteria to make Story 7 DoD validation unambiguous. | LOW | Clarify DoD port/protocol in business-plan success criteria |

### Layer 3 — Cross-Feature Dependencies

| # | Finding | Severity | Disposition |
|---|---|---|---|
| L3-01 | Internal DNS server type is explicitly unresolved. The trantor network uses `.trantor.internal` — if this is resolved by CoreDNS running in k3s, adding `ha.trantor.internal` requires a CoreDNS ConfigMap change in `terminus.infra`, which goes through ArgoCD. Story 3 cannot be implemented without knowing this first. This is a hidden dependency on the terminus.infra delivery pipeline. | HIGH | Must identify DNS server before Story 3 can be estimated; add pre-investigation task to Story 3 |
| L3-02 | Vault path `secret/trantor/home-assistant` is a new Vault namespace. The plan assumes Vault is accessible and the operator has write permission to this path. If the Vault token in use (or the AppRole/policy) doesn't cover `secret/trantor/*`, Story 4 will fail silently. | MEDIUM | Add Vault policy verification to Story 4 pre-check |

### Layer 4 — Assumptions and Blind Spots

| # | Finding | Severity | Disposition |
|---|---|---|---|
| L4-01 | The Proxmox bpg OpenTofu provider version is not pinned. "Same version as terminus.infra" is a reference constraint, not a specific version. Any drift in the provider registry between when this module is written and when terminus.infra next updates could cause incompatibility. | LOW | Pin `bpg/proxmox` provider version explicitly in `versions.tf` |
| L4-02 | The plan assumes Proxmox QEMU guest agent will be available for health checking. HAOS does not install QEMU guest agent by default — it has its own guest tools. The testing strategy row "Proxmox guest agent / ping" may not work as expected. | LOW | Change test method for VM health check to ping + curl (not guest agent) |
| L4-03 | No mention of HAOS backup strategy. This is out of scope for this feature but the runbook should note that the "rebuild-from-scratch" procedure does NOT restore existing automations/integrations — those are lost unless HA backup is configured separately. | INFO | Note in runbook as out-of-scope limitation |

---

## Party-Mode Blind Spot Challenge

**Winston (Architect):** The target repo gap is the real blocker. You have OpenTofu as the IaC tool but no repository to put it in — and the decision criteria in the plan are vague ("simpler to reason about"). What's the actual decision gate: is there a governance reason terminus.infra shouldn't own trantor homelab infra, or is this just aesthetic preference? Resolving it vaguely at Story 1 time means no story can be committed to a branch before then.

**Bob (SM):** Story 3 (DNS) says "confirm which DNS server handles trantor.internal" as its first task. That's a discovery task, not an implementation task. If you don't know the answer right now, Story 3 estimate is unreliable. Can you answer this before sprint start?

**Inquisitor Greyfax (Analyst):** The DoD says "HA is deployed and accessible via ha.trantor.internal." That's the minimum bar. But Tailscale access is also listed as a success criterion — is that part of the DoD or a stretch goal? If Tailscale fails to connect, does the sprint succeed or fail? This should be unambiguous before execution starts.

### Blind Spot Challenge Questions for Operator

1. **Target repo — decide now:** Will the HAOS OpenTofu module live in a new `trantor.ha-infra` repo (needs creation, onboarding, Vault, CI) or as a directory in `terminus.infra` (already has Vault, CI, provider versions pinned)? Which is less friction?
2. **DNS server type:** What resolves `.trantor.internal` right now — is it CoreDNS in k3s (ConfigMap in terminus.infra), or a standalone dnsmasq/bind/Pi-Hole?
3. **Tailscale in DoD:** Is Tailscale access part of the Definition of Done (sprint fails without it), or is it a secondary goal (sprint passes with local access, Tailscale is best-effort)?
4. **HAOS version:** Do you want to pin a specific HAOS version for reproducibility, or is "latest stable at time of install" acceptable with the version documented in the runbook post-install?
5. **OpenTofu state:** Does terminus.infra use a remote state backend (minio/S3/Consul)? Should the HA module use the same backend or local state is fine?

---

## Pre-Review Fixes Applied

All open items resolved by operator input. Applied to planning artifacts (tech-plan.md, sprint-plan.md, business-plan.md):

| Item | Resolution |
|---|---|
| Target repo (L1-01) | `terminus.infra` — `trantor/home-assistant/` module directory |
| Static IP (L2-04 / VM spec) | `10.0.0.250` — hardcoded in module variables |
| DNS server type (L3-01) | Synology NAS DNS (standalone; no CoreDNS/ArgoCD dependency) |
| DNS record (Story 3) | `ha.trantor.internal` → `10.0.0.250` via Synology DNS admin UI |
| Story 4 dependency (L1-02) | Changed to "None / Vault access pre-existing" + Vault policy check task |
| HAOS disk import mechanism (L1-03) | Manual `qm importdisk` on Proxmox host CLI; not wrapped in null_resource |
| OpenTofu state backend (L2-01) | Same remote backend as terminus.infra |
| Tailscale DoD scope | Best-effort; hard DoD = `ha.trantor.internal` accessible on LAN |
| TLS acknowledgment | HTTP-only LAN explicitly acceptable; deferred to future feature |
| VM health test method (L4-02) | Changed to `ping 10.0.0.250` (HAOS has no QEMU guest agent) |

---

## Deferred Items

| Item | Deferral Rationale |
|---|---|
| HAOS backup strategy | Out of scope; single-operator homelab; note in runbook |
| HTTPS/TLS for ha.trantor.internal | Explicit risk acceptance; HTTP-only LAN access is acceptable for this feature; deferred to future feature |
| USB dongle passthrough | Not needed for this feature; Proxmox node selection should accommodate it |
| GitHub Actions CI/CD for OpenTofu | Not in scope; manual `tofu apply` follows terminus.infra k3s VM pattern |

---

## Verdict

**PASS-WITH-WARNINGS → RESOLVED — CLEARED FOR BUNDLE GENERATION**

All critical pre-dev blockers (L1-01, L3-01) have been resolved by operator input. All high findings have been applied as pre-bundle fixes to planning documents. No hard gate violations.

Remaining low-severity items (HAOS version pinning, bpg provider version pin, backup strategy note) are tracked as story-level tasks or runbook notes — they do not block bundle generation or dev phase.

Bundle generation may proceed.
