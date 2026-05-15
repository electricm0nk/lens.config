---
feature: terminus-watchdog-llm-engine
doc_type: adversarial-review
phase: techplan
status: complete
verdict: pass-with-warnings
updated_at: 2026-05-14
reviewed_artifacts:
  - docs/terminus/watchdog/terminus-watchdog-llm-engine/architecture.md
---

# Adversarial Review: terminus-watchdog-llm-engine / techplan

**Reviewed:** 2026-05-14T00:00:00Z  
**Source:** phase-complete  
**Overall Rating:** pass-with-warnings

---

## Summary

The `architecture.md` for terminus-watchdog-llm-engine is substantive, well-scoped, and directly responsive to the BusinessPlan adversarial review findings. The additive extension approach (`watchdog/triage/` package, single integration point in `loop.py`) is the correct architectural posture for a safety-critical production service. All three BusinessPlan HIGH findings are resolved: OQ-2 (gateway Art.10) is explicitly closed (commit `b6d8ebd`), HIGH-2 (Temporal Elasticsearch dependency) is addressed in PRE-2 with specific ops steps, and MEDIUM-4 (never-touch definition) is fully specified in section 3.4. Constitutional hard gates Art.3, Art.7, Art.8, Art.9, Art.15, and Art.16 are applied and documented.

The review surfaces **no critical findings**. Two high findings remain: the 3-prerequisite dependency chain (PRE-1/2/3) lacks formal enforcement before E5 dev begins, and the in-memory `triage_run_id_map` has no eviction policy. Four medium findings are documented, including the unresolved BusinessPlan MEDIUM-2 carry-forward (silent auto-confirm graduation distinction) and the `asyncio.create_task()` wrapper exception handler gap.

**Verdict: PASS-WITH-WARNINGS.** FinalizePlan may proceed. The high findings must be converted to explicit SprintPlan stories — they are scope items, not blockers.

---

## Findings

### Critical

None.

---

### High

| # | Dimension | Finding | Recommendation |
|---|-----------|---------|----------------|
| H1 | Coverage Gaps | PRE-1, PRE-2, PRE-3 have no corresponding LENS story or initiative artifacts. Sprint 0 is advisory in the architecture but has no enforcement gate. If PRE-1 (gateway Art.10) or PRE-2 (Elasticsearch) slip without a formal story, E5 is blocked silently with no lifecycle signal. The architecture notes PRE-2 is "single-node, homelab scale" but does not characterize memory requirements for Elasticsearch 8.x (typically 2–4GB JVM heap). Existing k3s worker capacity for this deployment is unverified. | SprintPlan must include explicit stories for PRE-1 (target: `terminus-inference-gateway` repo), PRE-2 (target: `terminus.infra`, include memory/storage sizing), and PRE-3 (target: `terminus.infra`). Each story must have a lifecycle-tracked AC that blocks E5's sprint slot if incomplete. |
| H2 | Logic Flaws | `bot.triage_run_id_map` (in-memory `dict` mapping `run_id_hash → workflow_run_id`) has no eviction policy. It grows with every new escalate-tier alert and is never pruned. After 30 days (proposed Temporal retention window), workflows expire from Temporal but their entries remain in the bot's memory. In a high-alert-volume environment this is a slow memory leak; in any environment it is an eventually-incorrect lookup table. | Define an explicit eviction policy in architecture. Recommended: cull entries for workflow run IDs older than `temporal_retention_days` during startup reconciliation. Note this as a known operational characteristic in `runbook.md`. |

---

### Medium

| # | Dimension | Finding | Recommendation |
|---|-----------|---------|----------------|
| M1 | Coverage Gaps | BusinessPlan MEDIUM-2 carry-forward: the architecture's batch analysis design (section 3.5, graduation criteria) does not distinguish `HumanVerdict=auto-confirmed` from `HumanVerdict=correct`. The BusinessPlan review required that `auto-confirmed` count as neutral (not confirmed correct) for graduation. The graduation threshold in architecture/epics states `≥5 occurrences, ≥0.90 avg confidence, no incorrect verdicts` but does not exclude `auto-confirmed` from the correct-verdict count. | Update E2 acceptance criteria to require `HumanVerdict IN ('correct', 'approve')` (explicit human approval) for the graduation confidence window. `auto-confirmed` must be explicitly excluded from the numerator. Document in epics.md E2 story before SprintPlan. |
| M2 | Coverage Gaps | `asyncio.create_task()` in `loop.py` has no `add_done_callback` exception handler. The architecture guarantees `triage_dispatch()` "never raises," but a defensive done-callback that logs unexpected task exceptions is missing from the design and from the unit test layout (section 9). An exception escaping `triage_dispatch()` despite the guarantee will be silently swallowed by the event loop. | Add `task.add_done_callback(_log_triage_task_exception)` to the `create_task()` call site. Add a test case in `test_dispatch.py` that injects an unexpected exception and verifies the callback logs it without crashing the detection loop. |
| M3 | Assumptions | System prompt versioning is unspecified. Architecture states the prompt is "version-pinned by the module (not runtime-configurable in MVP)" but defines no version constant, changelog mechanism, or classification quality monitoring path. If production classification accuracy degrades, there is no lightweight path to identify which prompt version was active for which observations. | Add `TRIAGE_PROMPT_VERSION = "v1.0.0"` as a module-level constant in `prompt.py`. Include the version in `TriageResult` metadata and in the `INFO`-level triage log entry (NFR4). This closes the retroactive classification quality audit gap. |
| M4 | Assumptions | Never-touch ConfigMap live-reload behavior is unspecified. The architecture says the config is "loaded at startup" and "fail-closed if not parseable," but Helm/ArgoCD may update the ConfigMap in-place. If an operator changes the never-touch policy urgently (adding a new resource type), they may expect the change to take effect without a pod restart. The current design silently ignores the updated ConfigMap until restart. | State explicitly in architecture section 3.4 and `runbook.md`: "never-touch policy changes require pod restart to take effect. No live reload in MVP. Restart the watchdog deployment after updating `watchdog.neverTouch` in Helm values." |

---

### Low

| # | Dimension | Finding | Recommendation |
|---|-----------|---------|----------------|
| L1 | Complexity and Risk | OQ-3 defers pydantic version verification to SprintPlan but treats it as a trivial check. If `terminus.watchdog` currently pins pydantic v1, migrating to v2 is a breaking change affecting all existing models (discriminator syntax, validator decorators, field aliases all changed). The architecture uses `BaseModel`, `Field`, `model_validate_json()` — all pydantic v2 APIs. A v1→v2 migration would add scope comparable to a full epic. | Before SprintPlan begins, run `grep -r "pydantic" pyproject.toml` in `terminus.watchdog`. If v1 is pinned, add a separate migration story to Sprint 1 and re-scope the sprint. Do not treat pydantic version as a SprintPlan footnote. |
| L2 | Complexity and Risk | Temporal custom search attribute types are irreversible. Once registered on the namespace (`tctl namespace search-attribute add`), attribute types cannot be changed without deleting and recreating the namespace (destroying all workflow history). The 8 attributes defined in PRE-2 should be treated as durable schema decisions, not provisional choices. | Mark PRE-2 ops story as requiring schema sign-off before execution. Add a note in architecture section 2.2: "Search attribute types are permanent for the lifetime of the Temporal namespace. Type choices for `LLMConfidence` (Double), `ActionTier` (Keyword), etc. require explicit approval before registration." |

---

## Accepted Risks

None at this time. All findings are tracked above. The operator may accept H2 (map eviction) for MVP homelab scale; if so, record that acceptance explicitly in SprintPlan with a note in `runbook.md`.

---

## Party-Mode Challenge

**Rhys (SRE/Platform):** The entire feature hinges on Sprint 0 delivering PRE-1, PRE-2, and PRE-3 before any E5 work begins. But Sprint 0 is informal — there are no LENS stories for it, no lifecycle gate that blocks E5 if it doesn't complete. What happens if PRE-2 (Elasticsearch) gets deprioritized because the k3s cluster is running a GPU workload and there aren't enough resources? You've described 8 custom search attributes that need Elasticsearch to be queryable, and a batch analysis story (E2) built on top of that. If PRE-2 ships in Sprint 3 instead of Sprint 0, E2 becomes a blocking scope increase in the final sprint. The sequencing needs teeth, not an advisory table.

**Mira (Security/Compliance):** I want to understand the `INFERENCE_GATEWAY_TOKEN` flow end-to-end under failure. The ESO-managed secret rotates on a schedule. When it rotates, the existing `TriageClient` instance still holds the old token. The `httpx` client is initialized once with `Authorization: Bearer {old_token}`. All classify() calls start returning 401s from the gateway. What does `classify()` return on a 401? The architecture says it handles `httpx.ConnectError` and `httpx.TimeoutException` — it doesn't mention 401 or 403 HTTP status codes. If a 401 returns a non-empty response body, `TriageResponse.model_validate_json()` will fail validation, fall back to `needs-investigation / escalate` — but the gateway token is now silently broken. Every novel alert escalates indefinitely with zero LLM insight. How does the operator know the token is the problem, not the LLM?

**Dev (Implementation):** The persistent Discord views section describes rebuilding `bot.triage_run_id_map` at startup by querying Temporal for all RUNNING workflows. How many workflows might that query return? If the watchdog has been running for 7 days with 20 novel alerts per day, that's 140 RUNNING workflows (silent-tier, waiting 7 days). Temporal's `list_workflows` with a filter is paginated. Does the startup reconciliation handle pagination? The architecture shows `bot.add_view(ApproveRejectView(...))` in `setup_hook()` but doesn't address the paginator. If the first page is processed and the startup completes before the second page loads, views for those workflows are never registered. Escalate-tier buttons in Discord would be silently broken for those workflow IDs.

---

## Gaps You May Not Have Considered

1. **401/403 handling from the inference gateway is absent from the architecture.** The gateway token is ESO-managed and rotates. The architecture's error handling covers network failures only. An expired or rotated token produces an HTTP error response, not an httpx exception — the safe fallback path is not triggered by default, and the failure mode is silent.

2. **Temporal `list_workflows` pagination is unaddressed in the startup reconciliation procedure.** If more than one page of RUNNING TriageWorkflows exists, the startup re-registration may be incomplete, silently leaving Discord approve/reject buttons non-functional for a subset of escalation messages.

3. **PRE-1 (gateway Art.10) is a cross-service dependency with its own LENS lifecycle.** The architecture scopes it as a prerequisite ops story, but the inference gateway has its own feature lifecycle and its own constitutionally required review gate. If the gateway team (or solo operator wearing both hats) advances PRE-1 as a quick fix without a full lifecycle gate, the Art.10 compliance it delivers may not satisfy the constitution's evidence requirement. The architecture should describe what "PRE-1 complete" looks like in terms of observable behavior — not just what needs to be built.

4. **The `never-touch` config is mounted from a ConfigMap but the ConfigMap is generated from Helm values.** If a resource needs to be added to the never-touch list in an active incident, the operator must: update Helm values → commit → push → wait for ArgoCD sync → restart pod. This could take 5–10 minutes. In a never-touch enforcement scenario (e.g., watchdog has just misclassified a production database secret as `notify`), that is a long window. The architecture should document the emergency override path.

5. **The 24-hour escalation expiry workflow closes with `verdict="expired"` — but what happens to the Discord message?** The architecture describes signal handling when a user clicks Approve/Reject, but it does not describe what happens when the workflow expires on timeout. The embed stays active in Discord with functioning-looking buttons, but the workflow is now CLOSED. Subsequent button clicks would attempt to signal a closed workflow, which Temporal rejects. The interaction handler needs to handle `WorkflowNotFoundError` from Temporal and update the Discord embed to show "Expired" state.

---

## Open Questions Surfaced

| # | Question | Owner | Needed By |
|---|----------|-------|-----------|
| OQ-5 | How does `classify()` handle HTTP 401/403 from the inference gateway? Is the safe fallback triggered? | Dev | SprintPlan |
| OQ-6 | Does the startup Temporal `list_workflows` reconciliation handle pagination? What is the expected max RUNNING workflow count at steady state? | Dev | SprintPlan |
| OQ-7 | What does PRE-1 "complete" look like as an observable artifact? A constitution compliance check on the gateway feature? A passing integration test? | Todd / ops | before E5 dev story |
| OQ-8 | What is the emergency path for adding a resource to the never-touch list during an active incident? | Todd / ops | SprintPlan |
| OQ-9 | What happens to the Discord embed when a 24-hour escalation workflow expires and a user subsequently clicks the (still-visible) Approve/Reject buttons? Is `WorkflowNotFoundError` handled in the interaction callback? | Dev | E4 story |
