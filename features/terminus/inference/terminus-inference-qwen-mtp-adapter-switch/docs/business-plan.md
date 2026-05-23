# Business Plan: qwen-mtp-adapter-switch

**Feature:** terminus-inference-qwen-mtp-adapter-switch  
**Domain/Service:** terminus / inference  
**Track:** express  
**Phase:** expressplan  
**Date:** 2026-05-23

---

## Problem Statement

The terminus inference service currently serves Qwen3-35B-A22B (MoE) through Ollama on k3s-worker-07 (RTX 3090, 24 GB VRAM). Benchmarking conducted on 2026-05-23 confirmed the current configuration delivers measurable generation throughput, but significant speed headroom remains untapped.

Recent research findings establish that llama.cpp ≥ b9253 with Multi-Token Prediction (MTP) draft models enables speculative decoding natively at the GGUF level via `--spec-type draft-mtp --spec-draft-n-max 3`. This eliminates the Ollama management layer for the Qwen model and is expected to deliver approximately **1.5–2× generation speed improvement** with no model quality regression. MTP drafts are lightweight (Q4 ~1.2 GB, Q8 ~2.0 GB) and sit beside the main GGUF on the existing PVC.

The existing Ollama-managed configuration must be **preserved as a fallback** — both to satisfy the Internal Infrastructure Primacy governance requirement (Art. 16) and to allow rollback without redeployment if the MTP path proves operationally problematic.

---

## Target Users

| User | Need |
|---|---|
| Platform consumers via terminus gateway | Faster AI response latency on Qwen-routed requests |
| Todd (operator) | Reliable, observable inference with a clean rollback path |
| Downstream agents (kaylee, hermes) | Reduced wait time on inference-heavy workflows |

---

## Business Goals

1. **Speed**: Achieve ~1.5–2× token generation improvement on the Qwen3-35B-A22B model via MTP speculative decoding.
2. **Reliability**: MTP config auto-loads as primary on inference service restart; no manual intervention required.
3. **Reversibility**: Prior Ollama-based config remains intact and documented; rollback is a configuration swap, not a rebuild.
4. **Observability**: The active backend (MTP direct vs Ollama) is identifiable from service logs and/or the ArgoCD manifest state without accessing the pod directly.
5. **Governance compliance**: No new external dependencies introduced; llama.cpp is already built and present on the worker node.

---

## Scope

### In Scope

- Update the qwen adapter (in `terminus-inference-gateway` or equivalent qwen-specific config) to target llama.cpp binary directly via `--spec-type draft-mtp --spec-draft-n-max 3`
- Download and place the bartowski MTP-enabled Q4_K_M GGUF and its MTP draft GGUF onto the existing PVC storage path used by the inference service
- Configure autoload so the MTP llama.cpp path is the default serving backend for Qwen routes
- Document rollback procedure referencing the preserved Ollama config state
- Update route profile manifest for Qwen routes (new backend, same contract — Inference Art. 1)
- Verify llama.cpp version ≥ b9253 is present on k3s-worker-07 (already confirmed built as of 2026-05-23)

### Out of Scope

- Changes to the gateway API contract (no breaking changes to `/v1/` routes)
- Changes to other model routes (only Qwen MoE is affected)
- Replacing Ollama for other models
- Infrastructure provisioning (worker node, VRAM, PVC already provisioned)

---

## Dependencies

| Dependency | Status | Risk |
|---|---|---|
| llama.cpp ≥ b9253 on k3s-worker-07 | ✅ Already built and confirmed | None |
| bartowski MTP-enabled Q4_K_M GGUF | ⬜ Must be downloaded to PVC | Medium — download size ~20+ GB for main GGUF; MTP draft ~1.2 GB |
| MTP draft GGUF placement | ⬜ Must sit beside main GGUF | Low |
| Existing Qwen GGUF on PVC | ✅ Already present (confirmed via prior llama-bench runs) | None |
| Rollback config (Ollama) preserved | ✅ Not touching existing Ollama config | None |
| ArgoCD sync after config change | ⬜ Manifest update will trigger resync | Low |

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| MTP draft quality mismatch degrades output coherence | Low | Medium | Run smoke test against known prompts before promoting; compare outputs |
| VRAM pressure from main + draft GGUF together | Medium | Medium | Monitor GPU VRAM usage after enabling; draft is Q4 (~1.2 GB) so fits comfortably in 24 GB |
| llama.cpp process lifecycle not supervised by k8s | Medium | High | Use a supervised deployment pattern (pod with llama.cpp server binary) or validate watchdog covers it |
| Rollback path is untested | Low | High | Sprint must include a rollback smoke test task |

---

## Success Criteria

1. Qwen3-35B-A22B serves tokens through the MTP llama.cpp path at ≥ 1.5× the pre-switch benchmark throughput (measured via `llama-bench` or equivalent).
2. The inference gateway routes requests to the MTP path by default without operator intervention after a service restart.
3. Rollback to the Ollama-based path completes in under 5 minutes with no cluster reprovision.
4. No change to the gateway API contract observable by consumers.
5. ArgoCD reports the deployment as Synced and Healthy after the switch.
6. BDD acceptance tests pass for all Qwen route acceptance criteria (Art. 8).

---

## Constitution Compliance Notes

- **Art. 3 (Architecture Required):** `architecture.md` will be produced in FinalizePlan as this feature modifies the inference integration pattern (Ollama → llama.cpp direct).
- **Art. 4 (No Data Exfiltration):** All data flows remain local; the only new network path is the PVC-mounted GGUF download from HuggingFace/bartowski, which is a one-time pull.
- **Art. 15 (AI Safety Primacy):** Safety and data-handling section required in architecture; no prompt data leaves the local cluster.
- **Art. 16 (Internal Infrastructure Primacy):** llama.cpp is internally built; no external inference services introduced.
- **Inference Art. 5 (Controlled Rollout):** Rollback procedure must be documented before promotion.
