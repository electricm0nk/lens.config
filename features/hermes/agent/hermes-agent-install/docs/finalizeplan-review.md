---
feature: terminus-inference-hermes-by-nous-research-install
doc_type: finalizeplan-review
phase: finalizeplan
verdict: pass-with-warnings
reviewer: lens-adversarial-review
source: phase-complete
date: 2026-05-15
critical_count: 0
high_count: 0
medium_count: 2
low_count: 2
---

# FinalizePlan Adversarial Review
**Feature:** terminus-inference-hermes-by-nous-research-install
**Phase:** finalizeplan | **Source:** phase-complete
**Date:** 2026-05-15

## Artifacts Reviewed

### Current Planning Set
- `docs/hermes/agent/hermes-agent-install/product-brief.md`
- `docs/hermes/agent/hermes-agent-install/research.md`
- `docs/hermes/agent/hermes-agent-install/brainstorm.md`
- `docs/hermes/agent/hermes-agent-install/prd.md`
- `docs/hermes/agent/hermes-agent-install/ux-design.md`
- `docs/hermes/agent/hermes-agent-install/architecture.md`

### Predecessor Reviews
- `docs/hermes/agent/hermes-agent-install/preplan-adversarial-review.md` (pass-with-warnings)
- `docs/hermes/agent/hermes-agent-install/businessplan-adversarial-review.md` (pass-with-warnings: 0C, 2H, 4M, 4L)
- `docs/hermes/agent/hermes-agent-install/techplan-adversarial-review.md` (pass-with-warnings: 0C, 2H, 3M, 3L)

### Discovery Log
- `docs/hermes/.todo/2026-05-15-hermes-vm-internet-routing.md` (status: open)
- `docs/hermes/.todo/2026-05-15-hermes-postgres-config-key.md` (status: open)
- `docs/hermes/.todo/2026-05-15-hermes-model-tag.md` (status: open)
- `docs/hermes/.todo/2026-05-15-hermes-tool-calling-verify.md` (status: open)

---

## Pre-Review Fixes Applied

| Finding | Operator Response | Action |
|---------|------------------|--------|
| F-H1 — VLAN internet egress | Confirmed: egress is allowed from hermes VM VLAN | `.todo/2026-05-15-hermes-vm-internet-routing.md` deleted (addressed). Architecture §5.2 pre-flight AC remains in S1 for explicit verification during provisioning. |
| F-M2 — Postgres config key | Key is in Vault; confirmed: create a new dedicated Postgres account | `.todo/2026-05-15-hermes-postgres-config-key.md` deleted (addressed). Story S2 must provision a new `hermes` role (not reuse an existing account). Vault path `secret/terminus/dev/postgres/hermes` confirmed. Software env-var discovery (what Hermes reads) deferred to Story S1 discovery AC. |
| F-M1 — .todo status update | (Applied during review session) | All four `.todo` entries updated from `open` to `reviewed` during TechPlan/FinalizePlan gate. Two entries now deleted (addressed). Two remain (`hermes-model-tag`, `hermes-tool-calling-verify`) in `reviewed` status. |

---

## Summary

The full planning artifact set is complete, internally consistent, and implementable. Architecture addresses all hard-gate requirements from the hermes/agent effective constitution (Org Articles 1-16). PRD goals G1-G7 are fully covered by architecture scope and the proposed S1-S6 story set. No critical blockers prevent the bundle generation step.

One high finding (F-H1) has been **resolved** by operator confirmation: VLAN internet egress is allowed from the hermes VM. The `.todo` entry has been deleted. The Postgres deferred-decision (F-M2 Postgres portion) is **resolved**: credentials are in Vault, and a dedicated `hermes` Postgres role will be created in Story S2. Two medium findings remain (F-M3: runbook/Molecule CI story; F-M4: CLI reachability AC) as story-level requirements for the bundle. Two low findings remain informational.

**Verdict: pass-with-warnings** — FinalizePlan bundle generation may proceed.

---

## Predecessor Review Reconciliation

### Findings from businessplan-adversarial-review.md

| ID | Finding | Prior Status | FinalizePlan Disposition |
|----|---------|-------------|--------------------------|
| BPH1 | VLAN internet egress | open | Carry-forward → **F-H1** |
| BPH2 | Hermes Postgres memory config key | open | Carry-forward → **F-M2** |
| BPM1 | VM provisioning gap | resolved | Resolved via cicd.md pattern in architecture §5.2 |
| BPM2 | Hermes CLI PATH management | documented | Story S1 AC required |
| BPM3 | Vault auth method ambiguity | documented | Story S1/S3 AC required |
| BPM4 | Molecule CI not wired | open | Carry-forward → **F-M3** |
| BPL1-L4 | Minor gaps | informational | Noted; no bundle blockers |

### Findings from techplan-adversarial-review.md

| ID | Finding | Prior Status | FinalizePlan Disposition |
|----|---------|-------------|--------------------------|
| TPH1 | Patroni pg_hba via patronictl | open | Story S2 AC must be explicit |
| TPH2 | VLAN internet egress | open | Carry-forward → **F-H1** |
| TPM1 | psql `\c` meta-command not Ansible-executable | open | Story S2 Ansible role implementation |
| TPM2 | AC-10 psql direct assertion needed | open | Story S5 test implementation |
| TPM3 | Molecule CI wiring | open | Carry-forward → **F-M3** |
| TPL1-L3 | Vault auth, PATH, depends_on | informational | L1 → **F-L1**; L2/L3 noted |

---

## Findings

### HIGH

#### F-H1 — VLAN Internet Egress ~~RESOLVED~~
**Resolution:** Operator confirmed — VLAN internet egress is allowed from the hermes VM network.
**Action taken:** `.todo/2026-05-15-hermes-vm-internet-routing.md` deleted (addressed per Org Art. 17).
**Residual story action:** Story S1 should still include a lightweight pre-flight AC to verify egress at provisioning time (e.g., `curl https://hermes-agent.nousresearch.com` from the new VM), as a belt-and-suspenders check. This is no longer a blocker.

**Status:** ✅ Resolved.

---

### MEDIUM

#### F-M1 — .todo Entries Require Phase-Gate Status Update to `reviewed`
**Artifact:** `docs/hermes/.todo/` (all 4 entries)
**Finding:** Org Art. 17 requires `.todo` entries to be reviewed at each phase gate. All four entries remain in `open` status. They were assessed in TechPlan review but the entry files themselves were not updated to `reviewed`. As a result, the domain `.todo` log does not reflect that these items have been through a phase gate review.

**Required Mitigation:**
Each `.todo` entry must be updated with `status: reviewed` and a reviewer note before the FinalizePlan bundle is committed. The entries should not be deleted (they remain open findings), but the status change confirms they passed through this gate.

**Status:** Open — must be completed before bundle commit.

---

#### F-M2 — Deferred Decisions: Model Tag and Tool Calling Remain; Postgres Confirmed
**Artifact:** `.todo/2026-05-15-hermes-model-tag.md`, `.todo/2026-05-15-hermes-tool-calling-verify.md`
**Update:** The Postgres connection key decision has been resolved: credentials are in Vault at `secret/terminus/dev/postgres/hermes`; a new dedicated `hermes` Postgres role will be created in Story S2 (not reusing an existing account). `.todo/2026-05-15-hermes-postgres-config-key.md` has been deleted.

Two deferred decisions remain:
1. **Model tag** — exact Ollama tag uncertain (`qwen3:27b` vs `qwen3.6:27b`). `.todo/2026-05-15-hermes-model-tag.md` remains `reviewed`.
2. **Tool calling support** — qwen3:27b function calling capability unverified. `.todo/2026-05-15-hermes-tool-calling-verify.md` remains `reviewed`.

Stories S3 (LLM config) and S5 (smoke test) depend on these resolved values. Both must be discovery ACs in Story S1.

**Required Mitigation:**
- Story S1 must include explicit ACs: verify Ollama model tag via `ollama list`; verify tool calling via a test prompt.
- Story S3 and S5 ACs must reference the values discovered in S1.

**Status:** Partially resolved (Postgres ✅). Model tag and tool calling remain open — must be Story S1 discovery ACs.

---

#### F-M3 — Runbook Story Not Explicit in S1-S6 Scope
**Artifact:** `architecture.md §9.1` (runbook commitment), `prd.md NFR-9.x`
**Finding:** Org Art. 11 requires a `runbook.md` as a dev-phase required artifact. The architecture commits to a runbook and identifies required sections (install, restart, Patroni pg_hba, Vault re-seed, LLM endpoint switch, log locations). However, no dedicated runbook story is listed in the architecture's story scope (S1-S6). Molecule CI wiring (carry-forward TPM3) is also unscoped.

**Required Mitigation:**
The sprint bundle must include at minimum:
- Story S7 (or equivalent): Author `runbook.md` covering all sections committed in architecture §9.1. AC: runbook.md present in docs path with all named sections.
- Story S8 (or equivalent): Wire Molecule scenarios to CI (GitHub Actions job in terminus.infra). AC: `ansible-test` or `molecule test` runs cleanly in CI on the hermes role set.

Alternatively, the runbook AC can be merged into Story S6 (final validation) if scope justifies it.

**Status:** Open — must be scoped into the sprint bundle before FinalizePlan completes.

---

#### F-M4 — Art. 18 Deployment Confirmation: Hermes CLI Reachability Not Explicitly Modeled
**Artifact:** `architecture.md §1.2`, `prd.md §Goals G3`
**Finding:** Org Art. 18 requires that a service initiative include deployment stories covering the endpoint being reachable. Hermes is a CLI tool (not a network-reachable HTTP service), so the Art. 18 test is: "is the operator able to reach and invoke `hermes` on the VM?" This is addressed implicitly in story S5/S6 (smoke tests), but there is no explicit story AC confirming: "From the operator's workstation, SSH to `hermes.trantor.internal` and run `hermes 'say hello'` successfully with an LLM response."

**Required Mitigation:**
Story S5 or S6 must include a deployment-confirmation AC: "SSH from operator workstation to `hermes.trantor.internal`, run `hermes` CLI, receive a valid response via `https://ollama.trantor.internal/v1`." This satisfies Art. 18's "can the operator reach the service" test for a CLI workload.

**Status:** Open — must be explicit in sprint story ACs.

---

### LOW

#### F-L1 — feature.yaml `depends_on` Remains Empty Despite Ollama Dependency
**Artifact:** `TargetProjects/lens/lens-governance/features/hermes/agent/hermes-agent-install/feature.yaml`
**Finding:** TechPlan review L3 identified that the Ollama server's `OLLAMA_CONTEXT_LENGTH >= 32768` configuration is a cross-feature dependency. `feature.yaml.depends_on` is still empty. If the Ollama server feature is later refactored or context length changes, this feature's S3/S5 stories could silently break.

**Required Mitigation:** Add a comment or entry to `feature.yaml.depends_on` noting the Ollama server context-length dependency, or at minimum add a note in the runbook's prerequisites section.

**Status:** Informational — update before dev sprint closes.

---

#### F-L2 — product-brief.md Predates TechPlan Decisions
**Artifact:** `docs/hermes/agent/hermes-agent-install/product-brief.md`
**Finding:** The product-brief was authored during PrePlan and predates key TechPlan decisions: OpenTofu VM creation, Patroni pg_hba approach, Synology DNS requirement. The brief's "Technical Assumptions" section may reference a pre-architecture view of provisioning. This does not block story generation but creates a documentation alignment gap for future readers.

**Required Mitigation:** No sprint action required. Note that product-brief reflects pre-architecture intent; architecture.md is the authoritative source for all implementation decisions.

**Status:** Informational.

---

## Party-Mode Blind-Spot Challenge

Three planning perspectives applied for the FinalizePlan challenge:

**Perturabo (Architect):** *"The planning set is coherent, but I'm looking at the dependency chain and see a sequencing problem. Story S1 provisions the VM. Stories S3 and S4 depend on config values (model tag, Postgres key) that require a running hermes binary to verify. But hermes can't run until S1 is complete — and the config values aren't discovered until S3/S4 try to configure. The sprint plan is implicitly assuming the operator will discover these values informally. That's a discovery task, not an acceptance criterion buried in S3. Where is the explicit 'run hermes, read its help, document the config keys' task?*"

**Inquisitor Greyfax (Analyst):** *"The PRD has `open_questions` still listed: VM spec (CPU/RAM/disk) is unresolved. The architecture says 'new Proxmox VM' but nowhere in the planning artifacts is a VM resource spec confirmed. OpenTofu creates the VM — what's in `hermes.tf`? What's the `memory`, `cores`, `disk_size`? If that's not specced, the infrastructure story (S1) cannot have a complete definition-of-done. An operator could provision a 1-core, 2GB VM and wonder why Hermes is slow or why qwen3:27b OOMs during inference. What is the approved VM spec?*"

**Watch-Captain Artemis (QA):** *"I see 12 BDD ACs mapped in architecture §8.2, but no test file structure specified. Where do the Molecule verify tasks live? Which directory? What's the naming convention? Without a declared test layout, each story writer will invent their own and you'll end up with test debt immediately. Also: the smoke test for AC-10 (Postgres persistence) requires a direct psql query — but does the Ansible molecule environment wire up a test Postgres? Or does the test assume a live Patroni cluster? If Molecule tests run in Docker, there's no Patroni in the test environment. This means AC-10 can't be integration-tested in Molecule — it needs a live-environment smoke test in Story S5. Is that explicit?*"

### Blind-Spot Challenge Questions

1. **VM resource spec gap**: The PRD still lists "What Proxmox VM spec is appropriate?" as an open question. Architecture and infra stories depend on this. What CPU/RAM/disk is approved for `hermes.tf`? This must be confirmed before Story S1 can be written with a complete DoD.

2. **Discovery task sequencing**: Deferred config decisions (model tag, Postgres env var, tool calling) need a discovery task in S1 or a pre-sprint spike. How will these values be documented before S3/S4 AC verification begins?

3. **Molecule test layout**: Where do Molecule test scenarios live (`molecule/default/`, `molecule/verify/`)? Is there a test role directory convention in `terminus.infra`? Without this, test stories lack a canonical path.

4. **AC-10 test environment**: Molecule Docker driver has no live Patroni. AC-10 (Postgres persistence) must be a live-environment smoke test — not a Molecule scenario. Is this split (Molecule for unit-level AC 1-9, live-env for AC-10-12) explicit in the test strategy?

5. **Synology DNS A-record**: Who provisions the DNS entry for `hermes.trantor.internal`? This is a Semaphore-style manual operation (Art. 19: external service config that can't be in a manifest). Is it a story task or is it expected to be done informally?

---

## Constitution Compliance Check (FinalizePlan)

| Article | Requirement | Status |
|---------|-------------|--------|
| Org Art.2 | Phase artifacts present before gate | ✅ all 6 review artifacts present |
| Org Art.3 | architecture.md present and non-empty | ✅ |
| Org Art.4 | No confidential data exfiltration | ✅ all inference internal |
| Org Art.7 | TDD strategy documented | ✅ (F-M3: Molecule CI story needed) |
| Org Art.8 | BDD GWT per story AC | **Bundle must include GWT in every story file** |
| Org Art.9 | Vault wiring, no committed secrets | ✅ architecture §5.3 |
| Org Art.11 | Runbook commitment | ✅ architecture §9.1 (F-M3: story needed) |
| Org Art.15 | AI safety section | ✅ architecture §9 |
| Org Art.16 | Internal infra primacy, .todo entries | ✅ 4 entries present |
| Org Art.17 | .todo entries reviewed at phase gate | ⚠ F-M1: entries need status update |
| Org Art.18 | Deployment includes service reachability | ⚠ F-M4: CLI reachability AC needed |
| Org Art.20 | Go default — Python exception justified | ✅ ADR in architecture §4 |
| hermes Art.1 | VM-only install | ✅ |
| hermes Art.2 | Local-first inference endpoint | ✅ |
| hermes Art.3 | Patroni HA Postgres VIP | ✅ |
| hermes/agent Art.1 | Python 3.11 + uv | ✅ |
| hermes/agent Art.2 | Ansible provisioning in terminus.infra | ✅ |
| hermes/agent Art.3 | Config isolation, Vault-backed | ✅ |

---

## Gate Decision

**Verdict: pass-with-warnings**
- 0 critical findings
- 0 high findings (F-H1 resolved: egress confirmed; `.todo` deleted)
- 2 medium findings (F-M3: runbook + Molecule CI story not scoped; F-M4: Art. 18 CLI reachability AC needed in S5/S6)
- 2 low findings (F-L1: feature.yaml depends_on gap; F-L2: product-brief staleness)

**FinalizePlan bundle generation may proceed.** F-H1 is resolved. F-M2 Postgres approach is confirmed (dedicated account from Vault). Remaining medium findings are addressed at story authoring time. The VM resource spec question (Greyfax blind-spot) must be confirmed before Story S1 DoD is complete — scope as an explicit story pre-condition in epics/stories.
