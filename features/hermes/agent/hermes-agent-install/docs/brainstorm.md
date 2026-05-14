# Brainstorm — Hermes Agent Install + Terminus-Inference Gateway Integration

**Feature:** terminus-inference-hermes-by-nous-research-install  
**Phase:** PrePlan  
**Date:** 2026-05-14  
**Facilitator:** Inquisitor Greyfax (Analyst)

---

## Feature Framing

The user wants to **install Hermes Agent** (the autonomous agent application by Nous Research) and **wire it to the terminus-inference gateway** as its LLM backend, rather than relying on cloud providers or hardcoding it to a specific model. The terminus-inference gateway already exposes an OpenAI-compatible `/v1/chat/completions` endpoint backed by Ollama on `trantor.internal`.

---

## Brainstorm Output

### 1. Where Should Hermes Agent Run?

**Option A — Developer VM (local install)**  
Install Hermes Agent on a developer machine or VM (e.g., `worker01.trantor.internal` or a personal dev laptop). User invokes `hermes chat` from a terminal. No k3s deployment required. Lightest-weight path — single developer or small team uses it interactively.

**Option B — k3s Service (gateway/daemon mode)**  
Run Hermes as a long-lived k3s pod in gateway mode. Hermes connects to a messaging platform (e.g., Telegram, Discord) and acts as a persistent AI assistant accessible from mobile/desktop. Requires k3s deployment, Namespace, IngressRoute, cert-manager Certificate, imagePullSecret, ArgoCD app, smoke-test AC (Domain Art.12).

**Option C — Both (VM for dev, k3s for production)**  
Install for development/testing on a VM first, then optionally promote to k3s gateway mode. Allows validation before cluster deployment.

**Which is the right scope?**  
The user said "install as part of inference service" — this leans toward Option B (permanent k3s residency) or at minimum Option A for initial install with Option B as a follow-on. The key decision is whether this feature targets personal use or shared team use.

---

### 2. Model Selection Strategy

**Option A — Pin to a specific model**  
Set `model.default: hermes3:27b` in config. Requires pulling the Hermes3 model into Ollama, which may need GPU VRAM planning.

**Option B — Point at gateway, pick any available model**  
Set `model.base_url: https://inference.trantor.internal/v1`, `model.default: qwen3:27b` (already available). The agent works with whatever model the gateway currently routes. Decouple agent config from model availability.

**Option C — Dynamic model selection**  
Use `/model` at runtime to switch between models available on the gateway. No single pinned model — user selects per session. Useful for comparing model behaviors.

**Why Option B wins:**  
The user explicitly said "point at my inference gateway instead of forcing it to a specific model." This confirms a config-driven approach where model selection is managed by the gateway's routing config, not hardcoded in Hermes config.

---

### 3. Auth Strategy (Vault Token)

The terminus-inference-gateway requires a Vault token for auth. Hermes can store this as an API key.

**Option A — Static Vault token in `.env`**  
Set `OPENAI_API_KEY=<vault-token>` in `~/.hermes/.env`. Simple, works out of the box. Token rotation requires manual `.env` update.

**Option B — ExternalSecret / environment injection**  
In k3s deployment, inject Vault token as a k8s secret via ExternalSecret controller. Hermes pod reads token from environment. Supports rotation.

**Option C — Vault Agent Sidecar**  
Use Vault Agent to inject a renewable token into the Hermes container. Most complex, but ensures automatic token renewal.

**For VM (personal) use:** Option A is appropriate.  
**For k3s (daemon) use:** Option B is required; aligns with the ExternalSecret pattern already used in `terminus-inference-gateway/deploy/k8s/`.

---

### 4. Messaging Platform Integration

Hermes Agent supports 20+ platforms (Telegram, Discord, Slack, WhatsApp, Signal, etc.). If running as a k3s gateway:

- **Telegram bot** — Most likely candidate. Supports rich message formatting, inline keyboards, streaming via message editing. Privacy-first for personal team use.
- **Discord** — Useful if the team already uses Discord for coordination. Supports threads, channel mentions.
- **Slack** — More enterprise-oriented; overkill for a small lab setup.
- **CLI only** — If not running in gateway mode, no messaging integration is needed.

**Question:** Does the user want a messaging gateway, or is CLI use sufficient for the initial install?

---

### 5. Terminal Backend for Hermes Agent Tasks

Hermes Agent can execute shell commands as part of agentic tasks.

**Option A — Local backend (default)**  
Commands run directly on the host machine. No isolation. Appropriate for personal dev machine use. Simple setup.

**Option B — Docker backend**  
All commands run inside a Docker container. Security hardened (`--cap-drop ALL`). Best for shared k3s deployment where agent runs untrusted commands.

**Option C — SSH backend**  
Commands run on a remote server over SSH. Useful if Hermes is installed on a laptop but should run commands on a server inside `trantor.internal`.

---

### 6. Use Cases — What Will Hermes Do on Terminus?

**Infrastructure / DevOps tasks:**  
- Query service health and k3s pod status via kubectl
- Run Ansible/scripts against the homelab
- Assist with writing Kubernetes manifests and Helm values
- Analyze logs and suggest fixes

**Development assistance:**  
- Code review, story implementation support
- Test generation, debugging assistance

**Research / Documentation:**  
- Summarize technical documents  
- Draft architecture notes and PRDs
- Interface with the BMAD workflow system

**Conversational assistant (gateway mode):**  
- Answer queries via Telegram about homelab status
- Act as a persistent AI assistant accessible from anywhere

---

### 7. Context Length Considerations

The terminus-inference-gateway currently routes to Ollama. Ollama's default context window may be low (4k–32k depending on VRAM). Hermes Agent requires at least 16k–32k context for reliable agentic use (system prompt + tool schemas consume significant tokens).

**Risk:** If `OLLAMA_CONTEXT_LENGTH` is not set server-side, Hermes will get a 4k window, which will cause unreliable behavior.

**Mitigation:** Set `model.context_length: 32768` explicitly in Hermes config as a floor until the gateway reports correct values via `/v1/models`.

---

### 8. What Models Are Available on the Gateway?

Currently the gateway routes to Ollama. Available models depend on what's been pulled. From repo memory, `qwen3:27b` (also referenced as `qwen3.6:27b`) has been benchmarked. `hermes3:27b` (Llama 70B quantized) would be the "ideal" model for Hermes Agent since it uses the Hermes prompt format natively.

**Key question:** Should this feature include pulling the Hermes3 model into Ollama, or just use whatever is already available?

---

### 9. Compliance Touchpoints

From the constitution stack (org → terminus domain → terminus-inference service):

- **Domain Art.7:** All inference must stay on `trantor.internal`. ✅ Satisfied by pointing Hermes at the local gateway.
- **Domain Art.12:** If deployed to k3s, must have full deployment completeness (ArgoCD app, Namespace, IngressRoute, cert-manager Certificate, imagePullSecret runbook, smoke-test AC).
- **Service Art.7:** Go is the preferred language. Hermes Agent is Python. A **Language Justification** section is required in architecture docs.
- **Service Art.1:** Any new inference routes added must have a route profile.
- **Service Art.4:** Config-driven guardrails must be visible in route profile.

---

### 10. Open Questions for Product Brief

1. **Scope**: Personal VM install only, or k3s gateway service deployment?
2. **Messaging platform**: CLI-only or Telegram/Discord gateway?
3. **Model**: Use existing gateway model (e.g., qwen3:27b) or pull Hermes3 into Ollama?
4. **Auth**: Is a Vault token already available for Hermes, or does a new policy/role need to be created?
5. **Context length**: Has `OLLAMA_CONTEXT_LENGTH` been set on the Ollama server?
6. **Terminal backend**: Local (simple) or Docker (sandboxed)?

---

## Decision Point Summary

| Decision | Recommended Direction | Rationale |
|---|---|---|
| Deployment scope | VM install first, k3s optional follow-on | Lower risk, validates integration before cluster deployment |
| Model | Use gateway's current best model (qwen3:27b), don't pin Hermes3 | User intent: decouple from specific model |
| Auth | Vault token in `~/.hermes/.env` | Consistent with existing gateway auth pattern |
| Messaging | CLI-only for initial install | Reduces scope, k3s gateway is a natural v2 |
| Terminal backend | Local | Dev machine use case; Docker optional |
| Context length | Explicit `model.context_length: 32768` in config | Defense against Ollama default 4k window |
