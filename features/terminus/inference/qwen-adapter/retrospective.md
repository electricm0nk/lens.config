# Retrospective: Qwen Adapter Container Warmup

**Feature:** terminus-inference-qwen-adapter  
**Completed:** 2026-04-26  
**Track:** express  
**Domain/Service:** terminus/inference  

---

## What Went Well

- **Express track worked cleanly** — skipping full techplan/businessplan in favor of a combined expressplan delivered the planning artifacts quickly and with high confidence
- **Story decomposition was accurate** — all 11 stories (S1–S11) were correctly sized and unambiguous; no story required re-work due to unclear requirements
- **Pure stdlib warmup.py** — using stdlib-only for the warmup script eliminated dependency churn and simplified the Docker layer; the right call for a warmup-only container
- **Three-phase entrypoint pattern** (ollama → warmup → agent) was clean and correct; `set -e` + fail-fast design means container failure modes are clear and observable
- **Unit tests caught mocking approach early** — S8 spec had the wrong mock pattern (`requests` instead of `urllib`); the conductor caught this before delegation, preventing wasted subagent cycles
- **Adversarial review found a real bug** — curl missing from the Dockerfile for the readiness probe would have caused silent probe failures in K8s; caught before merge
- **Governance tooling worked end-to-end** — `next-ops.py`, `complete-ops.py`, `git-orchestration-ops.py` all functioning; the LENS lifecycle machinery handled phase transitions without manual intervention

## What Didn't Go Well

- **Phase field not updated alongside milestone** — after dev completion, the `milestone.dev-complete` timestamp was set but `phase: dev` was left unchanged in feature.yaml. This caused `/next` to recommend `/dev` again on the subsequent invocation. Minor but reveals that phase and milestone updates must be atomic.
- **PR #1 opened prematurely** — PR was opened at Sprint 1 close rather than at dev closeout. The dev skill spec calls for a single final PR created only after all gates pass. Opening early created a cosmetic inconsistency (PR title didn't match full scope). Low impact, but worth noting.
- **`gh pr edit` failure** — GitHub Projects Classic deprecation causes `gh pr edit` to return exit code 1 even on successful mutations. The title update failed silently. Should detect and work around this.
- **S8 story spec mismatch** — the unit test story referenced `requests.get` mock patterns, but `warmup.py` uses `urllib.request.urlopen`. Story spec was written before the S1 implementation was finalized. Planning artifacts should be reviewed for implementation consistency before the dev phase begins.

## Key Learnings

1. **Phase field and milestones must both be updated atomically** — writing `milestones.dev-complete` without updating `phase` causes lifecycle routing to lag one turn. Future: validate both fields in `complete-ops.py check-preconditions` and warn if milestone is set but phase hasn't advanced.

2. **PRs should only open at dev-closeout gate** — the single-PR model works well but requires discipline to not open early for partial sprints. The dev skill spec is explicit about this; follow it.

3. **Story specs can drift from implementation** — specs written during planning may assume implementation details that change during coding. For stdlib vs. third-party choices especially, the spec's technical examples should be marked as illustrative, not prescriptive.

4. **Warmup container pattern is reusable** — the three-phase entrypoint (server start → warmup → agent), readiness-via-file-marker, and KEEP_ALIVE=0 model is a clean pattern for any GPU-backed model container. Document this as a reusable pattern in team docs.

5. **curl in ollama image is not guaranteed** — `ollama/ollama:latest` is Debian-based but minimal; don't assume common CLI tools are present. Always check if probe/script tooling needs explicit installation.

## Metrics

- **Planned duration:** 1 sprint (expressplan estimate)
- **Actual duration:** ~1 session (~4 hours wall clock, 2 summarization cycles)
- **Stories completed:** 11/11 (S1–S11)
- **Total points delivered:** 55
- **Story sprints:** 3 (Sprint 1: core warmup; Sprint 2: K8s integration; Sprint 3: validation/checklist)
- **Code review findings — HIGH (fixed):** 1 (curl missing from Dockerfile)
- **Code review findings — MEDIUM (advisory):** 2 (socket timeout, missing main.py tests)
- **Code review findings — LOW (advisory):** 2 (KeyError propagation, ollama ENV var fragility)
- **Party-mode blind spots blocking:** 0
- **Bugs found post-merge:** 0 (PR not yet merged — pending integration test sign-off)
- **Phase field correction needed:** 1 (dev → dev-complete not applied atomically)

## Action Items

| # | Action | Owner | Priority |
|---|--------|-------|----------|
| 1 | Document `complete-ops.py` should warn when `milestones.dev-complete` is set but `phase != dev-complete` | LENS module | Low |
| 2 | Add `main.py` health endpoint unit tests in a follow-up story before production promotion | inference team | Medium |
| 3 | Document the three-phase warmup container pattern in team architecture docs as a reusable template | tech-writer | Low |
| 4 | Investigate `gh pr edit` exit code 1 on Projects Classic deprecation — file issue or add `--no-projects` workaround | platform | Low |
| 5 | Before S9 integration test execution: confirm `inference` namespace exists and StorageClass name for PVC | ops | High (pre-deploy) |
