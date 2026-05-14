---
feature: terminus-inference-hermes-by-nous-research-install
doc_type: techplan-adversarial-review
phase: techplan
verdict: pass-with-warnings
reviewer: lens-adversarial-review
source: phase-complete
date: 2026-05-15
critical_count: 0
high_count: 2
medium_count: 3
low_count: 3
---

# TechPlan Adversarial Review
**Feature:** terminus-inference-hermes-by-nous-research-install
**Phase:** techplan | **Source:** phase-complete
**Date:** 2026-05-15

## Artifacts Reviewed

- `docs/hermes/agent/hermes-agent-install/architecture.md`

## Predecessor Context Reviewed

- `docs/hermes/agent/hermes-agent-install/prd.md`
- `docs/hermes/agent/hermes-agent-install/ux-design.md`
- `docs/hermes/agent/hermes-agent-install/businessplan-adversarial-review.md` (pass-with-warnings)
- `TargetProjects/lens/lens-governance/constitutions/org/constitution.md`
- `TargetProjects/lens/lens-governance/constitutions/hermes/constitution.md`
- `TargetProjects/lens/lens-governance/constitutions/hermes/agent/constitution.md`

---

## Summary

The TechPlan architecture document is structurally sound and covers all required hard-gate domains:
data-flow statements (Org Art.4), TDD/Molecule strategy (Org Art.7), BDD AC map (Org Art.8),
security / Vault wiring (Org Art.9), runbook commitment (Org Art.11), AI safety (Org Art.15),
and internal infra primacy (Org Art.16). No critical blockers. Two high-severity findings
require story-level mitigations before Dev phase closes. The VM provisioning gap (H1 from the
adversarial session) was resolved during the review: VM creation is OpenTofu (`terminus.infra/tofu/`),
Ansible handles OS bootstrap — consistent with the cicd.md Phase 2 established pattern.

**Verdict: pass-with-warnings** — TechPlan may advance to FinalizePlan.

---

## Findings

### HIGH

#### H1 — Patroni pg_hba Management Approach Undefined
**Artifact:** architecture.md §7.2
**Finding:** The architecture acknowledges that `pg_hba.conf` may require patching to allow connections from `hermes.trantor.internal`, but describes it as "may need manual patching" without specifying the Patroni-safe mechanism. The shared Patroni cluster serves Kaylee, Temporal, and other live services. Incorrect `pg_hba` manipulation (direct file edit bypassing Patroni DCS) can disrupt all dependent services.

**Required Mitigation (before S2 story closes):** Story S2 must explicitly specify the Patroni-safe approach: `patronictl edit-config` to update `pg_hba` entries via the DCS, not direct file editing. The runbook must document this procedure.

**Status:** Open — must be addressed in Story S2 and runbook.

---

#### H2 — VM VLAN Internet Egress Unconfirmed (carry-forward from businessplan H1)
**Artifact:** docs/hermes/.todo/2026-05-15-hermes-vm-internet-routing.md
**Finding:** The Hermes install script and Python/uv package downloads require public internet access from the hermes VM during provisioning. Whether the VLAN assigned to the hermes VM has outbound internet egress is not confirmed. If blocked, the primary install approach is invalid.

**Required Mitigation (before S1 story closes):** During OpenTofu VM creation, assign the hermes VM to a Proxmox VLAN/bridge with confirmed internet egress (or document the fallback bundle approach). Confirm via `curl https://hermes-agent.nousresearch.com` from the VM before running the install script.

**Status:** Open — must be resolved in Story S1. `.todo` entry registered.

---

### MEDIUM

#### M1 — psql Meta-Command in Ansible SQL Block
**Artifact:** architecture.md §7.1
**Finding:** The SQL provisioning block includes `\c hermes_memory` — a psql console meta-command that cannot be executed by Ansible's `community.postgresql.postgresql_query` module. Mixing psql console syntax with Ansible task design will cause role failures.

**Required Mitigation:** Story S2 Ansible role must use separate `community.postgresql.postgresql_query` tasks and `community.postgresql.postgresql_privs` for schema-level grants, not a single SQL block with `\c`.

**Status:** Open — must be corrected in hermes-postgres-provision role implementation.

---

#### M2 — AC-10 Memory Persistence Test Cannot Distinguish Postgres vs In-Memory Fallback
**Artifact:** architecture.md §8.2 (AC-10 row)
**Finding:** If the Postgres config key is wrong (see `.todo/2026-05-15-hermes-postgres-config-key.md`), Hermes silently falls back to in-memory or SQLite storage. A test that only asks Hermes "recall the fact" will pass even on in-memory storage. The test cannot confirm the memory actually came from Postgres.

**Required Mitigation:** `tests/smoke/test_memory_persistence.py` must include an explicit psql assertion step: after recording a fact, query `hermes_memory` directly via `psql` to confirm the row exists in the database before asking Hermes to recall it. Two-part test: DB write confirmed by psql, then Hermes recall confirms read path.

**Status:** Open — must be addressed in Story S4/S5 test implementation.

---

#### M3 — Molecule CI Integration Not Defined
**Artifact:** architecture.md §8.1
**Finding:** The architecture asserts "Molecule with docker driver can run in CI" but no CI pipeline step is defined. Org Art.7 requires a CI gate with test coverage. Without a defined CI step, the TDD claim is unverifiable at merge time.

**Required Mitigation:** FinalizePlan epic/story set must include a story for wiring Molecule to the CI pipeline (GitHub Actions workflow in `terminus.infra`). Can be a single workflow job that runs all hermes role Molecule scenarios.

**Status:** Open — scope into FinalizePlan stories.

---

### LOW

#### L1 — Vault Auth Method Ambiguous ("AppRole or token")
**Artifact:** architecture.md §5.3
**Finding:** Specifying "AppRole or token" introduces ambiguity. The existing proxmox-bootstrap role uses a specific established auth pattern. New Ansible roles must match the established pattern to avoid introducing a second auth mechanism.

**Required Mitigation:** Story S1/S3 implementation must use the same Vault auth mechanism already in use by `proxmox-bootstrap`. Architecture text updated to "follow established proxmox-bootstrap Vault auth pattern."

**Status:** Informational — resolve during implementation.

---

#### L2 — Hermes PATH Configuration Ownership Not Specified
**Artifact:** architecture.md §5 (hermes-agent-install role)
**Finding:** The install script may or may not add `hermes` to `$PATH`. The Ansible `hermes-agent-install` role must explicitly configure `.bashrc` or `.profile` for PATH after install to ensure `hermes` is available in non-interactive sessions.

**Required Mitigation:** `hermes-agent-install` role tasks must include an idempotent `.bashrc`/`.profile` PATH entry for `~/.hermes/bin` with a corresponding Molecule verify task.

**Status:** Informational — resolve during implementation.

---

#### L3 — Ollama Context Length Cross-Feature Dependency Not in feature.yaml
**Artifact:** architecture.md §6.1
**Finding:** `OLLAMA_CONTEXT_LENGTH >= 32768` is a dependency on the Ollama server configuration owned by a separate terminus initiative. It is not tracked in `feature.yaml.depends_on`.

**Required Mitigation:** Add a `depends_on` entry to `feature.yaml` noting the Ollama server context length requirement, or at minimum document it as a prerequisite verification step in the runbook.

**Status:** Informational — update feature.yaml or runbook before Dev sprint.

---

## Party-Mode Blind-Spot Challenge

Three planning perspectives were applied:

**Perturabo (Architect):** VM creation gap — resolved. cicd.md Phase 2 pattern confirms OpenTofu creates the VM; Ansible handles bootstrap. Architecture §5.2 has been updated accordingly. This eliminates the implementation ambiguity in Story S1.

**Inquisitor Greyfax (Analyst):** Patroni pg_hba ownership remains a gap (H1 above). The solo-operator acceptance is valid, but Story S2 AC must explicitly include: "Confirm hermes user can connect from hermes VM IP before declaring S2 done." This is a concrete, testable criterion.

**Watch-Captain Artemis (QA):** AC-10 test design gap (M2 above). Without a direct Postgres assertion, the persistence test cannot distinguish Postgres from in-memory fallback. Two-part test required.

### Blind-Spot Challenge Responses

| Question | Response |
|----------|----------|
| How is the VM actually created? | **Resolved** — OpenTofu (`terminus.infra/tofu/`), consistent with cicd.md pattern. Architecture §5.2 updated. |
| Patroni pg_hba approach defined? | **Open (H1)** — `patronictl edit-config` required; must be explicit in S2 AC and runbook. |
| Will S2 include standalone hermes-user connect smoke test? | **Open (H1)** — Yes, required before S4 can start. Story S2 AC must include this. |
| How does AC-10 confirm Postgres vs in-memory? | **Open (M2)** — Direct psql assertion required in test implementation. |
| Confirmed VLAN with internet egress for hermes VM? | **Open (H2)** — Unknown; must be resolved in Story S1. `.todo` entry registered. |

---

## Constitution Compliance Check

| Article | Requirement | Status |
|---------|-------------|--------|
| Org Art.3 | architecture.md exists and non-empty | ✅ |
| Org Art.4 | Explicit data-flow statements; no external exfiltration | ✅ |
| Org Art.7 | TDD/Molecule strategy documented | ✅ (M3: CI wiring deferred to FinalizePlan) |
| Org Art.8 | BDD AC map — all 12 ACs have test scenarios | ✅ (M2: AC-10 test design needs hardening) |
| Org Art.9 | Security: Vault wiring, no committed secrets, trust boundaries | ✅ |
| Org Art.11 | Runbook commitment documented | ✅ |
| Org Art.15 | AI safety section present; no unmitigated risks | ✅ |
| Org Art.16 | Internal infra primacy confirmed; no `.todo` deviation entries needed | ✅ |
| hermes Art.1 | VM-only install scope confirmed | ✅ |
| hermes Art.2 | Ollama `https://ollama.trantor.internal/v1` only | ✅ |
| hermes Art.3 | Postgres `10.0.0.56:5432/hermes_memory` | ✅ |
| hermes/agent Art.1 | Python 3.11 + uv, language justification present | ✅ |
| hermes/agent Art.2 | Ansible provisioning in terminus.infra | ✅ |
| hermes/agent Art.3 | Config isolation; secrets via Vault; no committed .env values | ✅ |

---

## Gate Decision

**Verdict: pass-with-warnings**
- 0 critical findings
- 2 high findings (H1: pg_hba patroni approach; H2: VLAN internet egress) — must be addressed at story level, not blocking TechPlan gate
- 3 medium findings — must be addressed during implementation
- 3 low findings — informational

**TechPlan may advance to FinalizePlan.** High and medium findings are scoped into story acceptance criteria and implementation notes. No critical blocker prevents the phase transition.
