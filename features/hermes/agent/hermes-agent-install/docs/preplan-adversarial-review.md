# Adversarial Review: terminus-inference-hermes-by-nous-research-install / preplan

**Reviewed:** 2026-05-14T00:00:00Z  
**Source:** phase-complete  
**Overall Rating:** pass-with-warnings

---

## Summary

The PrePlan artifact set (brainstorm.md, research.md, product-brief.md) is coherent and well-grounded. The feature intent is clear: install Hermes Agent and wire it to the terminus-inference gateway to provide a local-first autonomous agent capability without cloud provider keys. Constitution compliance is correctly framed (Domain Art.7, Service Art.7 language justification). The primary gap is a cluster of three unresolved High-severity scoping decisions — the exact gateway URL, the Vault token issuance path, and the IngressRoute status — that must be resolved before BusinessPlan can produce a valid PRD. These are acceptable open questions at PrePlan stage but are the first blocking items for the next phase.

---

## Findings

### Critical

None.

---

### High

| # | Dimension | Finding | Recommendation |
|---|-----------|---------|----------------|
| H1 | Coverage Gaps | **Gateway endpoint URL is unresolved and contradictory.** AC-2 and research.md use `https://inference.trantor.internal/v1` as a guess, but the gateway architecture confirms no IngressRoute exists for the inference-gateway — only ClusterIP. The live endpoint visible in terminal history is `https://ollama.trantor.internal/v1/chat/completions`, which is the Ollama service directly, not the Go gateway. Pointing Hermes at the Ollama endpoint bypasses the gateway auth layer (Vault token validation) entirely. | Resolve before BusinessPlan: (a) confirm whether an IngressRoute for terminus-inference-gateway exists via `kubectl get ingressroute -A`; (b) if not, make an explicit scoping decision: create the IngressRoute in this feature, or point Hermes directly at Ollama and document the auth bypass; (c) update AC-2's `base_url` to the confirmed endpoint. |
| H2 | Cross-Feature Dependencies | **Vault token issuance path is unresolved with no owner.** AC-2 requires a Vault token scoped to inference-gateway access. The assumptions table acknowledges "may need new policy/token" but no action item, owner, or Vault policy name is documented. Creating a Vault policy and issuing a token is a non-trivial ops step. If this is blocked on platform access controls, it could stall the entire install. | Add a concrete prerequisite action before BusinessPlan: identify the Vault policy for inference consumers (or draft one), confirm whether a short-lived token can be issued, and document the exact command in the runbook plan. |
| H3 | Coverage Gaps | **IngressRoute dependency is declared "may be in scope" which is not a decision.** Without an IngressRoute, the smoke test (AC-3) is unverifiable from any machine outside the k3s cluster. The product brief leaves this open, which prevents BusinessPlan from scoping implementation work. | Make a binary decision before BusinessPlan: (a) IngressRoute creation is in scope for this feature (adds a new story to dev phase), or (b) the designated install host is inside the k3s cluster network with direct ClusterIP access. Vague "may be" blocks all downstream planning. |

---

### Medium / Low

| # | Severity | Dimension | Finding | Recommendation |
|---|---|---|---|---|
| M1 | Medium | Assumptions | **`ollama.trantor.internal` vs `terminus-inference-gateway` — live endpoint not confirmed.** Terminal history shows `ollama.trantor.internal` is live; it's unclear if this routes through the Go gateway or directly to Ollama. If it's a direct Ollama IngressRoute, using it bypasses gateway auth. | Run `kubectl get ingressroute -A` and confirm which service each hostname resolves to before BusinessPlan. |
| M2 | Medium | Coverage Gaps | **Context length server-side setting is acknowledged as a risk but has no verification AC.** The product brief lists `OLLAMA_CONTEXT_LENGTH ≥ 32768` as a constraint but no AC checks it. | Add a prerequisite check or explicit AC: send a large-context request and confirm the window is not truncated at 4k. |
| M3 | Medium | Logic Flaws | **AC-5 (context length validated) uses "Say hello" which fits in 4k tokens.** The smoke test does not actually stress the context window. | Strengthen AC-5: send a prompt exceeding 4k tokens and confirm a coherent response. |
| M4 | Medium | Coverage Gaps | **`~/.hermes/.env` file permissions not in any AC.** Vault token stored in plaintext; `chmod 600` is documented in research.md but absent from ACs and runbook scope. | Add to runbook and optionally to ACs: Vault token file must be `chmod 600` or stricter. |
| M5 | Medium | Cross-Feature Deps | **`target_repos` is empty in feature.yaml.** If IngressRoute or gateway changes are in scope, the target repo must be populated in TechPlan. | Not blocking for PrePlan; must be resolved in TechPlan scoping. |
| L1 | Low | Assumptions | **Hermes Agent install uses unversioned `curl | bash`.** No version pin means upstream breaking changes are silent. | Note Hermes version in runbook; include `hermes --version` output in AC-1 verification. |
| L2 | Low | Coverage Gaps | **No rollback procedure documented.** | Add to runbook: `rm -rf ~/.hermes/` is the full uninstall. |

---

## Accepted Risks

**H1 resolved — Gateway endpoint confirmed:** Install target is a dedicated VM inside `trantor.internal`. The live endpoint `https://ollama.trantor.internal/v1` is confirmed working (terminal history). Hermes `base_url` will be `https://ollama.trantor.internal/v1`. This routes directly to Ollama, bypassing the terminus-inference-gateway Vault auth layer. **Accepted**: this is the correct initial install approach; routing through the inference-gateway is a BusinessPlan-phase scoping decision if desired.

**H2 resolved — Vault token N/A for initial install:** `ollama.trantor.internal` has no auth layer. No Vault policy or token is required for Hermes to reach it. Vault auth is deferred to any future path that routes through the inference-gateway.

**H3 resolved — IngressRoute not needed:** The dedicated VM uses the existing `ollama.trantor.internal` IngressRoute. No new IngressRoute is in scope for this feature. VM provisioning (Proxmox + Ansible) is a new prerequisite story for the dev phase; `terminus.infra` becomes a target repo.

**M1 resolved:** `ollama.trantor.internal` is confirmed as the direct Ollama endpoint (not the Go inference-gateway). This is intentional for the initial install scope.

---

## Party-Mode Challenge

**Inquisitor Greyfax (Business Analyst):** The product brief says "VM install" but never names which VM. Is this Todd's laptop? A dedicated homelab node? The worker running Ollama? A dev laptop outside the network can't reach `trantor.internal` without a VPN or tunnel — none of that is scoped. Name the install target machine before BusinessPlan.

**Perturabo (Architect):** H1 is the real problem here. Terminal history shows `https://ollama.trantor.internal` is live and working. The terminus-inference-gateway is a separate Go service that routes through Ollama and adds Vault auth on top. If Hermes points at `ollama.trantor.internal` directly, there's no auth enforcement — the entire Vault-token AC fails silently because Ollama has no auth layer. The clean path is: create an IngressRoute for the inference-gateway, always go through it. But if the gateway has no external route, this feature must create one. Make that call now.

**Watch-Captain Artemis (QA):** AC-3's smoke test is "Say hello" — that validates the HTTP transport but nothing about tool calling. Hermes Agent's value proposition is agentic execution via tool calls. If `qwen3:27b` doesn't have function calling enabled or the model template doesn't include it, `hermes chat` works but every agentic task silently fails. There's no AC testing an actual tool call. At minimum, run `hermes` with a file-write task and confirm it executes before declaring the feature complete.

---

### Blind-Spot Challenge Questions

1. **Which specific VM or machine is the install target?** Name it — this blocks BusinessPlan scoping.
2. **Does `ollama.trantor.internal` route through the terminus-inference-gateway or directly to Ollama?** The answer changes whether Vault token auth is enforced at all.
3. **Does `qwen3:27b` support tool calling?** Has `ollama show qwen3:27b` been run to verify the template includes function calling?
4. **Who creates the Vault token for Hermes, and what policy does it use?** Is there an existing inference consumer Vault policy or does one need to be created?
5. **Has any agentic task (not just chat) been tested against this gateway/model combination?** If not, tool calling is an unvalidated assumption underpinning the entire feature value proposition.
