# Product Brief — Hermes Agent Install + Terminus-Inference Gateway Integration

**Feature:** terminus-inference-hermes-by-nous-research-install  
**Phase:** PrePlan  
**Track:** Full  
**Date:** 2026-05-14  
**Domain:** terminus  
**Service:** inference  

---

## 1. Problem Statement

The terminus platform has a functional inference gateway (`terminus-inference-gateway`) exposing an OpenAI-compatible API backed by Ollama running locally on `trantor.internal`. However, there is currently no autonomous AI agent application wired to that gateway. Developers and operators on the platform must either:

1. Use cloud LLM providers (violating Domain Art.7 — all inference must stay on `trantor.internal`), or  
2. Interact with the inference gateway manually via curl/API calls with no agentic automation capabilities.

There is no persistent AI assistant accessible from developer terminals or messaging platforms that uses the terminus inference stack.

---

## 2. Proposed Solution

Install **Hermes Agent** (the MIT-licensed autonomous agent application by Nous Research) on a terminal within the `trantor.internal` environment, configured to use the `terminus-inference-gateway` as its LLM backend via the `custom` provider type.

Hermes Agent will:
- Use the terminus-inference-gateway's OpenAI-compatible `/v1/chat/completions` endpoint
- Authenticate using a Vault token (consistent with existing gateway auth policy)
- Route through the gateway's model routing layer (no model hardcoded in Hermes config — model selection delegated to gateway routing)
- Provide an agentic CLI interface for development, infrastructure, and research tasks

**Initial scope:** VM install (user-space, single developer). k3s gateway deployment (messaging platform mode) is out of scope for this feature and will be addressed in a follow-on.

---

## 3. Goals

1. **G1 — Install Hermes Agent** on a VM within `trantor.internal` (or developer machine with network access to `trantor.internal`)
2. **G2 — Configure gateway integration** via `custom` provider in `~/.hermes/config.yaml` pointing at `terminus-inference-gateway`
3. **G3 — Validate end-to-end** via a smoke test confirming successful chat completion through the gateway
4. **G4 — Document the configuration pattern** so other developers on the platform can replicate the setup
5. **G5 — Enforce Domain Art.7** — zero external LLM provider keys configured; all inference stays local

---

## 4. Out of Scope

- k3s pod deployment of Hermes Agent (follow-on feature)
- Telegram / Discord / messaging gateway integration (follow-on feature)
- Pulling new models into Ollama (e.g., hermes3:27b) — out of scope unless needed for tool calling validation
- IngressRoute creation for terminus-inference-gateway (prerequisite dependency — may be in scope or a standalone infra task)
- BMAD / lens-work integration with Hermes Agent
- vLLM backend adapter for the terminus-inference-gateway

---

## 5. Acceptance Criteria

### AC-1: Installation Complete
Hermes Agent is installed to `~/.hermes/hermes-agent/` on the designated host. `hermes --version` or `hermes help` returns without error.

### AC-2: Gateway Configuration
`~/.hermes/config.yaml` contains:
```yaml
model:
  default: <model-name-available-on-gateway>
  provider: custom
  base_url: https://inference.trantor.internal/v1   # or confirmed gateway hostname
  context_length: 32768
```
`~/.hermes/.env` contains a valid Vault token scoped to inference-gateway access.

### AC-3: Smoke Test Passes
Running `hermes chat` with a simple user message (e.g., "Say hello") produces a valid chat response, confirming end-to-end connectivity through the gateway.

### AC-4: No External Provider Keys
`~/.hermes/config.yaml` and `~/.hermes/.env` contain **no** API keys for cloud LLM providers (OpenAI, Anthropic, Google, Cohere, etc.).

### AC-5: Context Length Validated
The response from AC-3 is not truncated, confirming the context window is sufficient for Hermes Agent's system prompt + tool schemas.

### AC-6: Documentation Written
A runbook document (`docs/terminus/inference/terminus-inference-hermes-by-nous-research-install/runbook.md`) covers: install procedure, config file template, Vault token setup, smoke test command, and troubleshooting steps.

---

## 6. Assumptions and Constraints

| Item | Detail |
|---|---|
| **Gateway URL** | `https://inference.trantor.internal/v1` — exact hostname must be confirmed or created as a prerequisite |
| **Gateway auth** | Vault token Bearer auth is already implemented in the gateway; a new Vault policy or token may need to be issued for Hermes |
| **Ollama model** | At least one tool-calling-capable model is available on `http://ollama.trantor.internal:11434` |
| **Context window** | Ollama server must be running with `OLLAMA_CONTEXT_LENGTH` ≥ 32768 |
| **Python 3.11** | Must be available on target host (Hermes installer will use `uv` to provision it) |
| **Target host** | A VM or developer machine with network access to `trantor.internal` |

---

## 7. Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Gateway has no IngressRoute (not externally reachable) | Medium | High | Confirm or create IngressRoute as prerequisite |
| Vault token scope too narrow / policy missing | Medium | High | Create dedicated Vault policy for inference-gateway consumer access |
| Ollama context window too small | High | High | Set `model.context_length: 32768` in Hermes config; verify server-side `OLLAMA_CONTEXT_LENGTH` |
| Tool calling not working for current model | Low | Medium | Test with `hermes chat` before declaring AC complete; fallback: switch model |
| Hermes install script fails due to network/Python issues | Low | Low | Use uv directly as fallback; Python 3.11 available separately |

---

## 8. Dependencies

| Dependency | Type | Status |
|---|---|---|
| `terminus-inference-gateway` deployed and healthy | Prerequisite | Deployed (verify ArgoCD health) |
| IngressRoute for inference gateway at `*.trantor.internal` | Prerequisite | Unknown — must confirm |
| Vault token with inference-gateway consumer policy | Prerequisite | Unknown — may need new policy/token |
| Ollama running with compatible model and adequate context | Prerequisite | Partially known (qwen3:27b confirmed) |

---

## 9. Constitution Compliance

### Domain Constitution (terminus)

| Article | Requirement | Disposition |
|---|---|---|
| Art.7 — Inference locality | All inference stays on `trantor.internal` | ✅ Met — `base_url` points at local gateway; no external provider keys |
| Art.12 — k3s completeness | ArgoCD app, Namespace, IngressRoute, cert-manager Certificate, imagePullSecret runbook, smoke-test AC | ⚠️ N/A for initial VM install — applies if follow-on k3s deployment is created |

### Service Constitution (terminus-inference)

| Article | Requirement | Disposition |
|---|---|---|
| Art.1 — Route profile completeness | New inference routes must have route profiles | ✅ N/A — no new routes added to the gateway |
| Art.4 — Config-driven guardrails | Guardrails visible in route profile | ✅ N/A — Hermes is a client, not a new gateway route |
| Art.7 — Language preference (Go) | Go preferred; Python requires Language Justification section in architecture | ⚠️ Hermes Agent is Python; Language Justification section required in TechPlan-phase architecture doc |

### Language Justification Summary

Hermes Agent is a pre-built, MIT-licensed third-party application developed and maintained by Nous Research. It cannot be rewritten in Go without forking the entire project, which is out of scope. Hermes Agent is a **consumer** of the terminus-inference API — it does not run inside the gateway module or contribute to its codebase. The Service Art.7 "Go preferred" rule applies to services authored as part of terminus; third-party tools using the API are outside that scope. The Python runtime (`uv` + 3.11) is fully isolated in `~/.hermes/` and does not cross-contaminate any Go service boundaries.

---

## 10. Success Metrics

| Metric | Target |
|---|---|
| Time to first chat response via gateway | < 30 seconds (excluding model inference time) |
| Context window (tokens available) | ≥ 32,768 |
| External LLM calls | 0 — confirmed by network audit |
| Smoke test pass rate | 100% on first run after configuration |

---

## 11. Future Considerations (Out of Scope)

- **v2: k3s deployment** — Run Hermes Agent as a persistent k3s pod with Telegram/Discord gateway integration, ExternalSecret Vault token injection, ArgoCD management
- **v2: Model catalog** — Pull Hermes3 model (Llama 70B, Hermes prompt format) into Ollama for optimized tool-calling performance
- **v3: MCP integration** — Connect Hermes Agent to internal MCP servers (kubectl, ArgoCD, vault, Prometheus) for infrastructure automation
- **v3: BMAD orchestration** — Use Hermes Agent as an alternative front-end for BMAD workflow execution on terminal

---

## 12. Stakeholders

| Name | Role | Engagement |
|---|---|---|
| Todd Hintzmann | Lead / Owner | Full lifecycle engagement |
