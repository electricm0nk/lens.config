---
permitted_tracks: [full, express, quickdev, hotfix-express, spike]
required_artifacts:
  planning:
    - business-plan
  dev:
    - stories
gate_mode: informational
sensing_gate_mode: informational
additional_review_participants: []
enforce_stories: true
enforce_review: true
---

# Service Constitution: hermes/agent

**Layer:** service
**Name:** agent
**Domain:** hermes
**Parent:** hermes domain constitution
**Ratified:** 2026-05-24
**Status:** active

---

## Preamble

This constitution governs features within the hermes/agent service. It inherits terminus and hermes domain constraints and adds service-specific governance for Python toolchain, Ollama wiring, and Postgres memory provisioning.

---

## Inheritance Validation Record

This service constitution is strictly additive. It does not modify or weaken any parent-level articles.

---

## Articles

### Article 1: Python 3.11 + uv Toolchain

**Rule:** All hermes/agent service features must use Python 3.11 managed via `uv` (Astral). The runtime Python version and the `uv` lockfile must be pinned in the feature repository. Other Python runtimes or package managers (pip, poetry, conda) must not be introduced without explicit approval.

**Rationale:** The hermes agent upstream package (`hermes-agent`) is tested against Python 3.11 and uses the `uv` tool for environment management. Standardising on this toolchain ensures reproducible installs and simplifies the Ansible provisioning playbooks.

**Evidence Required:** Architecture docs and implementation stories must confirm Python 3.11 and `uv`. CI or Ansible playbook tasks must pin the Python version.

**Gate:** informational
**Status:** active

---

### Article 2: Ansible-Based VM Provisioning

**Rule:** Installation of hermes/agent features onto terminus VMs must use Ansible playbooks checked into the terminus.infra repository. Ad-hoc shell provisioning without an Ansible playbook is not acceptable for production installation.

**Rationale:** Ansible playbooks provide reproducible, auditable, idempotent provisioning. Storing playbooks in terminus.infra keeps all VM configuration in the approved infrastructure repository and avoids configuration drift.

**Evidence Required:** Implementation stories must reference the Ansible playbook task(s) in terminus.infra. Architecture docs must name the target VM and the Ansible inventory group.

**Gate:** informational
**Status:** active

---

### Article 3: Hermes Config Isolation

**Rule:** Hermes agent configuration (`~/.hermes/config.yaml`) and credentials (`.env` files) must not be committed to any repository. Secrets must be injected via Vault-backed Ansible tasks or equivalent secret management. The config template must be committed; the populated config file must not be.

**Rationale:** Credential leakage via committed config files is a common attack vector. All runtime secrets must flow through approved secret management (Vault).

**Evidence Required:** Implementation stories must show that secrets are seeded from Vault into the VM via Ansible. No `.env` or `config.yaml` with populated secrets may appear in any committed artifact.

**Gate:** informational
**Status:** active
