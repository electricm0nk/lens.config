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

# Domain Constitution: hermes

**Layer:** domain
**Name:** hermes
**Parent:** terminus
**Ratified:** 2026-05-24
**Status:** active

---

## Preamble

This constitution governs all initiatives within the hermes domain. It inherits and preserves all org-level and terminus domain-level rules and adds domain-specific governance for agent workloads, VM installation scope, and AI data-sovereignty boundaries.

---

## Inheritance Validation Record

This domain constitution is strictly additive relative to the terminus domain constitution. It does not remove, narrow, weaken, or create exceptions to any org-level or terminus-level article. All articles below are domain-specific extensions only.

---

## Articles

### Article 1: VM-Only Installation Scope

**Rule:** Hermes-domain features install and run on terminus homelab VMs (bare-metal or hypervisor-hosted), not as workloads within the k3s cluster. A feature that requires k3s scheduling must document an explicit architectural justification. No hermes workload may be scheduled into the k3s cluster without a constitutional amendment or an approved architectural exception.

**Rationale:** Hermes agent workloads have direct host-level dependencies (GPU access, local storage paths, process isolation) that are better served by VM installation than by containerised scheduling. VM placement avoids k3s namespace overhead and keeps the agent environment reproducible via Ansible.

**Evidence Required:** Architecture artifacts must identify the target VM(s) and confirm that the installation method is Ansible (or equivalent) rather than Helm or k8s manifests. If k3s scheduling is proposed, the exception and justification must appear in the architecture doc before TechPlan gate.

**Gate:** informational
**Status:** active

---

### Article 2: Local-First Inference Endpoint

**Rule:** Hermes agent features must route all LLM inference to the terminus-internal Ollama endpoint (`https://ollama.trantor.internal/v1`). No hermes feature may introduce a configuration path that routes inference to an external cloud LLM API for confidential or sensitive data without an approved org Article 15 data-sovereignty exception. Non-sensitive research queries may optionally use external endpoints but must be explicitly flagged in the architecture doc.

**Rationale:** Hermes operates on an internal homelab and its primary value is local-first autonomous agent execution. Routing confidential data through external LLM APIs undermines the data-sovereignty guarantee and contradicts the purpose of hosting local inference. This article reinforces terminus Article 7 at the domain level.

**Evidence Required:** Architecture docs must name the inference endpoint and confirm it is `https://ollama.trantor.internal/v1` for any data classified as internal or confidential. Configuration templates and runbooks must not expose an external LLM API key by default.

**Gate:** informational
**Status:** active

---

### Article 3: Postgres Memory Backend via Patroni HA

**Rule:** When a hermes feature uses persistent memory storage backed by PostgreSQL, it must connect to the terminus Patroni HA cluster at `10.0.0.56:5432` (Patroni VIP). Connection strings must reference the VIP directly. Separate application-scoped databases and roles must be provisioned per feature — shared databases with cross-feature schema overlap are not permitted without explicit approval.

**Rationale:** The terminus Patroni cluster (pg-01 + pg-02, PostgreSQL 17) is the approved persistent storage authority. Using the Patroni VIP ensures automatic failover. Per-feature database isolation prevents schema conflicts and access-control leakage across hermes agent workloads.

**Evidence Required:** Architecture and implementation artifacts must include the Postgres connection string pattern (`postgresql://{role}:{password}@10.0.0.56:5432/{db_name}`), the provisioned database name, and the scoped role. The role must not have superuser privileges.

**Gate:** informational
**Status:** active
