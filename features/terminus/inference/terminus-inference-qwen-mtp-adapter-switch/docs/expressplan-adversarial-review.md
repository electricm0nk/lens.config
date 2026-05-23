# ExpressPlan Adversarial Review

**Feature:** terminus-inference-qwen-mtp-adapter-switch  
**Phase:** expressplan  
**Source:** phase-complete  
**Date:** 2026-05-23  
**Reviewer:** Lens Adversarial Review Conductor (party-mode)

---

## Artifacts Under Review

| Artifact | Present | Notes |
|---|---|---|
| `business-plan.md` | ✅ | Problem statement, goals, scope, risks, success criteria |
| `tech-plan.md` | ✅ | Adapter design, deployment, rollout, test strategy |
| `sprint-plan.md` | ✅ (supplementary) | 4 stories, TDD/BDD task sequences, DoD |

---

## Constitution Hard Gates Applied

| Article | Requirement | Status |
|---|---|---|
| Art. 2 | Phase artifacts non-empty before review | ✅ All three present and substantive |
| Art. 3 | `architecture.md` required | ⚠️ Deferred to FinalizePlan — documented intent in tech plan. Acceptable for expressplan gate. |
| Art. 4 | No data exfiltration | ✅ All inference local; GGUF download is pull-only |
| Art. 7 | TDD red-green discipline | ✅ Explicitly planned in QMTP-2 story tasks |
| Art. 8 | BDD acceptance criteria with Given/When/Then | ✅ All four stories include BDD scenarios |
| Art. 9 | No secrets in source | ✅ llamacpp adapter has no API key; Ollama unchanged |
| Art. 15 | AI Safety — safety/data-handling in architecture | ✅ Section present in tech plan |
| Art. 16 | Internal Infrastructure Primacy | ✅ llama.cpp is internal; no external inference dependency |
| Inference Art. 1 | Route profile manifest updated for new provider | ✅ QMTP-3 task includes route profile manifest update |
| Inference Art. 5 | Controlled rollout with rollback procedure | ✅ Rollout table + rollback procedure present in tech plan |

---

## Findings

### Critical (0)

None.

### High (1 — mitigated during review)

| ID | Finding | Status |
|---|---|---|
| A1 | llama.cpp commit/tag not locked — sprint could accept an older binary that silently ignores `--spec-type draft-mtp` | **Mitigated:** QMTP-1 now includes a pre-sprint verification task (step 0) to confirm binary version ≥ b9253 before GGUF download begins. Commit/tag to be recorded in infra runbook. |

### Medium (6 — addressed or documented)

| ID | Finding | Status |
|---|---|---|
| L1 | `llamacpp` adapter may duplicate OpenAI adapter code | **Accepted with rationale:** New package provides named registry identity and future telemetry hooks for MTP acceptance rate. Tech plan updated with explicit rationale. Not duplication. |
| L2 | Batch fallback must be declared as explicit policy, not silent default | **Accepted:** Tech plan routing config shows `llamacpp → ollama` chain in `batch` profile. Sprint QMTP-3 task 3 requires route profile manifest to declare fallback explicitly. |
| C1 | Architecture documentation (Art. 3) deferred to FinalizePlan | **Accepted:** Express track explicitly permits this deferral. FinalizePlan gate will enforce it. |
| R1 | Below-threshold benchmark decision rule undefined | **Resolved:** Sprint plan updated — re-bench with adjusted `--spec-draft-n-max` before deciding; fail gracefully to `.todo` + revert if still below 1.3×. |
| R2 | `llama-server` systemd unit lacked auto-restart policy | **Resolved:** `Restart=always, RestartSec=5s` added to tech plan and QMTP-1 systemd unit spec. |
| R3 | PVC free-space check before GGUF download | **Resolved:** QMTP-1 step 1 updated with pre-download space check (\< 30 GB free → abort). |
| A3 | DNS provisioning mechanism unspecified | **Resolved:** Internal bind9 A record confirmed as the mechanism. Tech plan and sprint plan both updated. |

### Low (3 — documented, non-blocking)

| ID | Finding | Status |
|---|---|---|
| L3 | Integration test environment may not be available if QMTP-1 runs in parallel | **Documented:** Sprint sequence shows QMTP-2 unit tests can proceed without a live server; integration test depends on QMTP-1 completing first. Sequential gate implicit in story dependency. |
| C2 | Streaming support not explicitly addressed | **Resolved:** Tech plan updated to state `Stream: false` hardcoded in `llamacpp` adapter (consistent with pipeline gate). |
| C3 | Warmup timeout may not cover full MTP GGUF load time | **Documented:** QMTP-4 should include warmup retry logic or a configurable warmup timeout. Flagged as a sprint implementation detail. |
| D1 | Ollama service changes during sprint could break fallback | **Documented:** Noted in sprint risks. Ollama is not being modified in this sprint. |
| D2 | `terminus-inference-qwen-warmup` repo Lens state unverified | **Accepted:** QMTP-4 is a small change in an existing repo; no full Lens feature initialization required for a minor story-sized change. |

---

## Blind-Spot Challenge Round

**Perturabo (Architect):** The plan produces a new adapter package instead of wrapping the existing OpenAI adapter. Rationale accepted: registry identity, MTP telemetry hooks. This is defensible if the new package explicitly re-uses the HTTP client pattern without copying the serialization logic. Ensure the implementation review verifies this during QMTP-2.

**Lord Commander Creed (PM):** Both HIGH blind spots are now mitigated. QMTP-1 adds a pre-sprint binary verification gate and a PVC space check. GGUF availability is confirmed. The sprint cannot be silently blocked on day one.

**Watch-Captain Artemis (QA):** Batch fallback must be declared as an explicit routing policy, not a silent default. QMTP-3 task 3 covers this. The BDD scenario for "Given llamacpp fails in batch, When a request arrives, Then fallback to ollama is per declared policy" must be present in the story AC when QMTP-3 is executed. The review conductor flags this as a sprint DoD enforcement item.

---

## User Responses to Blind-Spot Questions

| Question | Answer |
|---|---|
| New `llamacpp` adapter or reuse OpenAI adapter? | New named `llamacpp` package (as planned) |
| Bartowski MTP GGUF availability confirmed? | Yes — confirmed available |
| DNS mechanism for `llamacpp.trantor.internal`? | Internal DNS (bind9) A record |
| systemd restart policy? | `Restart=always` |
| Below-threshold benchmark decision rule? | Adjust `--spec-draft-n-max` and re-bench before deciding |

---

## Verdict

**PASS-WITH-WARNINGS**

The expressplan phase artifacts are substantive, internally consistent, and cover the required planning scope for the express track. All critical and high-severity findings have been mitigated or resolved. Medium and low-severity findings are documented and addressed in the updated sprint plan and tech plan.

Remaining warnings (all carry-forward to sprint execution):
1. **Art. 3 deferred:** `architecture.md` must be produced in FinalizePlan. FinalizePlan gate will hard-block if missing.
2. **Batch fallback explicit policy:** QMTP-3 must produce a route profile manifest with explicitly declared fallback, not a silent chain default.
3. **Warmup retry logic:** QMTP-4 should include a configurable warmup retry or timeout to account for MTP GGUF load time.

The phase may proceed to `expressplan-complete`. No blocker remains.

---

## Post-Review Action Required

The calling conductor (`lens-expressplan`) must:
1. Commit this review artifact via `git-orchestration-ops.py commit-artifacts`
2. Update `feature.yaml` phase to `expressplan-complete` via `lens-feature-yaml`
3. Signal `/finalizeplan` for the next phase
