---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11]
inputDocuments:
  - docs/hermes/agent/hermes-agent-install/prd.md
  - docs/hermes/agent/hermes-agent-install/ux-design.md
  - docs/hermes/agent/hermes-agent-install/businessplan-adversarial-review.md
  - TargetProjects/lens/lens-governance/constitutions/org/constitution.md
  - TargetProjects/lens/lens-governance/constitutions/hermes/constitution.md
  - TargetProjects/lens/lens-governance/constitutions/hermes/agent/constitution.md
workflowType: architecture
feature: terminus-inference-hermes-by-nous-research-install
domain: hermes
service: agent
track: full
updated_at: "2026-05-15T00:00:00Z"
---

# Architecture Decision Document — Hermes Agent Install

**Feature:** terminus-inference-hermes-by-nous-research-install
**Domain:** hermes | **Service:** agent
**Track:** full | **Phase:** techplan
**Author:** Todd Hintzmann
**Date:** 2026-05-15

---

## 1. System Overview

### 1.1 Purpose

This architecture document covers the installation and wiring of **Hermes Agent** (Nous Research, MIT license) on a dedicated Proxmox VM within `trantor.internal`. The goal is to provide a local-first, persistent autonomous AI agent capability that routes all inference through the existing `https://ollama.trantor.internal/v1` endpoint with a persistent Postgres memory backend.

No new terminus k3s service is introduced. This is a user-space VM install governed by the `hermes` domain. Accordingly, Terminus Art.12 (k3s completeness) is **not applicable**.

### 1.2 Architectural Scope

| Layer | Component | Notes |
|-------|-----------|-------|
| VM | `hermes.trantor.internal` | New Proxmox VM, user-space install |
| Application | Hermes Agent (Nous Research) | `~/.hermes/hermes-agent/`; CLI-only |
| Toolchain | Python 3.11 + `uv` | Hermes upstream requirement; see Language Justification §4 |
| LLM Backend | `https://ollama.trantor.internal/v1` | Existing endpoint, no auth, no change |
| Memory Backend | Postgres on Patroni VIP `10.0.0.56:5432` | `hermes_memory` DB, `hermes` user |
| Secrets | Vault (`secret/terminus/dev/postgres/hermes`) | Ansible-managed; no committed credentials |
| Provisioning | Ansible playbooks in `terminus.infra` | Follows existing proxmox-bootstrap pattern |

---

## 2. Architecture Diagram

```
 ┌────────────────────────────────────────────────────────────────────┐
 │                        trantor.internal                            │
 │                                                                    │
 │  ┌─────────────────────────┐                                       │
 │  │   hermes.trantor.internal │                                     │
 │  │   (Proxmox VM)            │                                     │
 │  │                           │                                     │
 │  │  ~/.hermes/hermes-agent/  │                                     │
 │  │  hermes CLI (Python 3.11) │                                     │
 │  │         │                 │                                     │
 │  │         │ HTTPS           │                                     │
 │  │         ▼                 │                                     │
 │  │  ~/.hermes/config.yaml    │ ──────────────────────────────────► │  ollama.trantor.internal
 │  │  ~/.hermes/.env (600)     │   model.base_url (no external)      │  (existing Traefik IngressRoute)
 │  │         │                 │                                     │        │
 │  │         │ PostgreSQL wire │                                     │        ▼
 │  │         ▼                 │                                     │  Ollama (qwen3:27b)
 │  │  10.0.0.56:5432           │                                     │  GPU workers
 │  │  hermes_memory DB         │                                     │
 │  │  (Patroni HA VIP)         │                                     │
 │  └─────────────────────────┘                                       │
 │                                                                    │
 │  ┌─────────────────────────┐                                       │
 │  │   Vault                 │                                       │
 │  │  secret/terminus/dev/   │                                       │
 │  │  postgres/hermes        │◄── Ansible reads on provisioning      │
 │  └─────────────────────────┘                                       │
 └────────────────────────────────────────────────────────────────────┘

 ╳ No external LLM traffic  (org Art.4, hermes Art.2, terminus Art.7 compliance)
```

---

## 3. Data Flow and Confidentiality (Org Art. 4)

### 3.1 Inference Traffic

All LLM inference initiated by Hermes Agent routes exclusively to `https://ollama.trantor.internal/v1`. This endpoint resolves within `trantor.internal` via internal DNS/Traefik. There is no fallback to any external LLM provider. No API keys for OpenAI, Anthropic, Google, Cohere, or any other external provider will be present in the configuration.

**Evidence at runtime:** `~/.hermes/config.yaml` will contain only a `custom` provider block with `base_url: https://ollama.trantor.internal/v1`. Any config audit (AC-4) confirming absence of external keys is a hard acceptance criterion.

### 3.2 Memory Traffic

Session memory is written to the Postgres `hermes_memory` database on the Patroni VIP at `10.0.0.56:5432`. This endpoint is within `trantor.internal` and does not cross any network boundary outside the internal cluster.

### 3.3 Installation Traffic

The upstream Hermes install script (`curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash`) and Python/uv package installation require **temporary public internet access from the hermes VM** during provisioning only. This is explicitly scoped to the install phase and does not affect inference or memory paths.

> **H1 — Open Finding:** The hermes VM VLAN routing to the public internet is not yet confirmed. This must be verified during VM provisioning. If VLAN-level internet egress is blocked (secure-enclave posture), an alternative is to pre-download the install bundle and Python packages to an internal mirror before VM provisioning. This finding is registered in `docs/hermes/.todo/` per Org Art. 17 — see §10 Discovery Log.

### 3.4 Summary: Data Exfiltration Boundary

| Data Type | Destination | External? |
|-----------|-------------|-----------|
| LLM inference prompts / completions | `ollama.trantor.internal` | ❌ No |
| Postgres memory writes/reads | `10.0.0.56:5432` | ❌ No |
| Secrets (Vault read) | `vault.trantor.internal` | ❌ No |
| Install packages (one-time provisioning) | public internet | ✅ Yes — scoped to install phase only; no sensitive data; H1 open |

**Statement:** No confidential or personal data is transmitted outside `trantor.internal`. All inference and memory traffic is internal. Provisioning-time package installation carries no sensitive data.

---

## 4. Language Justification (Terminus Inference Art. 7 / Org Art. 7)

### 4.1 Python as Application Language

Hermes Agent is a **third-party MIT-licensed application** authored by Nous Research. It is installed from an upstream release artifact, not authored in this repository. The `terminus/inference` service constitution (Inference Art.7) prefers Go for new terminus services. This article is **not applicable** here because:

1. We are not authoring a new terminus-inference service. We are installing an existing application.
2. There is no meaningful alternative language: the application is Python-only upstream.
3. Rewriting Hermes Agent in Go is explicitly out of scope.

**Declared exception:** Python 3.11 is accepted as the implementation language per the upstream application requirement. This exception is bounded to the Hermes Agent application itself (`~/.hermes/hermes-agent/`). Any tooling authored within this initiative (Ansible playbooks, provisioning scripts, test scaffolds) follows the existing terminus provisioning stack (Ansible/YAML/shell), not Python.

### 4.2 TDD Strategy for Python (Org Art. 7)

Org Art. 7 mandates red-green TDD discipline for all production code. For this initiative:

- **Hermes Agent itself** is third-party upstream code; we do not author it and therefore cannot apply TDD to it.
- **Ansible playbooks** are tested via **Ansible Molecule** (see §8 Testing Strategy).
- **Any Python helper scripts** authored for provisioning or validation will follow pytest red-green discipline: failing test committed first, then minimum passing implementation.
- **Acceptance tests** (smoke test, Postgres connectivity check) are implemented as Molecule verify tasks or pytest scenarios (see §8).

---

## 5. VM Provisioning Design (hermes/agent Art. 2)

### 5.1 Target VM

| Attribute | Value |
|-----------|-------|
| Hostname | `hermes.trantor.internal` |
| Proxmox Node | Worker node with VLAN access to `trantor.internal` |
| vCPU | 4 cores |
| RAM | 8 GB |
| Root Disk | 40 GB (SSD-backed) |
| OS | Ubuntu 22.04 LTS (standard terminus base) |
| Role | User-space app server; no k3s agent |

> **VM Spec Rationale:** Python 3.11 + uv workspace for Hermes requires minimal CPU (Hermes is not the inference compute source; GPU workers are separate). 4 vCPU / 8 GB provides headroom for parallel sessions and future hermes service expansion. 40 GB covers OS + Python env + uv cache + Hermes workspace.

### 5.2 Provisioning Structure (OpenTofu → Ansible, per cicd.md pattern)

**VM creation is OpenTofu, not Ansible.** This follows the established cicd.md Phase 2 / Step P2-1 pattern: `tofu apply` in `terminus.infra/tofu/` creates the Proxmox QEMU VM. Ansible only runs **after** the VM exists — for OS bootstrap, package install, and app configuration. This matches how `k3s-cp`, `k3s-worker`, `pg-01`, and `pg-02` VMs are created in the terminus stack.

**OpenTofu changes** in `terminus.infra/tofu/`:
```
terminus.infra/
  tofu/
    environments/proxmox/
      hermes.tf             # community.general.proxmox_kvm resource: hermes.trantor.internal
                            # 4 vCPU, 8 GB RAM, 40 GB SSD, cloud-init user/SSH-key injection
                            # Vault-sourced Proxmox API token (existing proxmox-bootstrap pattern)
```

`tofu apply` from `terminus.infra/tofu/environments/proxmox/` creates the VM. After apply, `hermes.trantor.internal` is reachable via SSH and ready for Ansible.

**Ansible changes** in `terminus.infra/ansible/`:
```
terminus.infra/
  ansible/
    inventories/
      hosts.yml                     # Add hermes.trantor.internal entry
    playbooks/
      bootstrap-hermes.yml          # Post-VM bootstrap: OS packages, uv, Python 3.11
      install-hermes-agent.yml      # Hermes install, config seeding, Postgres wiring
    roles/
      hermes-bootstrap/             # VM baseline: OS packages, uv, Python 3.11
        tasks/main.yml
        defaults/main.yml
      hermes-agent-install/         # Hermes install script execution, config templating
        tasks/main.yml
        templates/
          config.yaml.j2            # ~/.hermes/config.yaml template
          env.j2                    # ~/.hermes/.env template (Vault-seeded)
        defaults/main.yml
      hermes-postgres-provision/    # DB creation, user grant (runs against Postgres host)
        tasks/main.yml
        defaults/main.yml
```

**Story S1 sequence (per cicd.md pattern):**
1. Add `hermes.tf` to `terminus.infra/tofu/environments/proxmox/` → `tofu plan` → `tofu apply` → VM created
2. Add DNS entry on Synology NAS: `hermes.trantor.internal → <static VM IP>` (explicit entry required; this is a VM with its own IP, not a Traefik-routed service)
3. Run `bootstrap-hermes.yml` against the new VM
4. Verify `ssh user@hermes.trantor.internal` works and `python3.11 --version` returns

### 5.3 Vault Integration

The Ansible bootstrap pattern (established in `proxmox-bootstrap`) is used:

1. Ansible connects to Vault at `vault.trantor.internal` using `vault_auth_role` (AppRole or token).
2. Hermes Postgres password is retrieved from `secret/terminus/dev/postgres/hermes`.
3. The password is injected into `~/.hermes/.env` via the Jinja2 template; `no_log: true` is enforced on all Vault reads.
4. The `.env` file is written with `mode: '0600'` and owned by the operator user.

**No Vault path for Ollama** — the Ollama endpoint has no auth layer; `~/.hermes/config.yaml` will contain only the base URL and model name.

---

## 6. LLM Backend Configuration (hermes Art. 2)

### 6.1 Config Template

`~/.hermes/config.yaml` (provisioned via Ansible template):

```yaml
model:
  provider: custom
  base_url: https://ollama.trantor.internal/v1
  default: qwen3:27b
  context_length: 32768
  # No API key configured — Ollama has no auth layer on trantor.internal
```

**Notes:**
- `model.default: qwen3:27b` — the model naming must be verified against the running Ollama server during Dev. The terminal history suggests `qwen3.6:27b` may be the exact tag; the Ansible template will be parameterized as `hermes_model_default` to allow override without modifying the template.
- `context_length: 32768` — assumes the Ollama server runs with `OLLAMA_CONTEXT_LENGTH >= 32768`. This must be confirmed in the smoke test story (AC-5).

### 6.2 Config `.env` Template

`~/.hermes/.env` (provisioned via Ansible, Vault-seeded):

```bash
# Hermes Postgres memory backend — populated by Ansible from Vault
HERMES_DB_URL=postgresql://hermes:{{ hermes_postgres_password }}@10.0.0.56:5432/hermes_memory
```

> **Discovery Note:** The exact Hermes config key for the Postgres memory backend is not confirmed from upstream documentation. The key `HERMES_DB_URL` is a reasonable assumption based on common patterns; the actual key must be verified against `~/.hermes/hermes-agent/` source or upstream docs during the Dev phase. This finding is registered in §10. If the key differs, the `.env` template is the only change required.

---

## 7. Postgres Memory Backend (hermes Art. 3)

### 7.1 Database Provisioning

The `hermes-postgres-provision` Ansible role will execute the following against the Patroni VIP `10.0.0.56:5432` using a Vault-sourced Postgres admin credential:

```sql
-- Create dedicated database
CREATE DATABASE hermes_memory;

-- Create scoped user
CREATE USER hermes WITH PASSWORD '<vault-sourced>';

-- Scoped grant
GRANT ALL PRIVILEGES ON DATABASE hermes_memory TO hermes;

-- Default schema privileges (PostgreSQL 15+)
\c hermes_memory
GRANT ALL ON SCHEMA public TO hermes;
```

### 7.2 Connectivity Requirements

- Postgres (`pg_hba.conf`) must allow connections from `hermes.trantor.internal` (or its IP) for the `hermes` user to `hermes_memory`. The Ansible role will document the required `pg_hba.conf` entry in the runbook; manual patching of `pg_hba.conf` may be needed if the Patroni cluster does not expose a configuration API.
- Port 5432 must be reachable from the hermes VM. If a network policy or firewall blocks this, the infra team must add an allow rule.

### 7.3 Isolation

The `hermes` Postgres user is scoped exclusively to `hermes_memory`. It has no SUPERUSER, CREATEDB, or CREATEROLE privileges. Cross-database access is not permitted (AC-11).

---

## 8. Testing Strategy (Org Art. 7 + Art. 8)

### 8.1 Ansible Molecule Testing

All Ansible roles in this initiative will be tested via **Ansible Molecule** with the `docker` driver (consistent with terminus.infra test patterns). The TDD cycle:

1. **Red:** Write Molecule verify task asserting the expected state (e.g., `hermes --version` exits 0, `~/.hermes/.env` has `600` permissions, Postgres connection succeeds).
2. **Green:** Implement the role task until verify passes.
3. **Refactor:** Clean up the role without breaking verify.

Molecule scenarios per role:
- `hermes-bootstrap/molecule/default/` — Python 3.11 + uv available; PATH configured
- `hermes-agent-install/molecule/default/` — `hermes --version` exits 0; `config.yaml` has correct keys; `.env` is `600`
- `hermes-postgres-provision/molecule/default/` — DB `hermes_memory` exists; `hermes` user cannot connect to other DBs

### 8.2 BDD Acceptance Tests (Org Art. 8)

Each PRD acceptance criterion maps to at least one implemented test scenario. Tests are **not stubs** — they exercise real production behavior:

| AC | BDD Scenario | Test Type | Location |
|----|-------------|-----------|---------|
| AC-1 | `hermes --version` returns without error | Molecule verify | `hermes-agent-install/molecule/default/verify.yml` |
| AC-2 | `config.yaml` contains all required keys, no cloud keys | Molecule verify | `hermes-agent-install/molecule/default/verify.yml` |
| AC-3 | `hermes chat "Say hello"` returns HTTP 200 from Ollama | Integration smoke | `tests/smoke/test_hermes_chat.py` |
| AC-4 | No external provider keys in config | Ansible assert task | `hermes-agent-install/tasks/audit-config.yml` |
| AC-5 | 8k+ token prompt returns coherent non-truncated response | Integration smoke | `tests/smoke/test_context_length.py` |
| AC-6 | Agentic file-write task produces tool call invocation | Integration smoke | `tests/smoke/test_agentic_task.py` |
| AC-7 | `~/.hermes/.env` permissions are `600` | Molecule verify | `hermes-agent-install/molecule/default/verify.yml` |
| AC-8 | Proxmox VM reachable via SSH; Ansible playbook in git | CI check | Pipeline lint + SSH reachability probe |
| AC-9 | `runbook.md` published and contains required sections | File existence check | `tests/docs/test_runbook_structure.py` |
| AC-10 | Memory persists across process restart | Integration smoke | `tests/smoke/test_memory_persistence.py` |
| AC-11 | `hermes` user cannot connect to other DBs | Molecule verify | `hermes-postgres-provision/molecule/default/verify.yml` |
| AC-12 | `hermes` domain in governance | Lens lifecycle gate | Already satisfied (domain scaffolded pre-TechPlan) |

> **Note on integration smoke tests:** AC-3, AC-5, AC-6, AC-10 require a live `hermes.trantor.internal` VM and a live Ollama endpoint. These are manual + semi-automated — they run as pytest scripts on the hermes VM as the final acceptance gate, not in CI unit test runs.

---

## 9. Security Architecture (Org Art. 9)

### 9.1 Secret Handling

| Secret | Location | Access |
|--------|----------|--------|
| Hermes Postgres password | Vault `secret/terminus/dev/postgres/hermes` | Ansible reads on provision; written to `~/.hermes/.env` only |
| Postgres admin credential | Vault `secret/terminus/dev/postgres/admin` | Ansible reads on provision; `no_log: true` |
| Vault AppRole credentials | Ansible inventory vault-encrypted vars | Not committed in plaintext |
| Ollama endpoint | `~/.hermes/config.yaml` (URL only, no key) | No secret; Ollama has no auth layer |

**No secrets are committed to any repository.** All credential injection is performed at provision time via Ansible + Vault.

### 9.2 Trust Boundaries

```
hermes VM (untrusted app user)
  → Vault (AppRole auth, narrowly scoped policy)
  → Postgres (hermes user, hermes_memory DB only)
  → Ollama endpoint (no auth — internal network boundary is the control)
```

### 9.3 AI Safety (Org Art. 15)

Hermes Agent is a locally-hosted application with no external AI provider. The following safety measures are in place:

- **Data sovereignty:** All inference stays within `trantor.internal`; no external LLM API keys are configured.
- **No credential exfiltration:** Hermes can execute shell commands on the hermes VM. It cannot access Vault credentials (those are written to `.env` at provision time; Hermes only reads `HERMES_DB_URL`, not the Vault path). The VM is isolated to a non-privileged user on `hermes.trantor.internal`.
- **Scope isolation:** Hermes Agent runs on a dedicated VM isolated from k3s nodes, Postgres cluster admin, and Vault. Its blast radius is bounded to its own user-space on the hermes VM.
- **No known unmitigated AI safety risks** requiring an exception artifact per Org Art. 15. The application is not deployed as a daemon or service; it runs interactively under operator supervision.

---

## 10. Internal Infrastructure Primacy (Org Art. 16)

All infrastructure dependencies use the internal terminus stack:

| Dependency | Internal Equivalent | Used |
|------------|---------------------|------|
| AI inference | Ollama on `ollama.trantor.internal` | ✅ |
| Memory persistence | Patroni Postgres on `10.0.0.56:5432` | ✅ |
| Secrets management | Vault on `vault.trantor.internal` | ✅ |
| VM compute | Proxmox (`trantor.internal`) | ✅ |
| Provisioning | Ansible in `terminus.infra` | ✅ |

**No external services.** No `.todo` deviation entry required for this initiative.

---

## 11. Runbook Commitment (Org Art. 11)

A `runbook.md` will be authored as part of the Dev phase and committed to `docs/hermes/agent/hermes-agent-install/runbook.md`. It will cover:

- Component purpose and dependencies
- VM provisioning: Ansible playbook command sequence
- Hermes install: step-by-step from `curl` to `hermes --version`
- Config templates: `config.yaml` and `.env` with placeholder values
- Postgres setup: repeatable SQL (CREATE DATABASE, CREATE USER, GRANT)
- Smoke test commands: chat test, agentic task test, memory persistence test
- Context length verification: Ollama `OLLAMA_CONTEXT_LENGTH` check
- Troubleshooting: common failure modes and remediation
- Rollback: `rm -rf ~/.hermes/` and Postgres user/DB drop
- `Last Verified` timestamp (to be updated after every session touching the component)
- Language Justification section (Python as upstream app, not new terminus service)

---

## 12. Discovery Log / .todo Entries (Org Art. 17)

The following open findings were identified during TechPlan and must be registered as `.todo` entries in `docs/hermes/.todo/` before TechPlan gates:

| Slug | Label | Finding |
|------|-------|---------|
| `2026-05-15-hermes-vm-internet-routing.md` | `infra-deviation` | H1: Internet routing from hermes VM VLAN to public internet is unconfirmed. Required for Hermes install script + pip packages. If blocked, an internal mirror or pre-staged bundle approach must be designed. Source: hermes-agent-install. Status: open |
| `2026-05-15-hermes-postgres-config-key.md` | `deferred-decision` | The exact Hermes config key for the Postgres memory backend URL is not confirmed from upstream docs. `HERMES_DB_URL` is assumed; must be verified against hermes-agent source before implementing `env.j2` template. Source: hermes-agent-install. Status: open |
| `2026-05-15-hermes-model-tag.md` | `deferred-decision` | Exact Ollama model tag for `qwen3:27b` must be verified via `ollama ls` on the Ollama server. May be `qwen3:27b` or `qwen3.6:27b`. Ansible template parameterized as `hermes_model_default`. Source: hermes-agent-install. Status: open |
| `2026-05-15-hermes-tool-calling-verify.md` | `deferred-decision` | `qwen3:27b` function calling support must be confirmed via `ollama show qwen3:27b` before the agentic smoke test. If unsupported, an alternative model must be selected. Source: hermes-agent-install. Status: open |

---

## 13. Story Scope Guidance (Org Art. 8 / FinalizePlan)

The following stories are anticipated for the Dev phase. Exact story files will be produced in FinalizePlan:

| Story | Scope |
|-------|-------|
| S1: Provision hermes VM | Ansible playbook: `provision-hermes-vm.yml`; VM reachable via SSH; inventory entry committed |
| S2: Provision hermes_memory Postgres DB | Ansible role: `hermes-postgres-provision`; DB created, user scoped; SQL in runbook |
| S3: Install Hermes Agent + configure LLM backend | Ansible roles: `hermes-bootstrap` + `hermes-agent-install`; `hermes --version` ✓; `config.yaml` ✓ |
| S4: Wire Postgres memory backend | `.env` seeded via Vault; memory persistence verified across restart |
| S5: Smoke test + agentic validation | Chat smoke, agentic task, context length, config audit |
| S6: Runbook authoring + `docs/hermes/.todo/` review | `runbook.md` published; all .todo entries reviewed; Language Justification section complete |

---

## 14. Architectural Decision Records

### ADR-1: VM Install vs. k3s Deployment

**Decision:** Hermes Agent is installed as a user-space application on a dedicated Proxmox VM, not deployed to k3s.

**Rationale:** Hermes Agent is designed for interactive terminal use by an operator. It is not a long-running service serving external traffic. k3s deployment would require a daemon mode, persistent volumes, and an ingress route — none of which add value for the CLI agentic use case. Hermes constitution Art.1 explicitly restricts this feature to VM-only install scope.

**Consequence:** No Traefik IngressRoute, no ArgoCD app, no Helm chart. Provisioning is fully Ansible-managed.

### ADR-2: Direct Ollama Endpoint vs. Inference Gateway

**Decision:** Hermes wires directly to `https://ollama.trantor.internal/v1`, bypassing the terminus-inference-gateway.

**Rationale:** The inference-gateway has no IngressRoute configured (`ClusterIP` only). Adding an IngressRoute to route Hermes through the gateway is a separate initiative. The direct Ollama endpoint is already live and satisfies all requirements for this feature.

**Consequence:** Hermes cannot benefit from gateway-level auth, routing, or observability. This is acceptable for a solo-operator interactive tool and can be revisited in a follow-on feature.

### ADR-3: Patroni VIP vs. DNS Hostname for Postgres

**Decision:** Postgres connection uses the Patroni VIP `10.0.0.56:5432` (direct IP), not a DNS hostname.

**Rationale:** All other terminus services use the direct VIP (confirmed from live Kaylee `DATABASE_URL` secret). No DNS hostname for the Postgres VIP exists in the current trantor.internal configuration. Using the VIP directly is consistent and validated.

**Consequence:** If the Patroni VIP changes, all connection strings must be updated. This is acceptable given the static homelab topology and is documented in the runbook.

### ADR-4: Ansible as Provisioning Tool

**Decision:** VM provisioning and Hermes configuration are implemented via Ansible playbooks in `terminus.infra`.

**Rationale:** Ansible is the established provisioning tool for `trantor.internal` infrastructure (existing `proxmox-bootstrap`, `k3s-agent`, `k3s-server` roles). Using Ansible preserves the source-of-truth discipline required by Org Art. 11 and hermes/agent constitution Art. 2.

**Consequence:** Ansible must be available on the control machine. Vault credentials for proxmox and postgres must be accessible to the operator running the playbook.
