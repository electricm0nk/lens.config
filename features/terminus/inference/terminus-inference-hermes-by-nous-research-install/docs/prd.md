---
feature: terminus-inference-hermes-by-nous-research-install
doc_type: prd
status: draft
goal: "Install Hermes Agent on a dedicated VM within trantor.internal, wired to Ollama's OpenAI-compatible endpoint with persistent Postgres memory, providing a local-first autonomous AI agent capability."
key_decisions:
  - "Endpoint: https://ollama.trantor.internal/v1 — direct Ollama, NOT inference-gateway (no IngressRoute exists for gateway)"
  - "No Vault auth required — Ollama has no auth layer"
  - "Install target: new dedicated Proxmox VM inside trantor.internal"
  - "k3s deployment of Hermes is out of scope — VM install only"
  - "Domain Art.12 (k3s completeness) NOT applicable — no new k3s service endpoint"
  - "Python language exception accepted — Hermes is a third-party MIT-licensed application, not a new authored service"
  - "Postgres memory backend: in scope — provision hermes_memory DB on existing Postgres server"
  - "GOVERNANCE RESTRUCTURE PENDING: hermes established as top-level domain (peer of terminus); feature will be re-homed to hermes/agent once domain is scaffolded via lens-new-domain"
open_questions:
  - "What Proxmox VM spec is appropriate? (CPU cores, RAM, disk)"
  - "Does qwen3:27b template include function calling? Needs verification via 'ollama show qwen3:27b'"
  - "Should Hermes be installed from the upstream install script or cloned from a pinned commit?"
  - "What is the exact Hermes config key for Postgres memory backend? (needs verification against Hermes docs)"
  - "What Postgres hostname/port is used within trantor.internal? (postgres.trantor.internal:5432 assumed)"
depends_on: []
blocks: []
stepsCompleted: [step-01-init, step-02-discovery, step-02b-vision, step-02c-executive-summary, step-03-success, step-04-journeys, step-05-domain, step-06-innovation, step-07-project-type, step-08-scoping, step-09-functional, step-10-nonfunctional, step-11-polish, step-12-complete]
inputDocuments:
  - docs/terminus/inference/terminus-inference-hermes-by-nous-research-install/product-brief.md
  - docs/terminus/inference/terminus-inference-hermes-by-nous-research-install/research.md
  - docs/terminus/inference/terminus-inference-hermes-by-nous-research-install/brainstorm.md
  - docs/terminus/inference/terminus-inference-hermes-by-nous-research-install/preplan-adversarial-review.md
workflowType: prd
updated_at: "2026-05-14T00:00:00Z"
---

# Product Requirements Document — Hermes Agent Install

**Feature:** terminus-inference-hermes-by-nous-research-install
**Author:** Todd Hintzmann
**Date:** 2026-05-14
**Track:** Full | **Domain:** hermes *(top-level, pending governance scaffold)* | **Service:** agent

> **Governance Note:** `hermes` is being established as a top-level domain (peer of `terminus`). This feature is currently tracked under `terminus/inference` in governance. A domain rename operation (`lens-new-domain hermes` + `lens-new-service hermes/agent` + feature re-home) is required before TechPlan. The PRD reflects the target domain structure.

---

## Executive Summary

The terminus platform has a working local AI inference stack (Ollama on `trantor.internal`, Go inference-gateway exposing an OpenAI-compatible API) but no autonomous agent application wired to it. Engineers and operators who want AI assistance must use cloud providers — violating the local-first data sovereignty requirement — or interact with the inference endpoint manually.

This feature installs **Hermes Agent** (MIT-licensed autonomous agent app by Nous Research) on a dedicated VM within `trantor.internal`, configured to use `https://ollama.trantor.internal/v1` as its LLM backend with a **persistent Postgres memory backend**. Hermes provides a terminal-driven agentic CLI that can perform multi-step tasks (file operations, shell commands, web research, code generation) using locally hosted models, with memory of prior interactions persisting across sessions.

**Scope:** VM install + Postgres memory backend. The Ollama `trantor.internal` endpoint is already live. No Vault token is required — Ollama has no auth layer. A dedicated database (`hermes_memory`) will be provisioned on the existing Postgres server. k3s deployment, messaging platform integration (Telegram/Discord), and routing through the terminus-inference-gateway are out of scope and designated as follow-on work.

**Domain:** This feature establishes the `hermes` domain as a first-class governance entity within the platform. `hermes/agent` is the first service registered under it.

---

## Problem Statement

### Current Gaps

1. **No local agent capability.** There is no persistent AI assistant configured to run against the terminus inference stack. All AI agent use today routes through external cloud providers.

2. **Domain Art.7 compliance gap.** Cloud-routed AI workflows cannot be used for any confidential or operational data — this covers virtually all practical use cases within the terminus homelab.

3. **Manual inference interaction only.** The inference gateway and Ollama endpoint can only be called via `curl` or custom scripts. There is no agentic interface that can plan, execute multi-step tasks, and remember context across sessions.

4. **Hermes Agent is already purpose-built.** Nous Research's Hermes Agent is an MIT-licensed application specifically designed for this role. Building an equivalent from scratch is out of scope.

---

## Goals

| ID | Goal | Priority |
|----|------|----------|
| G1 | Install Hermes Agent on a dedicated VM within `trantor.internal` | P0 |
| G2 | Configure the `custom` provider pointing to `https://ollama.trantor.internal/v1` | P0 |
| G3 | Validate end-to-end: chat and tool calling work against the local endpoint | P0 |
| G4 | Document the configuration so any operator can replicate the setup | P1 |
| G5 | Enforce local-first AI data sovereignty — zero external LLM provider keys configured | P0 |
| G6 | Provision the dedicated Proxmox VM as a prerequisite story | P0 |
| G7 | Provision Postgres memory backend (`hermes_memory` DB) for persistent agent memory | P0 |
| G8 | Establish `hermes` as a top-level governance domain with `agent` as its first service | P1 |

---

## Success Criteria

### User Success

| # | Criterion | Measurable Signal |
|---|-----------|-------------------|
| U1 | An operator can install Hermes Agent on the dedicated VM in under 20 minutes from a clean base | `hermes --version` returns without error after running install script |
| U2 | An operator can have a full agentic conversation (not just chat) with `qwen3:27b` locally | `hermes` with a file-write task completes successfully and the file is created |
| U3 | No setup step requires a cloud provider account or API key | Configuration check: `~/.hermes/config.yaml` and `~/.hermes/.env` contain no cloud provider keys |
| U4 | Agent memory persists across sessions | Start session → record interaction → stop Hermes → restart → confirm prior context recalled from Postgres |

### Technical Success

| # | Criterion | Measurable Signal |
|---|-----------|-------------------|
| T1 | Hermes connects to `https://ollama.trantor.internal/v1` without errors | Smoke-test chat request returns valid JSON response with `status 200` |
| T2 | Context window is sufficient for Hermes system prompt + tool schemas | A prompt exceeding 8k tokens returns a coherent non-truncated response |
| T3 | Tool calling is functional for the configured model | `hermes` agentic task produces a tool call invocation visible in logs |
| T4 | Credential files secured at rest | `~/.hermes/.env` permissions are `600` or stricter |
| T5 | Postgres memory backend connected | Hermes config points to `hermes_memory` DB; no connection errors on startup |
| T6 | Memory survives process restart | Session data written to Postgres is retrievable after `hermes` process killed and restarted |

### Platform Success

| # | Criterion | Measurable Signal |
|---|-----------|-------------------|
| P1 | Config pattern documented for replication by additional operators | `runbook.md` published and peer-reviewed |
| P2 | Domain Art.7 compliance confirmed | No external inference traffic generated during smoke test (network verified) |
| P3 | VM provisioned via Proxmox/Ansible, documented in `terminus.infra` | VM resource in inventory; Ansible playbook committed |

---

## User Journeys

### Journey 1: Initial Setup (Operator installs Hermes for first time)

```
1. Operator receives VM credentials for hermes.trantor.internal
2. SSH into VM: ssh user@hermes.trantor.internal
3. Run install: curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
4. Run: hermes model
   → Select "Custom endpoint"
   → Base URL: https://ollama.trantor.internal/v1
   → API key: (skip — no auth on Ollama)
   → Model: qwen3:27b
5. Run: hermes chat
   → "Say hello" → receive response
6. Run: hermes (with file-write task) → confirm tool call executes
7. Verify: grep for cloud keys in ~/.hermes/config.yaml → none found
```

**Outcome:** Hermes is installed, wired to local Ollama, agentic tasks confirmed working.

### Journey 2: Daily Agentic Use (Operator uses Hermes for infra tasks)

```
1. SSH into hermes.trantor.internal
2. hermes → describe an infrastructure task
3. Hermes plans the task, executes tool calls (shell, file ops)
4. Hermes returns result and persists session memory
5. Operator iterates in same session context
```

**Outcome:** Multi-step autonomous task execution using local model — no cloud dependency.

### Journey 3: Another Operator Replicates Setup (Following runbook)

```
1. Read docs/terminus/inference/.../runbook.md
2. Provision VM via Ansible playbook
3. Run install script
4. Copy config template from runbook
5. Smoke test: hermes chat → confirm working
```

**Outcome:** Second operator is operational in under 30 minutes without custom intervention.

---

## Scope

### In Scope

| Item | Notes |
|------|-------|
| Proxmox VM provisioning | New dedicated VM (`hermes.trantor.internal`), Ansible playbook, committed to `terminus.infra` |
| Hermes Agent install | Via official install script; Python 3.11 + uv; `~/.hermes/hermes-agent/` |
| LLM backend configuration | `custom` provider pointing at `https://ollama.trantor.internal/v1`, model `qwen3:27b` |
| Context length configuration | `model.context_length: 32768` in Hermes config; verify Ollama server-side `OLLAMA_CONTEXT_LENGTH` |
| Tool calling validation | Confirm `qwen3:27b` function calling support via `ollama show qwen3:27b`; run agentic smoke test |
| **Postgres memory backend** | Provision `hermes_memory` database on existing Postgres server; create scoped `hermes` DB user; configure Hermes memory connection string |
| Credential security | `chmod 600 ~/.hermes/.env`; connection string for Postgres stored as env var, not plaintext in config |
| Runbook authoring | Install procedure, config template, Postgres setup, smoke test commands, troubleshooting, rollback |
| Local-first AI compliance verification | Confirm zero external AI traffic in smoke test |
| **Governance domain scaffold** | `lens-new-domain hermes` + `lens-new-service hermes/agent` + feature re-home before TechPlan |

### Out of Scope

| Item | Rationale |
|------|-----------|
| k3s deployment of Hermes Agent | Follow-on feature under `hermes/agent`; this is a VM install |
| Messaging platform integration (Telegram, Discord, Slack) | Follow-on feature; requires k3s daemon mode |
| IngressRoute for terminus-inference-gateway | Gateway is ClusterIP only; Hermes uses `ollama.trantor.internal` directly |
| Vault token auth for Ollama | Ollama has no auth layer; Vault auth is gateway-side only |
| Pulling new models into Ollama (e.g., hermes3:27b) | Use `qwen3:27b` which is already confirmed available |
| vLLM backend adapter for inference-gateway | Separate feature |
| BMAD / lens-work integration with Hermes | Not in scope |
| Routing Hermes through terminus-inference-gateway | Gateway has no IngressRoute; follow-on if IngressRoute is created |
| Hermes domain constitution authoring | Created via `lens-new-domain` before TechPlan; not part of PRD authoring |

---

## Functional Requirements

### FR-1: VM Provisioning

| ID | Requirement |
|----|------------|
| FR-1.1 | A dedicated Proxmox VM must be provisioned with the hostname `hermes.trantor.internal` |
| FR-1.2 | The VM must be reachable via SSH from within `trantor.internal` |
| FR-1.3 | The VM must have network access to `https://ollama.trantor.internal` and the public internet (for Hermes install script and pip packages) |
| FR-1.4 | VM provisioning must be automated via an Ansible playbook committed to `terminus.infra` |
| FR-1.5 | The VM hostname and IP must be registered in the internal DNS / Proxmox inventory |

### FR-2: Hermes Agent Installation

| ID | Requirement |
|----|------------|
| FR-2.1 | Hermes Agent must be installed to `~/.hermes/hermes-agent/` on the target VM |
| FR-2.2 | The `hermes` CLI must be available on `$PATH` without manual shell configuration after install |
| FR-2.3 | Python 3.11 must be available (via `uv` automatic provision or pre-installed) |
| FR-2.4 | The installed Hermes version must be recorded in the runbook (via `hermes --version`) |

### FR-3: LLM Backend Configuration

| ID | Requirement |
|----|------------|
| FR-3.1 | `~/.hermes/config.yaml` must set `model.provider: custom` |
| FR-3.2 | `~/.hermes/config.yaml` must set `model.base_url: https://ollama.trantor.internal/v1` |
| FR-3.3 | `~/.hermes/config.yaml` must set `model.default: qwen3:27b` (or current confirmed available model) |
| FR-3.4 | `~/.hermes/config.yaml` must set `model.context_length: 32768` |
| FR-3.5 | No cloud provider API keys (OpenAI, Anthropic, Google, Cohere) may be present in `~/.hermes/config.yaml` or `~/.hermes/.env` |
| FR-3.6 | If a future Vault token is required, it must be stored in `~/.hermes/.env` as `OPENAI_API_KEY=<token>` with `chmod 600` |

### FR-4: Tool Calling and Agentic Validation

| ID | Requirement |
|----|------------|
| FR-4.1 | Before the smoke test, `ollama show qwen3:27b` must be run to confirm function calling support in the model template |
| FR-4.2 | A chat smoke test (`hermes chat` → "Say hello") must succeed with a valid response |
| FR-4.3 | An agentic smoke test (a task requiring at least one tool call, e.g., write a file, run a shell command) must succeed and produce a visible tool invocation |
| FR-4.4 | If `qwen3:27b` lacks function calling support, an alternative model with tool calling must be identified and substituted |

### FR-5: Context Length

| ID | Requirement |
|----|------------|
| FR-5.1 | The Ollama server must be verified to run with `OLLAMA_CONTEXT_LENGTH` ≥ 32768 (check `OLLAMA_CONTEXT_LENGTH` env var on the Ollama host) |
| FR-5.2 | The smoke test must include a prompt exceeding 8k tokens to confirm the context window is not truncated |

### FR-6: Security

| ID | Requirement |
|----|------------|
| FR-6.1 | `~/.hermes/.env` must have file permissions `600` or stricter |
| FR-6.2 | The dedicated VM must not be shared with other services (isolation enforced by Proxmox VM boundary) |
| FR-6.3 | All inference traffic during smoke tests must remain within `trantor.internal` (Domain Art.7) |

### FR-8: Postgres Memory Backend

| ID | Requirement |
|----|------------|
| FR-8.1 | A database named `hermes_memory` must be provisioned on the existing Postgres server within `trantor.internal` |
| FR-8.2 | A Postgres user `hermes` must be created with permissions scoped exclusively to the `hermes_memory` database (no superuser, no cross-database access) |
| FR-8.3 | The Hermes memory backend must be configured to connect to the Postgres instance via a connection string stored in `~/.hermes/.env` (not in `config.yaml` plaintext) |
| FR-8.4 | The Postgres connection string format must be: `postgresql://hermes:PASSWORD@postgres.trantor.internal:5432/hermes_memory` (hostname TBD — to be confirmed against existing cluster config) |
| FR-8.5 | Hermes memory must persist across process restarts: a fact recorded in one session must be recallable in a subsequent session |
| FR-8.6 | The Postgres provisioning procedure (database creation, user creation, grant statements) must be documented in the runbook as repeatable SQL commands |
| FR-8.7 | The `hermes` Postgres user password must be stored in the Ansible vault or equivalent secrets store; it must not appear in plaintext in any committed file |

### FR-7: Documentation

| ID | Requirement |
|----|------------|
| FR-7.1 | A `runbook.md` must be authored covering: install procedure, config template, Postgres memory setup, smoke test commands, troubleshooting, and rollback |
| FR-7.2 | The runbook must include a "Language Justification" section (Python as third-party upstream application, not an authored terminus service) |
| FR-7.3 | The VM provisioning Ansible playbook must be committed to `terminus.infra` with documentation in the runbook |
| FR-7.4 | The runbook must include repeatable SQL for Postgres memory setup (CREATE DATABASE, CREATE USER, GRANT) |

---

## Non-Functional Requirements

### NFR-1: Performance

| ID | Requirement |
|----|------------|
| NFR-1.1 | First token latency for `hermes chat` requests must not exceed 30 seconds (network + model load) for `qwen3:27b` on the current Ollama server |
| NFR-1.2 | The Hermes install script must complete within 15 minutes on the provisioned VM |

### NFR-2: Reliability

| ID | Requirement |
|----|------------|
| NFR-2.1 | Hermes Agent install is user-space; no system service, no systemd unit; crash does not affect other terminus services |
| NFR-2.2 | Full rollback is `rm -rf ~/.hermes/` — no residual state in system directories |

### NFR-3: Security

| ID | Requirement |
|----|------------|
| NFR-3.1 | No credentials stored in plaintext in system-wide paths (only `~/.hermes/.env` with `600` permissions) |
| NFR-3.2 | Hermes Agent must not be configured with any external AI provider keys (enforced by config audit in AC-4) |
| NFR-3.3 | VM network egress to public internet is allowed only for package installation; inference traffic must route exclusively to `ollama.trantor.internal` |

### NFR-4: Maintainability

| ID | Requirement |
|----|------------|
| NFR-4.1 | Install version must be recorded in the runbook to enable future upgrade targeting |
| NFR-4.2 | Configuration must be documented as a replicable template for additional operators |

---

## Dependencies

| Dependency | Type | Status | Owner |
|------------|------|--------|-------|
| Proxmox VM provisioned (`hermes.trantor.internal`) | Prerequisite | Not started | Todd (infra) |
| `ollama.trantor.internal` IngressRoute live | Prerequisite | ✅ Confirmed live | Terminus infra |
| `qwen3:27b` available on Ollama server | Prerequisite | ✅ Confirmed available | Terminus infra |
| Ollama server-side `OLLAMA_CONTEXT_LENGTH ≥ 32768` | Prerequisite (verify) | Unconfirmed — must check | Terminus infra |
| `hermes-agent.nousresearch.com/install.sh` accessible from VM | Prerequisite | Requires internet access on VM | Network |
| Existing Postgres server on `trantor.internal` | Prerequisite | ✅ Confirmed running (used by Kaylee, Temporal, etc.) | Terminus infra |
| `hermes_memory` database provisioned on Postgres | Prerequisite (new story) | Not started | Todd (infra) |
| terminus.infra target repo registered in feature.yaml | Planning | Not yet added (TechPlan phase) | Lens lifecycle |
| `hermes` governance domain scaffolded via `lens-new-domain` | Planning (blocking TechPlan) | Not started | Lens lifecycle |

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| `qwen3:27b` lacks function calling support in Ollama template | Low | High | Verify with `ollama show qwen3:27b` before smoke test; fallback: use `qwen3:8b` or `hermes3:8b` if available |
| Ollama context window is too small (< 32768) | Medium | High | Check server `OLLAMA_CONTEXT_LENGTH` env var; request update from infra if needed |
| Hermes install script fails due to network restrictions on VM | Low | Low | Use `uv` directly as fallback; Python 3.11 install is separable |
| Upstream Hermes breaking change (unversioned install) | Low | Low | Pin version from `hermes --version` output after install; document in runbook |
| VM sizing too small for Hermes + uv environment | Low | Low | Size to at least 4 vCPU / 8GB RAM based on Python + uv workspace needs |
| Hermes Postgres memory config key name not confirmed | Medium | Medium | Verify exact config schema from Hermes docs before implementation; test with default SQLite first if Postgres key differs |
| Postgres connection refused from hermes VM (firewall/pg_hba) | Medium | Medium | Confirm `pg_hba.conf` allows connections from `hermes.trantor.internal`; document required pg_hba entry in runbook |

---

## Constitution Compliance

### Applicable Hard Gates (Resolved)

| Article | Requirement | Status |
|---------|------------|--------|
| Org Art.1 | Track declared | ✅ `full` |
| Org Art.2 | Phase artifacts before gate | ✅ prd.md + ux-design.md (in progress) |
| Org Art.4 | No confidential data exfiltration | ✅ All-local design; `https://ollama.trantor.internal/v1` only |
| Org Art.5 | Lifecycle evidence discipline | ✅ feature.yaml maintained |

### Informational Articles Addressed

| Article | Requirement | Resolution |
|---------|------------|-----------|
| Terminus Art.7 | Local-first AI data sovereignty | ✅ No external provider keys; all inference via `ollama.trantor.internal` |
| Terminus Art.12 | k3s deployment completeness | ✅ NOT APPLICABLE — VM install, no new k3s service endpoint |
| Inference Art.7 | Go as preferred implementation language | ✅ Python justified: Hermes Agent is a third-party MIT-licensed application authored by Nous Research; not a new terminus-inference service. Architecture doc must include explicit "Language Justification" section per Art.7 |
| Inference Art.5 | Controlled rollout and rollback | ✅ Rollback: `rm -rf ~/.hermes/`; documented in runbook |

### Pending Governance Actions (Pre-TechPlan)

| Action | Description | Status |
|--------|------------|--------|
| `lens-new-domain hermes` | Create hermes domain constitution scaffold in governance | ⏳ Blocked on BusinessPlan completion |
| `lens-new-service hermes/agent` | Register hermes/agent service under the new domain | ⏳ Blocked on domain scaffold |
| Feature re-home | Move feature from `terminus/inference` to `hermes/agent` in governance and control repo | ⏳ Blocked on domain + service scaffold |
| hermes constitution authoring | Author hermes-level and agent-level constitution rules | ⏳ TechPlan phase |
| `terminus.infra` in `target_repos` | Add terminus.infra to feature.yaml target_repos for Ansible/Postgres work | ⏳ TechPlan phase |

---

## Acceptance Criteria

### AC-1: Installation Complete
Hermes Agent is installed to `~/.hermes/hermes-agent/` on `hermes.trantor.internal`. `hermes --version` returns without error and the version is recorded in `runbook.md`.

### AC-2: Gateway Configuration
`~/.hermes/config.yaml` contains:
```yaml
model:
  default: qwen3:27b
  provider: custom
  base_url: https://ollama.trantor.internal/v1
  context_length: 32768
```
No cloud provider API keys are present in `~/.hermes/config.yaml` or `~/.hermes/.env`.

### AC-3: Chat Smoke Test Passes
`hermes chat` with a simple prompt (e.g., "Say hello") returns a valid response with HTTP 200 from `https://ollama.trantor.internal/v1`.

### AC-4: No External Provider Keys
Configuration audit confirms no keys for OpenAI, Anthropic, Google, Cohere, or any other cloud LLM provider exist in Hermes config files.

### AC-5: Context Length Validated
A prompt exceeding 8k tokens returns a coherent, non-truncated response, confirming the context window is not capped at 4096.

### AC-6: Agentic Tool Call Validated
An agentic task (e.g., "create a file called hello.txt with content 'hello world'") executes successfully, producing a visible tool call invocation and the expected file is created.

### AC-7: File Permissions Secured
`~/.hermes/.env` has permissions `600` or stricter. Verified via `ls -la ~/.hermes/.env`.

### AC-8: VM Provisioned via Ansible
A Proxmox VM (`hermes.trantor.internal`) is provisioned and accessible via SSH. The provisioning Ansible playbook is committed to `terminus.infra`.

### AC-9: Runbook Published
`runbook.md` is committed to the feature docs path, covering: install procedure, config template, Postgres memory setup (SQL commands), smoke test commands, context length verification, troubleshooting, rollback, and Language Justification section.

### AC-10: Memory Persistence Verified
A Hermes session is started, a specific fact is discussed (e.g., "Remember that my Postgres password is stored in Bitwarden"), Hermes is stopped, restarted, and the prior context is recalled from the `hermes_memory` Postgres database without re-providing it.

### AC-11: Postgres User Permissions Scoped
The `hermes` Postgres user can connect to `hermes_memory` but cannot connect to other databases on the shared Postgres cluster. Verified via `\l` in psql as the `hermes` user.

### AC-12: Hermes Domain Governance Scaffolded
`lens-new-domain hermes` has been run and the `hermes` domain constitution exists in governance. The feature has been re-homed to `hermes/agent` in feature.yaml before TechPlan begins.
