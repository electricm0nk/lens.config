---
feature: terminus-inference-hermes-by-nous-research-install
doc_type: businessplan-adversarial-review
phase: businessplan
verdict: pass-with-warnings
reviewer: lens-adversarial-review
source: manual-rerun
date: 2026-05-14
critical_count: 0
high_count: 2
medium_count: 4
low_count: 4
---

# BusinessPlan Adversarial Review
**Feature:** terminus-inference-hermes-by-nous-research-install
**Phase:** businessplan | **Source:** manual-rerun
**Date:** 2026-05-14

## Artifacts Reviewed
- `prd.md` (415 lines, BusinessPlan artifact)
- `ux-design.md` (BusinessPlan artifact)

## Predecessor Context Loaded
- `product-brief.md` (PrePlan)
- `research.md` (PrePlan)
- `preplan-adversarial-review.md` (PrePlan gate — verdict: pass-with-warnings)

---

## Review Summary

Both artifacts are structurally sound. The PRD correctly supersedes the product-brief's inference-gateway endpoint assumption based on research findings (no IngressRoute for gateway → use direct Ollama endpoint). The UX design appropriately scopes to CLI/terminal interaction patterns for a non-web tool.

**No critical findings.** Two high-severity risks require resolution before dev implementation begins (not blocking TechPlan authoring).

---

## Findings

### Critical — 0

None.

---

### High Severity — 2

#### H1: Internet Access from `hermes.trantor.internal` is a potential install blocker

The install script requires `curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash` from the new Proxmox VM. The PRD Dependencies table lists "Requires internet access on VM — Network" with no confirmed status. In `trantor.internal`, new VMs may be on isolated subnets without default internet routing (consistent with the secure-enclave infrastructure posture). The offline fallback ("use uv directly") is vague — it doesn't identify which files are needed or where to get them.

**Impact:** Could block every implementation story for the feature.
**Resolution:** Before implementation begins, confirm internet routing from the new Proxmox VM VLAN with `curl https://hermes-agent.nousresearch.com/install.sh -o /dev/null -w "%{http_code}"`. If routing fails, document the offline install path explicitly in the runbook.

---

#### H2: Postgres hostname `postgres.trantor.internal` is assumed, not confirmed

FR-8.4 explicitly marks the hostname as "TBD — to be confirmed" but the rest of the PRD, UX design, and SQL examples treat it as authoritative. If Postgres runs as a k3s internal service, the correct DNS name may be `postgres.terminus.svc.cluster.local` or similar — unreachable from a VM outside the cluster. If it's a VM-hosted Postgres, the hostname may be different from `postgres.trantor.internal`.

Kaylee and Temporal both have working Postgres connection strings. These are the authoritative reference.

**Impact:** Could break the entire memory backend if the hostname is wrong.
**Resolution:** Before the Postgres memory story begins, verify the hostname by checking the Kaylee or Temporal `DATABASE_URL` secret: `kubectl get secret -n kaylee kaylee-db-url -o jsonpath='{.data.DATABASE_URL}' | base64 -d`. Use the confirmed hostname in FR-8.4 and the runbook.

---

### Medium Severity — 4

#### M1: No network connectivity pre-validation story

The hermes VM must reach both `ollama.trantor.internal:443` AND `postgres.trantor.internal:5432`. These may be on different network segments. No acceptance criterion or story validates network connectivity before attempting installation or Postgres setup.

**Resolution:** Add a network smoke test as the second story after VM provisioning: `curl https://ollama.trantor.internal/v1/models` returns 200 from hermes VM; `psql` connection test returns 1.

---

#### M2: `hermes.trantor.internal` DNS entry not addressed

Neither the PRD nor UX design addresses creation of the DNS A-record for the new VM. In Proxmox environments, DNS is frequently manually managed. Without a DNS entry, SSH access and cross-service references require IP addresses, breaking the runbook's assumed workflows.

**Resolution:** Add DNS provisioning as a sub-step in FR-1 (VM Provisioning) or as a requirement in AC-8 (VM Provisioned via Ansible).

---

#### M3: Hermes memory config key unknown — silent failure risk

The exact YAML key for Postgres memory (`memory.type`, `memory.backend`, `memory.provider`, `memory.database_url`, etc.) is unknown. If the wrong key is used, Hermes will start without error but will not write to Postgres — memory will appear to work within a session but will not persist across restarts. AC-10 tests persistence after configuration, but a misconfigured key produces no error and makes AC-10 fail with no clear diagnostic path.

**Resolution:** Add "verify Hermes memory config schema from upstream docs" as the first task in the Postgres memory implementation story. Add a diagnostic step to the runbook: confirm a record exists in `hermes_memory` after a session by querying `SELECT COUNT(*) FROM <memory_table>;` (table name TBD from Hermes docs).

---

#### M4: Hermes version pinning not addressed

The install script is `curl | bash` with no version pin. The open question ("install from script or pinned commit?") is unresolved. On reinstall after a Hermes release, breaking changes could silently corrupt the Postgres memory schema.

**Resolution:** Note the installed version from `hermes --version` in the runbook immediately after install. Check whether the install script supports `--version` or `--tag` flags. Pin in runbook.

---

### Low Severity — 4

#### L1: FR ordering — FR-8 appears before FR-7

Numbering artifact from insertion order. Minor readability issue only.

---

#### L2: Hardcoded `kaylee` database in UX SQL example

`REVOKE CONNECT ON DATABASE kaylee FROM hermes;` in ux-design.md Section 4.1 hardcodes a database name. The actual revoke list should be dynamic. Replace with: an instruction to "REVOKE CONNECT on all databases other than `hermes_memory`" rather than hardcoded SQL that may not match the cluster's actual DB inventory.

---

#### L3: Password rotation procedure absent

Rotating the `hermes` Postgres user password requires updating `~/.hermes/.env` AND restarting Hermes. This Day-2 procedure is not documented. Not a blocking concern for initial install but should be a runbook appendix section.

---

#### L4: Cross-feature sensing not run

A `terminus-watchdog-llm-engine` feature is in-flight (separate plan branch active in the current session). It also uses the inference stack. Potential overlap: `OLLAMA_CONTEXT_LENGTH` server-side configuration changes could affect both features. No sensing has been run to confirm non-collision.

**Resolution:** Run `bmad-lens-sensing` before TechPlan to check for shared-infrastructure conflicts.

---

## Party-Mode Blind-Spot Challenge

> **3 perspectives, focused pressure round**

**Perturabo (Architect):** This PRD says "Postgres hostname TBD" and then builds FR-8 entirely on that assumption. Two Postgres clients are working today — Kaylee and Temporal. Both resolved their connection strings. Why isn't checking those secrets the first implementation action? The hostname might not be `postgres.trantor.internal` at all.

**Inquisitor Greyfax (Analyst):** Hermes is being established as a top-level domain, but it will start TechPlan with an empty constitution scaffold created by `lens-new-domain`. Any compliance checks in TechPlan will pass trivially since no domain rules exist yet. Should the hermes domain constitution be seeded with at least a local-first requirement and a "VM-only install policy" before TechPlan begins?

**Magos Domina (Dev):** I'm implementing AC-10. I start Hermes, tell it something, kill it, restart it. Memory isn't there. The runbook says "check .env, verify Postgres connectivity, run psql." But maybe the config key is wrong and Hermes silently ran without Postgres. There's no diagnostic step that tells me "is Hermes currently using Postgres or in-memory fallback?" Does Hermes have a `status` command? Or can I confirm writes by querying the DB directly?

---

### Blind-Spot Challenge Questions Raised

1. What is the actual Postgres connection string used by Kaylee or Temporal today? (Resolves H2 definitively)
2. Does the new Proxmox VM VLAN have outbound internet routing? (Resolves H1)
3. Does Hermes Agent have a diagnostic command confirming whether it's using the Postgres memory backend vs. in-memory fallback?
4. What rules should the `hermes` domain constitution have at minimum before TechPlan? (local-first, no-cloud-egress, VM-only-install)
5. Where is the `hermes` Postgres user password stored? The PRD says Ansible vault; the UX design says Bitwarden. These must align.

---

## Verdict: pass-with-warnings

The BusinessPlan artifacts are structurally complete and sufficient to proceed to TechPlan. H1 (internet routing) and H2 (Postgres hostname) are real implementation risks but do not block TechPlan authoring — they should be resolved as the first two implementation stories. M3 (Hermes memory config key) should be resolved before the Postgres memory story is estimated.

**Actions required before TechPlan:**
- [ ] Run `lens-new-domain hermes` and seed the hermes constitution with minimum rules
- [ ] Run `lens-new-service hermes/agent`
- [ ] Re-home feature in governance and control repo docs
- [ ] Run cross-feature sensing (`bmad-lens-sensing`) for watchdog overlap check

**Actions required before dev implementation (not blocking TechPlan):**
- [ ] Confirm Postgres hostname from Kaylee/Temporal secrets
- [ ] Confirm internet routing from hermes VM VLAN
- [ ] Verify Hermes memory config key from upstream docs
- [ ] Pin Hermes version after install
