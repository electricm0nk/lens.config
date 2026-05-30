---
feature: trantor-home-assistant-ha-vm-deploy
doc_type: implementation-readiness
status: ready
updated_at: 2026-05-30T00:00:00Z
---

# Implementation Readiness — Home Assistant VM Deployment

## Summary

| Gate | Status | Notes |
|---|---|---|
| Business plan | ✅ PASS | Goals, DoD, risks documented; Tailscale scoped as best-effort |
| Tech plan | ✅ PASS | HAOS/Proxmox architecture confirmed; terminus.infra target repo; static IP `10.0.0.250` |
| Sprint plan | ✅ PASS | 7 stories with defined AC, tasks, dependencies |
| Adversarial review | ✅ PASS (warnings resolved) | All blockers cleared; pre-bundle fixes applied |
| Hard gate compliance | ✅ PASS | Article 3 (tech-plan), Article 9 (Vault), Article 11 (runbook story) |
| Target repo | ✅ RESOLVED | `terminus.infra` (`trantor/home-assistant/` module directory) |
| Static IP | ✅ RESOLVED | `10.0.0.250` |
| DNS server | ✅ RESOLVED | Synology NAS DNS; no ArgoCD dependency |
| OpenTofu state | ✅ RESOLVED | Same remote backend as terminus.infra |
| Story dependency chain | ✅ CLEAN | No cycles; HA-1 → HA-2 → HA-3/HA-5; HA-4 independent |
| Implementation-blocking questions | ✅ NONE | All open questions from planning resolved |

**Overall readiness: ✅ IMPLEMENTATION READY**

---

## Article Compliance

### Article 3 — Architecture Documentation

`tech-plan.md` documents the HAOS/Proxmox VM architecture, networking design, VM spec, and deployment methodology. Compliant.

### Article 9 — Security: Secrets in Vault

Tailscale auth key → `secret/trantor/home-assistant/tailscale-auth-key`  
HA admin credentials → `secret/trantor/home-assistant/admin` (optional, HA prompts on first boot)  
No secrets committed to git.  
Story HA-4 requires Vault policy verification before write.  
Compliant.

### Article 11 — Runbook Required

Story HA-6 explicitly produces `runbook.md` covering full rebuild-from-scratch procedure, Tailscale key rotation, DNS registration, and all validation steps. Compliant.

---

## Dependency Map

```
HA-1 (scaffold) 
  └── HA-2 (provision VM)
        ├── HA-3 (DNS)
        └── HA-5 (Tailscale add-on) ← also depends on HA-4

HA-4 (Vault seed) — independent; run parallel to HA-1/HA-2

HA-2 + HA-3 + HA-4 + HA-5 → HA-6 (runbook)
HA-6 → HA-7 (DoD validation)
```

---

## Target Repo

| Repo | Local Path | Module | Write Scope |
|---|---|---|---|
| terminus.infra | `TargetProjects/fourdogs/central` | — | `trantor/home-assistant/` |

**Note:** `terminus.infra` is the resolved target repo for OpenTofu module authorship. The `target_repos` field in `feature.yaml` must be updated before `/dev` activation to include terminus.infra with the `trantor/home-assistant/` write scope.

---

## Known Limitations

- HAOS backup is out of scope. A rebuild-from-scratch restores infrastructure but not HA automations, integrations, or device configs. This limitation is documented in `runbook.md` (HA-6).
- TLS/HTTPS for `ha.trantor.internal` is explicitly deferred. HTTP on LAN is acceptable for this feature. Future feature required for HTTPS.
- Tailscale access is best-effort. Sprint DoD passes on local `ha.trantor.internal` access alone.
- bpg/proxmox provider version must match existing terminus.infra pin (resolved in HA-1, Story task 2).

---

## Readiness Verdict

**PASS — Ready for Dev.**  
All planning blockers resolved. All hard gates satisfied. Story dependency chain is clean. Target repo and write scope confirmed. `/dev` can activate.
