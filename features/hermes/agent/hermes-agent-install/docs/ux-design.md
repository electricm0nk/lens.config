---
feature: terminus-inference-hermes-by-nous-research-install
doc_type: ux-design
status: draft
domain: hermes
service: agent
input_documents:
  - docs/terminus/inference/terminus-inference-hermes-by-nous-research-install/prd.md
  - docs/terminus/inference/terminus-inference-hermes-by-nous-research-install/product-brief.md
  - docs/terminus/inference/terminus-inference-hermes-by-nous-research-install/research.md
  - docs/terminus/inference/terminus-inference-hermes-by-nous-research-install/brainstorm.md
---

# UX Design — Hermes Agent Install
**Feature:** terminus-inference-hermes-by-nous-research-install
**Date:** 2026-05-14
**Domain:** hermes *(top-level, pending governance scaffold)* | **Service:** agent

> This is a **CLI/terminal tool UX design**, not a web application. UX here means:
> interaction patterns, command ergonomics, configuration file design, first-run experience,
> error messaging, and operator mental models.

---

## 1. UX Context and Design Principles

### 1.1 Target Operator Profile

**Primary user:** A solo infrastructure operator (Todd) who:
- Is comfortable in a terminal; expects CLI-first interaction
- Manages `trantor.internal` infrastructure (k3s, Proxmox, ArgoCD, Temporal, Postgres)
- Uses BMAD/Lens for planning; uses Copilot for code assistance
- Wants an **autonomous local AI agent** for infra tasks without cloud egress
- Values **minimal ceremony**: install once, use daily, forget it's configured

**Key expectation**: After setup, `hermes` should feel like a natural terminal companion —
not a complex tool that requires remembering flags or re-explaining context every session.

### 1.2 Design Principles

| Principle | Implication for This Feature |
|-----------|------------------------------|
| **Local-first** | Zero cloud egress; all interaction stays within `trantor.internal` |
| **Low ceremony** | Install once, configure once, use forever |
| **Transparent** | Operator can see what model/endpoint is active; no hidden state |
| **Recoverable** | All configuration is files; bad state is fixable by editing a file |
| **Persistent context** | Memory backend means Hermes "remembers" — this is a feature, not a footgun |

---

## 2. Installation Experience

### 2.1 First-Run Journey (Operator Persona: Todd)

```
SSH into hermes.trantor.internal
  └─► Install Python 3.11 + uv (via Ansible playbook)
        └─► Run Hermes install script
              └─► Configure ~/.hermes/config.yaml
                    └─► Configure ~/.hermes/.env (Postgres conn string)
                          └─► Provision hermes_memory DB (run SQL on Postgres server)
                                └─► First `hermes chat` smoke test
                                      └─► First agentic task test
```

**Design decision:** The install experience should be **fully documented and reproducible**.
An operator who has never seen Hermes before should be able to follow `runbook.md` from
top to bottom and have a working, memory-enabled agent in under 20 minutes.

### 2.2 Install Script UX

The upstream install script (`curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash`)
will likely:
1. Check Python 3.11 / uv availability
2. Clone or download the Hermes agent bundle to `~/.hermes/hermes-agent/`
3. Install a shell wrapper or add to PATH

**Operator expectation:** After the install script runs, `hermes --version` should work.
If it doesn't, the runbook must provide the PATH fix: `export PATH="$HOME/.hermes/bin:$PATH"`.

**Shell integration:** The runbook should instruct adding the PATH export to `~/.bashrc`
or `~/.zshrc` so it persists across SSH sessions.

### 2.3 First-Run Configuration Wizard vs. Manual Config

Hermes may prompt for initial configuration interactively. The runbook must cover both cases:

**Case A: Hermes prompts for provider on first run**
- Select `custom` or `openai-compatible` provider
- Enter `https://ollama.trantor.internal/v1` as base URL
- Enter `qwen3:27b` as default model
- Skip API key (leave blank — Ollama has no auth)

**Case B: Direct config file editing**
- Write `~/.hermes/config.yaml` manually from the template in the runbook
- No interactive prompts needed

The runbook provides **both paths** so the operator is not blocked by either behavior.

---

## 3. Configuration File Design

### 3.1 `~/.hermes/config.yaml` — Annotated Template

```yaml
# Hermes Agent Configuration
# Generated: 2026-05-14 | trantor.internal hermes VM

model:
  default: qwen3:27b
  provider: custom          # Use 'custom' for OpenAI-compatible endpoints
  base_url: https://ollama.trantor.internal/v1
  context_length: 32768     # Must match or exceed Ollama server OLLAMA_CONTEXT_LENGTH

# Memory backend — connection string stored in .env, not here
# memory:
#   type: postgres          # Exact key TBD from Hermes docs; verify before implementation
#   database_url: ${HERMES_MEMORY_DATABASE_URL}   # Reads from .env
```

**UX note:** The `memory` section key name must be verified against the live Hermes docs
before implementation. The runbook must include the exact key after verification.

### 3.2 `~/.hermes/.env` — Secrets File

```bash
# Hermes Agent Secrets
# File permissions: chmod 600 ~/.hermes/.env
# DO NOT commit this file

HERMES_MEMORY_DATABASE_URL=postgresql://hermes:PASSWORD@postgres.trantor.internal:5432/hermes_memory
```

**UX rationale:** Separating secrets from config follows the 12-factor principle and
prevents accidental exposure when copying `config.yaml` to documentation or tickets.

**Permission enforcement:** The runbook must include `chmod 600 ~/.hermes/.env` as a
required post-setup step. AC-7 requires this as verified.

### 3.3 Configuration Discoverability

After initial setup, an operator should be able to verify the active configuration with:

```bash
hermes config show      # or: cat ~/.hermes/config.yaml
hermes status           # or equivalent health/status subcommand
```

If no such command exists, the runbook documents: `cat ~/.hermes/config.yaml` as the
verification step.

---

## 4. Postgres Memory Backend UX

### 4.1 Provisioning Experience (Operator View)

The Postgres memory setup is a one-time operator action. The runbook delivers it as
**copy-paste SQL** — no operator should need to think about syntax:

```sql
-- Run as postgres superuser on postgres.trantor.internal
CREATE DATABASE hermes_memory;
CREATE USER hermes WITH PASSWORD 'REPLACE_WITH_STRONG_PASSWORD';
GRANT ALL PRIVILEGES ON DATABASE hermes_memory TO hermes;

-- Restrict cross-database access (safety constraint)
REVOKE CONNECT ON DATABASE postgres FROM hermes;
REVOKE CONNECT ON DATABASE kaylee FROM hermes;
-- Add REVOKE statements for all other DBs on the cluster
```

**UX decision:** The password is NOT in the runbook. The runbook instructs:
> "Generate a strong password and store it in Bitwarden. Use it below and in `.env`."

### 4.2 Connection Verification UX

After setup, the operator verifies the connection works before starting Hermes:

```bash
# From hermes.trantor.internal
psql "postgresql://hermes:PASSWORD@postgres.trantor.internal:5432/hermes_memory" \
  -c "SELECT 1;"
```

Expected response: `?column? = 1`. This is the **connection smoke test** in the runbook.

### 4.3 Memory Persistence UX (Operator-Facing Behavior)

The key UX value of the Postgres backend: **Hermes remembers across sessions.**

**What this means for the operator:**
- Session A: "I named the hermes VM `hermes.trantor.internal`"
- *Stop and restart Hermes*
- Session B: Hermes can reference that fact without being re-told

**AC-10 test procedure (from runbook):**
```
1. Start: hermes chat
2. Say: "Remember that the k3s API server is at api.trantor.internal:6443"
3. Exit Hermes
4. Restart: hermes chat
5. Ask: "What is the k3s API server address?"
6. Expect: Hermes recalls the address from memory (not from system prompt)
```

### 4.4 Memory Failure UX (Error Path)

If Postgres is unreachable on startup, Hermes should:
- Log a clear error: `Cannot connect to memory backend: postgresql://hermes@postgres...`
- Either fall back to in-session memory (no persistence) OR refuse to start

The runbook documents the recovery procedure:
1. Check network connectivity to `postgres.trantor.internal`
2. Verify `pg_hba.conf` allows the hermes VM IP
3. Verify the `hermes_memory` database exists and user has CONNECT rights
4. Test with psql directly before restarting Hermes

---

## 5. Daily Interaction Patterns

### 5.1 Chat Mode (Conversational)

```bash
hermes chat
```

**Interaction model:** Natural language, multi-turn. Hermes maintains context within the
session. The Postgres backend extends this across sessions.

**Operator mental model:** "I'm talking to a terminal assistant that knows my infra."
Not: "I'm submitting API requests to a language model."

### 5.2 Agentic Task Mode

```bash
hermes "create a file called deploy-notes.txt with today's deployment summary"
hermes "search the k3s logs for the last ArgoCD sync error and summarize it"
```

**UX expectation:** Hermes should show progress — tool calls should be visible to the
operator so they can intervene if Hermes takes an unexpected action. This is a standard
agentic tool — the UX of tool call visibility is upstream (Hermes author's concern);
the runbook documents that tool calls are visible and expected.

### 5.3 Interrupt and Recovery

If an agentic task goes wrong, the operator can:
- `Ctrl+C` to interrupt mid-execution
- The interrupted session's partial state may be in Postgres memory

**Operator expectation:** Ctrl+C should always work. Hermes should not silently spawn
background processes that survive after the terminal exits.

---

## 6. Error Messages and Recovery UX

### 6.1 Connection Errors (LLM Endpoint Unreachable)

**Scenario:** `ollama.trantor.internal` is down.

**Expected UX:**
```
Error: Cannot connect to LLM endpoint https://ollama.trantor.internal/v1
Status: Connection refused / DNS resolution failed
Recovery: Verify Ollama is running on the inference server
  → Check: kubectl get pod -n ollama (or: curl https://ollama.trantor.internal/v1/models)
```

The runbook documents this recovery path explicitly.

### 6.2 Model Not Found

**Scenario:** `qwen3:27b` is not available on Ollama.

**Expected UX:**
```
Error: Model 'qwen3:27b' not found at https://ollama.trantor.internal/v1
Recovery: Run 'ollama pull qwen3:27b' on the Ollama server
  → SSH into the inference server: ollama list
```

### 6.3 Context Overflow

**Scenario:** Operator feeds Hermes a very large file; response is truncated.

**Design response:** The runbook notes the `context_length: 32768` setting and instructs:
- If truncation occurs, increase `context_length` in config.yaml
- Verify `OLLAMA_CONTEXT_LENGTH` on the Ollama server matches or exceeds the requested value

### 6.4 Postgres Memory Error

**Scenario:** Postgres is unreachable or credentials are wrong.

**Expected UX:**
```
Warning: Memory backend unavailable (postgres://hermes@postgres.../hermes_memory)
Hermes will continue without persistent memory this session.
Recovery: Check ~/.hermes/.env and verify Postgres connectivity
  → psql "postgresql://hermes:PASSWORD@postgres.trantor.internal:5432/hermes_memory"
```

The runbook documents this recovery path (Section: Postgres Troubleshooting).

---

## 7. Onboarding Sequence Design

### 7.1 Runbook Structure (Operator-Facing Document)

The `runbook.md` is the primary UX artifact for operators who need to install or
replicate this setup. Its structure follows the **install journey** (Section 2.1):

```
runbook.md
├── Prerequisites
│   ├── VM provisioning (Ansible playbook reference)
│   └── Postgres server access confirmed
├── Installation
│   ├── Python 3.11 + uv setup
│   ├── Hermes install script
│   └── PATH configuration
├── Configuration
│   ├── config.yaml template (annotated)
│   ├── .env template + permission hardening
│   └── Language Justification (Art.7 compliance)
├── Postgres Memory Setup
│   ├── SQL commands (copy-paste)
│   ├── Connection smoke test
│   └── pg_hba.conf notes
├── Smoke Tests
│   ├── Chat smoke test (AC-3)
│   ├── No external keys check (AC-4)
│   ├── Context length test (AC-5)
│   ├── Agentic tool call test (AC-6)
│   ├── File permissions check (AC-7)
│   └── Memory persistence test (AC-10)
├── Troubleshooting
│   ├── LLM endpoint unreachable
│   ├── Model not found
│   ├── Context overflow
│   └── Postgres connection failure
└── Rollback
    └── rm -rf ~/.hermes/ (full uninstall)
```

### 7.2 Section: Language Justification (Art.7 Compliance)

The runbook must include a dedicated section:

```markdown
## Language Justification

Hermes Agent is authored in Python by Nous Research and distributed under the MIT license.
This is a third-party application, not a new terminus-authored service. The Go language
preference (Inference Art.7) applies to new services authored within the terminus platform.
It does not govern the choice of third-party tool installations. No Python services are
being authored as part of this feature; Hermes is installed and configured as-is.
```

---

## 8. Configuration Validation UX

### 8.1 Pre-Launch Checklist (Operator Self-Verification)

Before declaring the install complete, the operator should be able to run through:

| Check | Command | Expected |
|-------|---------|----------|
| Hermes version | `hermes --version` | Version string, no error |
| Config file exists | `cat ~/.hermes/config.yaml` | Shows qwen3:27b + ollama URL |
| Env file secure | `ls -la ~/.hermes/.env` | Permissions: `-rw-------` |
| LLM endpoint live | `curl -s https://ollama.trantor.internal/v1/models \| jq .` | JSON with model list |
| qwen3:27b available | Check curl output above | `qwen3:27b` in model list |
| Postgres connection | `psql "$HERMES_MEMORY_DATABASE_URL" -c "SELECT 1;"` | Returns `1` |
| Chat smoke test | `hermes chat` → "Say hello" | Response received |
| Agentic test | `hermes "create file /tmp/test.txt content 'hello'"` | File created |
| Memory test | See AC-10 procedure | Prior context recalled |

This checklist is embedded in the runbook as the **Go-Live Verification** section.

---

## 9. Information Architecture Summary

### Key Files and Their UX Roles

| File | Purpose | Operator Interaction |
|------|---------|---------------------|
| `~/.hermes/config.yaml` | Primary configuration | Edit once at setup; rarely touched after |
| `~/.hermes/.env` | Secrets (Postgres conn string) | Set once; never committed |
| `~/.hermes/hermes-agent/` | Application code | Never touched by operator |
| `hermes_memory` (Postgres DB) | Persistent memory store | Operator never directly queries; Hermes manages it |
| `runbook.md` (docs) | Operator onboarding reference | Read during setup; reference for troubleshooting |
| Ansible playbook (`terminus.infra`) | VM provisioning | Run once during VM creation |

### UX Flow Summary

```
VM provisioned (Ansible)
  │
  ▼
Hermes installed + PATH set
  │
  ▼
config.yaml written (model + endpoint)
  │
  ▼
.env written (Postgres URL) + chmod 600
  │
  ▼
Postgres: DB + user provisioned
  │
  ▼
Smoke tests passed (chat + agentic + memory)
  │
  ▼
Hermes is operational ✅
  │
  ▼
Daily use: `hermes chat` or `hermes "task description"`
```

---

## 10. Open UX Questions

| # | Question | Impact | Resolution Path |
|---|----------|--------|----------------|
| UX-1 | What exact config key does Hermes use for the Postgres memory backend? | High — determines `config.yaml` template | Verify against live Hermes docs during implementation story |
| UX-2 | Does Hermes display tool call output in the terminal, or is it silent? | Medium — affects operator trust in agentic tasks | Discover during first agentic smoke test |
| UX-3 | Does Hermes have a `--config` flag to point to a non-default config file? | Low — useful for multi-user or test configs | Check `hermes --help` during install |
| UX-4 | What does Hermes show when memory backend is unreachable on startup — error or silent fallback? | Medium — affects runbook troubleshooting section | Discover by temporarily misconfiguring Postgres during setup |
| UX-5 | Does Hermes support a `hermes status` or `hermes config show` command for config inspection? | Low — convenience for operators | Check `hermes --help` during install |
