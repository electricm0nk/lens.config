# Research — Hermes Agent by Nous Research

**Feature:** terminus-inference-hermes-by-nous-research-install  
**Phase:** PrePlan  
**Date:** 2026-05-14  
**Researcher:** Inquisitor Greyfax (Analyst)  
**Sources:** Hermes Agent official documentation (hermes-agent.nousresearch.com), terminus-inference-gateway architecture and deployment files

---

## 1. What Is Hermes Agent?

Hermes Agent is an **open-source autonomous AI agent application** developed by Nous Research under the MIT license. It is **not a model** — it is a software application that acts as an AI assistant and autonomous agent, using any compatible LLM as its inference backend.

Key characteristics:
- **Autonomous agentic execution**: Executes multi-step tasks using tool calls, file system access, shell commands, and web browsing
- **Multi-platform messaging gateway**: Connects to 20+ messaging platforms (Telegram, Discord, Slack, WhatsApp, Signal, etc.) to provide a persistent AI assistant accessible from any device
- **Memory and skills system**: Persists user preferences, task history, and learned patterns across sessions
- **MCP (Model Context Protocol) support**: Integrates with MCP servers for extended tool capabilities
- **Python runtime**: Python 3.11 + `uv` package manager
- **Installation path**: `~/.hermes/hermes-agent/` (user-space install)
- **Config location**: `~/.hermes/config.yaml` and `~/.hermes/.env`

---

## 2. Installation

**Official installation command:**
```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

This installs:
- Python 3.11 (via `uv` if not present)
- Hermes Agent to `~/.hermes/hermes-agent/`
- CLI entry point: `hermes`

**Post-install steps:**
1. `hermes model` — configure the LLM backend
2. `hermes chat` — verify interactive chat works
3. `hermes` — list all available commands

---

## 3. LLM Backend Configuration

Hermes Agent is backend-agnostic and supports any OpenAI-compatible endpoint via the "Custom endpoint" provider type. This is the primary integration mechanism for terminus.

### 3a. Configuration File Format

Path: `~/.hermes/config.yaml`

```yaml
model:
  default: qwen3:27b              # model name as exposed by the gateway
  provider: custom                # use OpenAI-compatible custom endpoint
  base_url: https://inference.trantor.internal/v1   # terminus-inference-gateway
  context_length: 32768           # explicit override — required for Ollama-backed endpoints
```

**No API key is needed for endpoints without auth**; for endpoints requiring a token (like terminus-inference-gateway using Vault tokens), set it in `~/.hermes/.env`:

```bash
# ~/.hermes/.env
OPENAI_API_KEY=<vault-token>
```

### 3b. Interactive Configuration Method

Alternatively, via the CLI:
```
hermes model
```
Select "Custom endpoint (self-hosted / VLLM / etc.)", then:
- **Base URL**: `https://inference.trantor.internal/v1`
- **API key**: Skip (or enter Vault token)
- **Model name**: `qwen3:27b` (or current gateway-routed model)

### 3c. Named Custom Provider (Multi-Gateway Support)

For environments with multiple LLM backends:
```yaml
custom_providers:
  - name: terminus-inference
    base_url: https://inference.trantor.internal/v1
    # api_key omitted for internal endpoints

model:
  default: qwen3:27b
  provider: custom:terminus-inference
  context_length: 32768
```

---

## 4. Terminus-Inference Gateway — Integration Profile

**What it is:** A custom Go service implementing the OpenAI-compatible chat completions API (`POST /v1/chat/completions`). Deployed on k3s inside `trantor.internal`. Routes requests to Ollama at `http://ollama.trantor.internal:11434`.

**Auth model:** Vault token passed as Bearer token. Required for all inference requests (HTTP 401 on missing/invalid token).

**Public endpoint pattern:** `https://inference.trantor.internal/v1` (exact hostname TBD — see Section 7 below)

**API compatibility:**
- `POST /v1/chat/completions` — primary inference endpoint (OpenAI-compatible request/response)
- `GET /healthz` — liveness check (200 OK)
- `GET /readyz` — per-adapter health check

**Hermes compatibility:** The terminus-inference-gateway is fully compatible with Hermes Agent's `custom` provider type. No changes to the gateway are required for Hermes integration.

---

## 5. Context Length Management

**Risk:** Ollama defaults to a low context window (typically 2048–4096 tokens) unless `OLLAMA_CONTEXT_LENGTH` or `OLLAMA_NUM_CTX` is set server-side.

**Impact on Hermes:** The Hermes system prompt + tool schema definitions alone consume ~4000–8000 tokens. A 4k context window will be exhausted immediately, causing truncated or broken responses.

**Mitigation strategies:**

| Strategy | Location | Effect |
|---|---|---|
| `model.context_length: 32768` in Hermes config | `~/.hermes/config.yaml` | Forces context window request in each inference call |
| `OLLAMA_CONTEXT_LENGTH=32768` | Ollama server env var | Sets default window for all models server-side |
| `OLLAMA_NUM_CTX=32768` in model Modelfile | Ollama | Sets window for a specific model's default |

**Recommendation:** Set `model.context_length: 32768` in Hermes config **and** request that the Ollama server configuration be validated. Both layers provide defense-in-depth.

---

## 6. Tool Calling Requirements

Hermes Agent relies heavily on tool/function calling for agentic execution.

**Ollama (default backend):** Tool calling is enabled by default for models that include function calling in their template. Verify with:
```bash
ollama show qwen3:27b | grep -A5 template
```
Qwen3 and Hermes3 model families support function calling natively.

**vLLM (if gateway adds vLLM adapter):** Requires `--tool-call-parser hermes` flag at server startup. The "hermes" parser was specifically designed for the Hermes/Qwen 2.5 model family.

**Current terminus-inference-gateway:** Routes to Ollama only. Tool calling works out of the box for compatible models (Qwen3, Hermes3). No special vLLM flag needed.

---

## 7. Terminus IngressRoute — Gateway Public URL

**Current state:** The terminus-inference-gateway Helm chart does not include an IngressRoute template. The deploy manifests (`deploy/k8s/`) and Helm values do not define an external hostname. The gateway currently runs as a ClusterIP service accessible only within the k3s cluster.

**Implication for Hermes Agent:** If Hermes Agent is installed on a VM outside the k3s cluster (e.g., on a developer machine or a separate VM), an IngressRoute exposing the gateway at a `*.trantor.internal` hostname is required.

**Expected URL pattern** (based on Domain Art.12 and existing service naming):  
`https://inference.trantor.internal` (or `https://inference-gw.trantor.internal`)

**Action required:** The inference gateway IngressRoute must be created (or confirmed existing) as a prerequisite to Hermes Agent configuration. This may be in scope for this feature or may already exist via a separate ArgoCD app.

---

## 8. Python Language Justification

**Terminus-inference Service Constitution Art.7:** Go is the preferred language for inference service components.

**Hermes Agent is a Python application.** This is an upstream constraint — the application is authored by Nous Research and cannot be changed. The language justification rationale:

1. **Third-party application**: Hermes Agent is a pre-built, MIT-licensed application maintained upstream. Authoring an equivalent Go application is out of scope for this feature.
2. **No service boundary crossed**: Hermes Agent is a client that calls the inference gateway API. It does not run inside the gateway module or share its codebase. The "Go preferred" rule applies to services authored as part of terminus, not to third-party tools consuming the API.
3. **uv runtime isolation**: Hermes installs into `~/.hermes/` with its own isolated Python environment, fully managed by `uv`. No cross-contamination with any Go services.

**Required documentation:** The architecture document (TechPlan phase) must include an explicit "Language Justification" section per Service Art.7.

---

## 9. Security Considerations

**Vault token storage:** The Vault token stored in `~/.hermes/.env` is a plaintext credential at rest. On a shared machine, this is a risk. Mitigations:
- Restrict file permissions: `chmod 600 ~/.hermes/.env`
- Use a short-lived Vault token with narrow policy (inference API calls only, no admin capabilities)
- For k3s deployment: inject via ExternalSecret; never store in image or ConfigMap

**Terminal backend isolation:** By default, Hermes executes shell commands directly on the host. On shared systems, sandboxing via Docker is recommended.

**No external LLM calls:** Domain Art.7 compliance is enforced by the Hermes config pointing exclusively at `trantor.internal`. No API keys for external providers (OpenAI, Anthropic, etc.) should be configured.

---

## 10. Deployment Options Summary

| Option | Complexity | k3s Required | Auth Mechanism | Status |
|---|---|---|---|---|
| VM / dev machine install | Low | No | `~/.hermes/.env` Vault token | ✅ Recommended for initial install |
| k3s pod (CLI gateway) | Medium | Yes | ExternalSecret Vault token | Optional follow-on |
| k3s pod (Telegram gateway) | High | Yes | ExternalSecret + Telegram bot token | Optional v2 |

---

## 11. Upstream References

- Hermes Agent documentation: https://hermes-agent.nousresearch.com/docs
- Installation guide: https://hermes-agent.nousresearch.com/docs/getting-started/installation
- Configuration reference: https://hermes-agent.nousresearch.com/docs/user-guide/configuration
- Providers / custom endpoint: https://hermes-agent.nousresearch.com/docs/integrations/providers
- License: MIT — https://github.com/NousResearch/hermes-agent
- Terminus-inference-gateway architecture: `docs/terminus/inference/gateway/architecture.md`
- Terminus-inference-gateway deploy: `TargetProjects/terminus/inference/terminus-inference-gateway/deploy/`
