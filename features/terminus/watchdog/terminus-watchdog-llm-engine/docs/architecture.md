---
feature: terminus-watchdog-llm-engine
doc_type: architecture
status: draft
updated_at: 2026-05-13T15:00:00Z
constitution_gates_applied:
  - Art.3 (architecture required — this document)
  - Art.7 (TDD red-green: component boundaries designed for unit isolation)
  - Art.8 (BDD: architecture maps directly to PRD acceptance criteria)
  - Art.9 (security-first: credential scrubbing, secret loading, never-touch)
  - Art.15 (AI safety: never-touch enforced in code, Temporal as authority)
  - Art.16 (internal infra primacy: all inference via gateway, no direct cloud calls)
stepsCompleted: [step-01-init, step-02-context, step-03-starter, step-04-decisions, step-05-patterns, step-06-structure, step-07-validation, step-08-complete]
---

# Architecture — Watchdog LLM Engine

**Feature:** `terminus-watchdog-llm-engine`  
**Service:** `terminus/watchdog`  
**Owner:** Todd Hintzmann  
**Date:** 2026-05-13  
**Status:** Draft — pending TechPlan adversarial review

---

## 1. Architectural Overview

### Context

terminus-watchdog MVP1 is a production Python 3.12 asyncio service running 11 hardcoded `BaseDetector` implementations. The detection loop (`watchdog/loop.py`) polls every detector, applies suppression and cooldown rules, and routes alerts to Discord via `WatchdogBot`. This feature adds an LLM reasoning layer that is invoked when a novel alert arrives with no matching hardcoded detector pattern, and a Temporal-based observation store for human-in-the-loop feedback.

### Architectural Approach: Additive Extension

The LLM engine is added as a new `watchdog/triage/` package. Existing code paths are not modified beyond a single integration point in `loop.py`. All new components are:
- **Unit-testable in isolation** (Art.7 compliance)
- **Non-blocking to the detection loop** (asyncio `Semaphore` gates LLM concurrency)
- **Fail-safe** (LLM path failure never degrades MVP1 alert routing)

### Component Map

```
watchdog/
  triage/
    __init__.py
    schema.py          # TriageRequest, TriageResponse Pydantic models
    client.py          # TriageClient: httpx → inference gateway
    workflow.py        # TriageWorkflow Temporal workflow + activities
    dispatch.py        # triage_dispatch(): novel alert → triage → route
    scrubber.py        # credential scrubbing (Art.9)
    never_touch.py     # never-touch config loader + enforcement (Art.15)
    prometheus.py      # watchdog_llm_triage_calls_total, _latency_seconds
  discord/
    triage_views.py    # (new) ApproveRejectView persistent views
  config.py            # (extend) INFERENCE_GATEWAY_URL, INFERENCE_GATEWAY_TOKEN
                       #           NEVER_TOUCH_CONFIG, LLM_MAX_CONCURRENT
```

### Integration Point in `loop.py`

A single call site is added after a detector returns alerts. For each alert that matches no suppression key and has `remediation_available=False` (novel/unclassified), the loop calls `triage_dispatch()` under the asyncio `Semaphore`. The existing routing path is unchanged for all suppressed, cooldown, and known alerts.

---

## 2. Prerequisites (must complete before dev starts)

### PRE-1 — Inference Gateway Art.10 (DEP-1, TechPlan prerequisite)

The terminus-inference-gateway must implement Art.10 observability (route id, duration, token counts, outcome, cost tag per request) before this feature enters the dev phase.

**Story (to be authored in SprintPlan):**
- Target repo: terminus-inference-gateway
- Scope: Add `X-Route-Id`, `X-Request-Duration-Ms`, `X-Input-Tokens`, `X-Output-Tokens`, `X-Outcome`, `X-Cost-Tag` response headers (or Prometheus metrics) to all `/v1/chat/completions` responses
- Acceptance gate: Art.10 compliance verified by constitution check on the inference gateway feature before terminus-watchdog-llm-engine enters dev

**Critical path:** Blocks E5 dev story. E4 and E2 can be developed in parallel once E5 is unblocked.

### PRE-2 — Elasticsearch Visibility Backend for Temporal (DEP-4)

Temporal custom search attributes (required by Option B — E2 batch analysis) need an Elasticsearch-backed Visibility API.

**Ops steps (terminus-infra story):**
1. Deploy Elasticsearch 8.x to the k3s cluster (single-node, homelab scale)
2. Configure Temporal server to use Elasticsearch as the Visibility backend (`TEMPORAL_VISIBILITY_ADDRESS`)
3. Restart Temporal server components
4. Register search attributes via `tctl`:
   ```
   tctl --namespace default namespace search-attribute add \
     --name LLMClassification --type Keyword \
     --name LLMConfidence --type Double \
     --name ActionTier --type Keyword \
     --name PatternId --type Keyword \
     --name AlertSource --type Keyword \
     --name HumanVerdict --type Keyword \
     --name PatternGraduated --type Bool \
     --name AlertTs --type Datetime
   ```
5. Smoke test: start a test workflow, set a search attribute, verify `tctl workflow list` filter returns it

**Critical path:** Blocks E2 batch analysis queries. E5, E1, E4 are not blocked by PRE-2.

### PRE-3 — Terminus-Infra Triage Routing Profile (DEP-2)

The `triage` routing profile must exist in the inference gateway's `ROUTING_CONFIG_PATH` before E5 can be tested in production.

**Ops story (terminus-infra):** Add `triage` profile to `ROUTING_CONFIG_PATH` config. Profile should route to local Qwen as primary, Anthropic as secondary. Merge before E5 dev story is worked.

---

## 3. New Package: `watchdog/triage/`

### 3.1 `schema.py` — Pydantic Models

```python
class AlertContext(BaseModel):
    """Context bundle sent to the inference gateway. Scrubbed of credentials (Art.9)."""
    alert_text: str
    loki_snippets: list[str]
    resource_state: dict[str, Any]
    source: Literal["argocd", "temporal", "k8s", "loki"]
    pattern_id: str
    alert_ts: datetime

class TriageResponse(BaseModel):
    """Structured LLM classification output. Validated against this schema."""
    classification: Literal[
        "transient-glitch",
        "known-degraded-dependency",
        "config-drift",
        "resource-pressure",
        "needs-investigation",
        "no-action",
    ]
    confidence: float = Field(ge=0.0, le=1.0)
    hypothesis: str
    action_tier: Literal["silent", "notify", "escalate", "never-touch"]
    evidence: list[str]
    degraded: bool = False  # true if gateway provider fallback was active

class TriageResult(BaseModel):
    """Internal result wrapper including gateway call metadata."""
    response: TriageResponse
    workflow_run_id: str
    latency_ms: int
    gateway_unreachable: bool = False
```

**Fallback behavior:** On Pydantic `ValidationError`, `TriageClient` returns a safe fallback:
```python
TriageResponse(
    classification="needs-investigation",
    confidence=0.0,
    hypothesis="LLM response could not be parsed",
    action_tier="escalate",
    evidence=[],
)
```

### 3.2 `client.py` — TriageClient

**Responsibilities:** Send `AlertContext` to the inference gateway and return a `TriageResponse`.

**Key implementation decisions:**

| Decision | Value | Rationale |
|---|---|---|
| HTTP client | `httpx.AsyncClient` | Existing dependency; async native |
| Timeout | `httpx.Timeout(5.0)` | 5-second fast-fail (not httpx default 30s); NFR-1 |
| Auth | `Authorization: Bearer {INFERENCE_GATEWAY_TOKEN}` | ESO-managed secret |
| Request body | `POST /v1/chat/completions` with `response_format: {"type": "json_object"}`, `temperature: 0.1` | Deterministic classification |
| Response parsing | Pydantic `TriageResponse.model_validate_json(choice.message.content)` | Strict validation |
| Error handling | `httpx.ConnectError`, `httpx.TimeoutException` → `gateway_unreachable=True` fallback | Never raise from classify() |

**`classify()` signature:**
```python
async def classify(self, ctx: AlertContext) -> TriageResult:
    """Call inference gateway. Never raises. Returns safe fallback on any failure."""
```

**Secret loading (Art.9):** `INFERENCE_GATEWAY_TOKEN` is loaded once at startup from the environment (ESO-managed). It is never logged at any level. The token is stored as an instance attribute, not serialized into any log record, error message, or Temporal workflow payload.

**System prompt:** Stored in `triage/prompt.py` as a module-level constant. Loaded at `TriageClient` init. The prompt:
- Contains few-shot examples of valid `TriageResponse` JSON
- Contains the full classification enum and action tier enum with descriptions
- Does NOT reference production secret paths, vault paths, or internal URLs
- Is version-pinned by the module (not runtime-configurable in MVP)

### 3.3 `scrubber.py` — Credential Scrubbing (Art.9)

**Responsibilities:** Scan `AlertContext` fields for patterns that indicate credential material before the context bundle is sent to the gateway.

**Scrub patterns (compiled regex at module load):**

| Pattern | Rationale |
|---|---|
| `(?i)(password\|secret\|token\|api_key)\s*[:=]\s*\S+` | Generic key=value credentials |
| `(?i)vault/v1/secret/[^\s]+` | Vault secret paths |
| `Bearer\s+[A-Za-z0-9\-._~+/]+=*` | Bearer tokens |
| `[A-Za-z0-9+/]{40,}={0,2}` (base64-looking) | Possible base64 secrets |

**Scrub behavior:** Replace matched content with `[REDACTED]`. Scrubbing mutates a copy of the `AlertContext` — the original is not modified. If the scrubber raises an exception, the context is blocked from sending and the alert is routed as a `needs-investigation` escalation (fail-closed, not fail-open).

### 3.4 `never_touch.py` — Never-Touch Resource Enforcement (Art.15)

**Responsibilities:** Load the never-touch resource list from config and enforce it as a hard boundary before any action tier routing.

**Config structure (Helm values → ConfigMap):**

```yaml
# Helm values: watchdog.neverTouch
neverTouch:
  - resourceType: "PersistentVolumeClaim"
    namespacePattern: ".*"        # regex
    namePattern: ".*"             # matches all PVCs
  - resourceType: "Secret"
    namespacePattern: ".*"
    namePattern: ".*"
  - resourceType: "Deployment"
    namespacePattern: "production"
    namePattern: "terminus-temporal.*"
```

**Enforcement contract:**
1. `NeverTouchGuard.is_never_touch(resource_type, namespace, name) -> bool` is called inside `TriageWorkflow` before any action tier is acted upon
2. If `is_never_touch() == True`, the action tier is **overridden to `never-touch` regardless of LLM output**
3. The override is logged at WARN level with the resource identity
4. This is enforced in Python code; no LLM instruction can bypass it (Art.15)

**Config loading:** The ConfigMap is mounted at a path defined by `NEVER_TOUCH_CONFIG` env var. Loaded at startup. If the file cannot be parsed, the watchdog startup fails with a clear error (fail-closed — no never-touch config = no LLM triage).

### 3.5 `workflow.py` — TriageWorkflow (Temporal)

**Responsibilities:** Orchestrate the full triage lifecycle for one alert observation.

**Workflow ID format:**
```
triage:{alert_source}:{pattern_id}:{alert_ts_unix_ms}
```
Example: `triage:argocd:argocd-stuck-sync:1747181640000`

This format is deterministic and stable across pod restarts. If a workflow with this ID already exists (RUNNING or CLOSED), `start_workflow` with `id_reuse_policy=AllowDuplicate` handles it idempotently.

**Search attributes (set at workflow start via `upsert_search_attributes`):**

| Attribute | Type | Set At |
|---|---|---|
| `LLMClassification` | Keyword | workflow start |
| `LLMConfidence` | Double | workflow start |
| `ActionTier` | Keyword | workflow start |
| `PatternId` | Keyword | workflow start |
| `AlertSource` | Keyword | workflow start |
| `HumanVerdict` | Keyword | on signal receipt |
| `PatternGraduated` | Bool | batch analysis post-graduation |
| `AlertTs` | Datetime | workflow start |

**Workflow structure:**

```python
@workflow.defn
class TriageWorkflow:
    def __init__(self):
        self._verdict: str | None = None

    @workflow.run
    async def run(self, input: TriageWorkflowInput) -> TriageWorkflowResult:
        # 1. Set initial search attributes
        workflow.upsert_search_attributes(...)
        
        # 2. Branch by action_tier
        if input.action_tier == "silent":
            # Wait 7 days for contradiction; auto-confirm if none received
            try:
                await workflow.wait_condition(
                    lambda: self._verdict is not None,
                    timeout=timedelta(days=7),
                )
                verdict = self._verdict
            except asyncio.TimeoutError:
                verdict = "auto-confirmed"
            workflow.upsert_search_attributes({"HumanVerdict": verdict})
            return TriageWorkflowResult(verdict=verdict)
        
        elif input.action_tier in ("notify", "escalate", "never-touch"):
            if input.action_tier == "escalate":
                # Wait 24h for human signal
                try:
                    await workflow.wait_condition(
                        lambda: self._verdict is not None,
                        timeout=timedelta(hours=24),
                    )
                    verdict = self._verdict
                except asyncio.TimeoutError:
                    verdict = "expired"
            else:
                verdict = "no-action"
            workflow.upsert_search_attributes({"HumanVerdict": verdict})
            return TriageWorkflowResult(verdict=verdict)

    @workflow.signal
    def human_response(self, verdict: str) -> None:
        """Signal from Discord Approve/Reject interaction."""
        if self._verdict is None:  # idempotency guard
            self._verdict = verdict
```

**Idempotency:** The `human_response` signal only sets `_verdict` if it is currently `None`. Duplicate signals (two users clicking the same button) are ignored silently.

**Temporal SDK version requirement:** `temporalio >= 1.3.0` (signal with payload, `upsert_search_attributes`). This must be pinned in `pyproject.toml`.

### 3.6 `dispatch.py` — Triage Dispatch Coordinator

**Responsibilities:** Coordinate the full flow for a single novel alert: scrub → classify → check never-touch → start TriageWorkflow → route to Discord.

**`triage_dispatch()` contract:**

```python
async def triage_dispatch(
    alert: Alert,
    triage_client: TriageClient,
    temporal_client: TemporalClient,
    bot: WatchdogBot,
    semaphore: asyncio.Semaphore,
    never_touch_guard: NeverTouchGuard,
    metrics: TriageMetrics,
) -> None:
    """
    Full triage pipeline for a novel alert. Never raises.
    Falls back to MVP1-style notify on any failure.
    """
```

**Pipeline steps:**
1. Acquire `semaphore` (max 3 concurrent LLM calls — non-blocking wait)
2. Build `AlertContext` from `Alert`
3. Run `scrubber.scrub(ctx)` → scrubbed context
4. Call `triage_client.classify(scrubbed_ctx)` → `TriageResult`
5. If `gateway_unreachable`: post MVP1-style Discord embed + return (no Temporal workflow)
6. Apply `never_touch_guard.is_never_touch()` → override action tier if needed
7. Start `TriageWorkflow` via `temporal_client.start_workflow(TriageWorkflow, ...)`
8. If Temporal start fails: route as `notify` via Discord (never silently drop)
9. Route Discord embed per action tier:
   - `silent` → no Discord message
   - `notify` → embed only, no buttons
   - `escalate` → embed + `ApproveRejectView(workflow_run_id=...)`
   - `never-touch` → embed (escalation only), no Approve button
10. Release `semaphore`

### 3.7 `discord/triage_views.py` — Persistent Discord Views

**Responsibilities:** Render and handle Discord interactive buttons for `escalate`-tier triage workflows.

**`custom_id` format (MEDIUM-1 resolution):**

```
{action}:{run_id_hash}
```

Where:
- `action` ∈ `{"approve", "reject"}` (7 chars max)
- `run_id_hash` = first 16 hex chars of `sha256(workflow_run_id.encode())` (16 chars)
- Separator: `:` (1 char)
- **Total: 24 chars maximum** — well within Discord's 100-char limit

The mapping from `run_id_hash` to `workflow_run_id` is held in an in-memory `dict` on the `WatchdogBot` instance (`bot.triage_run_id_map: dict[str, str]`). This map is rebuilt at startup by querying Temporal for all RUNNING TriageWorkflows.

**`ApproveRejectView` design:**

```python
class ApproveRejectView(discord.ui.View):
    def __init__(self, workflow_run_id: str, bot: WatchdogBot):
        super().__init__(timeout=None)  # persistent — survives pod restart
        self._run_id = workflow_run_id
        self._bot = bot
        run_id_hash = _hash_run_id(workflow_run_id)
        self.add_item(discord.ui.Button(
            label="Approve",
            style=discord.ButtonStyle.success,
            custom_id=f"approve:{run_id_hash}",
        ))
        self.add_item(discord.ui.Button(
            label="Reject",
            style=discord.ButtonStyle.danger,
            custom_id=f"reject:{run_id_hash}",
        ))
```

**Startup re-registration:**

At `WatchdogBot.setup_hook()`, query Temporal for all RUNNING TriageWorkflows (via `list_workflows` with `ExecutionStatus=Running` filter). For each open workflow, call `bot.add_view(ApproveRejectView(...))` to re-register the persistent view. This restores button functionality for pre-restart escalation messages.

**Interaction handler:**

On button click: look up `workflow_run_id` from `bot.triage_run_id_map[run_id_hash]`, call `temporal_client.get_workflow_handle(workflow_run_id).signal("human_response", verdict)`, then update the Discord message to show the resolved state embed.

---

## 4. Integration: `loop.py` Changes

### Novel Alert Detection

The detection loop gains a single new concept: **novel alerts** are those where:
1. The alert is not suppressed
2. The alert is not in cooldown
3. The alert's `pattern` is not in the set of hardcoded detector `pattern_id` values
4. `remediation_available=False` (LLM triage, not direct remediation)

The `triage_dispatch()` function is called for novel alerts. The existing routing path runs unchanged for all other alerts.

### Loop Change Sketch

```python
# In run_detection_cycle(), after the existing suppress/cooldown checks:
for alert in alerts:
    if _is_novel(alert, hardcoded_pattern_ids):
        # Non-blocking: fire and forget under semaphore
        asyncio.create_task(
            triage_dispatch(alert, triage_client, temporal_client, bot,
                            triage_semaphore, never_touch_guard, triage_metrics)
        )
    else:
        # Existing MVP1 routing path — unchanged
        await bot.post_alert(alert)
        state.record_alert(alert)
```

`asyncio.create_task()` ensures the dispatch runs concurrently and the detection loop continues polling.

---

## 5. Config Changes (`config.py`)

New fields added to `Settings`:

| Field | Env Var | Type | Default | Description |
|---|---|---|---|---|
| `inference_gateway_url` | `INFERENCE_GATEWAY_URL` | `str` | (required) | Gateway base URL |
| `inference_gateway_token` | `INFERENCE_GATEWAY_TOKEN` | `str` | (required) | Auth token (ESO-managed) |
| `never_touch_config` | `NEVER_TOUCH_CONFIG` | `str` | `/etc/watchdog/never_touch.yaml` | ConfigMap mount path |
| `llm_max_concurrent` | `LLM_MAX_CONCURRENT` | `int` | `3` | Semaphore limit |
| `llm_confidence_warn_threshold` | `LLM_CONFIDENCE_WARN_THRESHOLD` | `float` | `0.70` | Discord ⚠️ threshold |

`INFERENCE_GATEWAY_TOKEN` must be masked in `__repr__` (same pattern as `DISCORD_BOT_TOKEN` in existing `Settings`).

---

## 6. New ESO Secret

A new `ExternalSecret` is added to the Helm chart targeting a Vault path:
- Secret name: `watchdog-inference-token`
- Key: `INFERENCE_GATEWAY_TOKEN`
- Vault path: `secret/terminus/watchdog/inference-gateway-token`

---

## 7. Helm Chart Changes

### New values keys:

```yaml
triage:
  enabled: true
  inferenceGatewayUrl: "https://inference.trantor.internal"
  maxConcurrent: 3
  confidenceWarnThreshold: 0.70
  neverTouch:
    - resourceType: "PersistentVolumeClaim"
      namespacePattern: ".*"
      namePattern: ".*"
    - resourceType: "Secret"
      namespacePattern: ".*"
      namePattern: ".*"
```

The `neverTouch` list is rendered into a `ConfigMap` mounted at `/etc/watchdog/never_touch.yaml`.

---

## 8. Prometheus Metrics (`prometheus.py`)

Two new metrics added to the existing `watchdog/health.py` Prometheus registry:

```python
watchdog_llm_triage_calls_total = Counter(
    "watchdog_llm_triage_calls_total",
    "Total LLM triage calls by classification and action tier",
    ["classification", "action_tier"],
)

watchdog_llm_triage_latency_seconds = Histogram(
    "watchdog_llm_triage_latency_seconds",
    "LLM triage call latency in seconds",
    buckets=[0.5, 1.0, 2.0, 5.0, 10.0, 30.0],
)
```

These are incremented in `dispatch.py` after the gateway call completes. The existing `/metrics` endpoint (port 9090) exposes them without change.

---

## 9. Test Architecture (Art.7 — TDD Red-Green)

### Unit Test Layout

```
tests/unit/triage/
  __init__.py
  test_schema.py          # TriageResponse validation, fallback behavior
  test_client.py          # TriageClient: valid response, parse error, timeout, unreachable
  test_scrubber.py        # credential pattern matching
  test_never_touch.py     # guard logic, config loading, override behavior
  test_dispatch.py        # full dispatch pipeline: mock client + temporal + bot
  test_views.py           # custom_id hash format, view registration
tests/unit/discord/
  test_triage_views.py    # ApproveRejectView signal routing
```

### Timeout Mocking Strategy (MEDIUM-1 resolution — E5 BDD)

The E5 BDD test for 5-second gateway fast-fail uses `pytest-mock` + `unittest.mock.AsyncMock` to mock `httpx.AsyncClient.post()` to raise `httpx.TimeoutException` immediately (no real wall-clock wait). Tests run at normal speed.

```python
# Example: test_client.py
async def test_gateway_timeout_fast_fail(mocker):
    mock_post = mocker.patch("httpx.AsyncClient.post", side_effect=httpx.TimeoutException("timeout"))
    client = TriageClient(url="https://test", token="test-token")
    result = await client.classify(sample_alert_context())
    assert result.gateway_unreachable is True
    assert result.response.classification == "needs-investigation"
    assert result.response.action_tier == "escalate"
    mock_post.assert_called_once()
```

Temporal `wait_condition` timeout tests (7-day silent auto-confirm, 24-hour escalation expiry) use `temporalio.testing.WorkflowEnvironment.start_time_skipping()` to fast-forward time without wall-clock delay.

### BDD Acceptance Test Map (Art.8)

All PRD BDD acceptance criteria map to test files:

| PRD Story | BDD Scenario | Test File |
|---|---|---|
| E5 | Gateway healthy → valid TriageResponse | `test_client.py::test_classify_success` |
| E5 | Gateway returns schema-invalid JSON | `test_client.py::test_classify_parse_error_fallback` |
| E5 | Gateway unreachable, 5s fast-fail | `test_client.py::test_gateway_timeout_fast_fail` |
| E5 | Gateway degraded=true flag | `test_client.py::test_classify_degraded_flag` |
| E1 | action_tier=escalate → Discord embed + buttons | `test_dispatch.py::test_dispatch_escalate` |
| E1 | action_tier=silent → no Discord message | `test_dispatch.py::test_dispatch_silent` |
| E1 | action_tier=never-touch → embed, no Approve | `test_dispatch.py::test_dispatch_never_touch` |
| E1 | Alert storm (25 alerts) → Semaphore queues | `test_dispatch.py::test_dispatch_semaphore_queue` |
| E1 | Temporal start fails → notify fallback | `test_dispatch.py::test_dispatch_temporal_failure` |
| E1 | Pod restart → views re-registered | `test_views.py::test_startup_view_reregistration` |
| E4 | Approve signal → workflow verdict=approve | `test_views.py::test_approve_button_signal` |
| E4 | Reject signal → no remediation | `test_views.py::test_reject_button_signal` |
| E4 | Duplicate approval → idempotency check | `test_workflow.py::test_duplicate_signal_idempotency` |
| E4 | 24h expiry → verdict=expired | `test_workflow.py::test_escalation_expiry` |
| E2 | Auto-confirm after 7 days | `test_workflow.py::test_silent_auto_confirm_7d` |
| E2 | Batch analysis query returns qualifying workflows | `test_workflow.py::test_batch_analysis_query` |

---

## 10. Dependency Versions

New dependencies to add to `pyproject.toml`:

| Package | Version Constraint | Use |
|---|---|---|
| `temporalio` | `>=1.3.0` | Signal with payload, upsert_search_attributes |
| `pydantic` | `>=2.0` (already present?) | TriageRequest/Response models |

Verify existing `pyproject.toml` for current `temporalio` and `pydantic` versions. If `temporalio` is not yet present, add it. If `pydantic` v1 is pinned, a migration scope may be needed — flag in SprintPlan.

---

## 11. Sprint Sequencing

Based on the critical path (PRE-1 blocks E5; PRE-2 blocks E2; PRE-3 blocks E5 prod test):

| Sprint | Stories |
|---|---|
| Sprint 0 (infra) | PRE-1 (gateway Art.10), PRE-2 (Elasticsearch), PRE-3 (triage routing profile) |
| Sprint 1 | E5 (TriageClient + gateway wiring), E1-core (dispatch + loop integration) |
| Sprint 2 | E4 (Discord persistent views + Temporal signal), E1-views (startup re-registration) |
| Sprint 3 | E2 (batch analysis tooling), E1-metrics (Prometheus), hardening + load test |

E3 is post-MVP. Not scheduled.

---

## 12. Open Questions (TechPlan scope)

| # | Question | Owner | Needed By |
|---|---|---|---|
| OQ-3 | What is the current pinned `temporalio` SDK version in `pyproject.toml`? Is it ≥ 1.3.0? | Dev | SprintPlan |
| OQ-4 | What is the current Temporal namespace retention policy? Is 30-day feasible? | Todd / ops | SprintPlan |
