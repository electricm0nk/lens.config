---
feature: terminus-watchdog-llm-engine
doc_type: research
status: draft
updated_at: 2026-05-13
---

# Technical Research — terminus-watchdog-llm-engine

**Purpose:** Answer the 6 open questions raised in `brainstorm.md` and establish implementation-ready technical findings for the BusinessPlan and TechPlan phases.

**Sources:**
- `docs/terminus/inference/gateway/architecture.md`
- `docs/terminus/inference/provider-routing/architecture.md`
- `docs/terminus/watchdog/terminus-watchdog-agent/architecture.md`
- OpenAI Structured Outputs documentation (openai.com)
- ArgoCD Notifications Webhook documentation (argo-cd.readthedocs.io)
- Prior Watchdog LLM design session (2026-05-10)

---

## Research Question 1: How does watchdog consume the inference gateway?

### Finding

The terminus-inference-gateway exposes an OpenAI-compatible REST surface at `/v1/chat/completions`. Watchdog calls this endpoint directly using `httpx` (already a dependency). Provider selection, fallback, and budget enforcement are fully handled inside the gateway's `internal/routing/` library — the caller names a routing profile, not a provider.

**Consumption pattern:**

```python
# watchdog/llm/triage_client.py
import httpx
import os

GATEWAY_URL = os.environ["INFERENCE_GATEWAY_URL"]  # e.g. http://inference-gateway.terminus-inference.svc.cluster.local
GATEWAY_TOKEN = os.environ["INFERENCE_GATEWAY_TOKEN"]
TRIAGE_PROFILE = os.environ.get("TRIAGE_ROUTING_PROFILE", "triage")

async def classify(alert_text: str, context: str) -> dict:
    payload = {
        "model": TRIAGE_PROFILE,   # maps to a ROUTING_CONFIG_PATH profile
        "messages": [
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": f"Alert text:\n{alert_text}\n\nContext:\n{context}"}
        ],
        "response_format": {"type": "json_object"},
        "temperature": 0.1,
        "max_tokens": 512,
    }
    async with httpx.AsyncClient(timeout=30.0) as client:
        response = await client.post(
            f"{GATEWAY_URL}/v1/chat/completions",
            headers={"Authorization": f"Bearer {GATEWAY_TOKEN}"},
            json=payload,
        )
        response.raise_for_status()
        return response.json()["choices"][0]["message"]["content"]
```

**Key points:**
- Watchdog never specifies a provider name — the gateway routing profile resolves the provider chain (local Qwen → Anthropic → OpenAI fallback order is defined in the `ROUTING_CONFIG_PATH` YAML maintained by platform ops)
- The gateway returns an OpenAI-normalized response regardless of which provider served the request — watchdog sees identical response structure whether Qwen or OpenAI answered
- Provider degradation is surfaced in the gateway response's `degraded: true` field (from provider-routing spec); watchdog should log this field to the observation log for cost tracking
- Streaming is deferred to a future gateway release (returns 501). Watchdog must use non-streaming chat completions only — compatible with synchronous classification use case

### New Secret Required

The gateway uses Vault token lookup per request. Watchdog needs a new ESO-managed secret:

```
Vault path:     terminus/watchdog/inference-gateway-token
k8s Secret:     watchdog-inference-gateway-token
env var:        INFERENCE_GATEWAY_TOKEN
```

This follows the same ESO + Vault pattern as the existing `watchdog-discord-token` and `watchdog-argocd-token` secrets in the MVP1 chart.

### Gateway Routing Profile

Platform ops (separate story, not watchdog-owned) must add a `triage` profile to the gateway's `ROUTING_CONFIG_PATH` YAML:

```yaml
profiles:
  - name: triage
    workload_class: interactive
    providers:
      - name: qwen-local
        priority: 1
      - name: anthropic
        priority: 2
      - name: openai
        priority: 3
```

The watchdog implementation story for E5 is a wiring story — connect to the existing HTTP endpoint. No new adapter development is required in the watchdog service.

---

## Research Question 2: Optimal LLM prompt structure and output schema for triage

### Finding

**Output schema (JSON object mode):**

The gateway normalizes responses to OpenAI format. Since local Qwen models may not support strict JSON Schema enforcement (OpenAI's `response_format: json_schema` with `strict: true` requires specific model snapshots), watchdog uses `response_format: {"type": "json_object"}` with explicit schema instruction in the system prompt. This is compatible with all current gateway providers (Qwen, Anthropic tool use, OpenAI JSON mode).

```json
{
  "classification": "<enum string>",
  "confidence": 0.0,
  "hypothesis": "<one-sentence causal hypothesis>",
  "recommended_action": "<imperative sentence>",
  "action_tier": "<enum string>",
  "evidence": ["<observation 1>", "<observation 2>"]
}
```

**Classification enum values:**

| Value | Meaning |
|---|---|
| `transient-glitch` | Error likely self-resolves; monitor only |
| `known-degraded-dependency` | Upstream dependency issue; watchdog cannot remediate |
| `config-drift` | ArgoCD OOSSync or live drift; approve-to-sync is appropriate |
| `resource-pressure` | Pod OOM, eviction, CPU throttle |
| `needs-investigation` | LLM cannot classify confidently; human must inspect |
| `no-action` | Alert is noise; suppress recommended |

**Action tier enum values:**

| Value | Meaning |
|---|---|
| `silent` | Log to observation log only; do not notify Discord |
| `notify` | Send Discord embed; no remediation button |
| `escalate` | Send Discord embed with approve/reject remediation buttons |
| `never-touch` | Human-protected resource; classify but always escalate |

**System prompt design principles (from prior session synthesis):**

1. **Context priming:** Open with a description of the terminus homelab topology and current service map — LLM needs to know that Kaylee, FourDogs Central, and Watchdog itself are co-tenants and that ArgoCD manages all of them from the terminus-infra repo
2. **Known patterns table:** Include a short Markdown table of currently hardcoded detector pattern IDs and their canonical descriptions so the LLM can identify if an alert matches a known detector
3. **Output schema declaration:** Specify the JSON schema inline in the system prompt with field-level descriptions. Instruct the model to always emit all fields; use `"needs-investigation"` with `confidence: 0.0` for truly ambiguous cases rather than hallucinating a classification
4. **Chain-of-thought via evidence array:** The `evidence` array serves as the chain-of-thought trace — the LLM is instructed to populate it with 2-4 specific observations from the alert text that support the classification before naming the classification
5. **Failure mode instruction:** If alert text is empty or malformed, emit `classification: "no-action"`, `confidence: 1.0`, `hypothesis: "malformed alert"`, `action_tier: "silent"`

**Prompt length budget:** System prompt will be approximately 800-1200 tokens at steady state. User message (alert text + Loki context snippets) will be capped at 2000 tokens. Total input budget: ~3200 tokens per triage call. At homelab alert volume (estimated 5-30 triage calls/day), this is negligible for local Qwen and affordable for cloud fallback.

**Temperature:** `0.1` — classification should be deterministic. Lower temperature reduces hallucination risk on enum fields.

### Structured Outputs vs JSON Mode Decision

OpenAI Structured Outputs (`strict: true` with JSON Schema) guarantees schema adherence but requires `gpt-4o-mini-2024-07-18` or later. Since the gateway normalizes responses across providers and local Qwen does not support strict structured outputs, watchdog uses `json_object` mode with validation at the application layer:

```python
from pydantic import BaseModel, field_validator
from typing import Literal

class TriageResponse(BaseModel):
    classification: Literal[
        "transient-glitch", "known-degraded-dependency", "config-drift",
        "resource-pressure", "needs-investigation", "no-action"
    ]
    confidence: float
    hypothesis: str
    recommended_action: str
    action_tier: Literal["silent", "notify", "escalate", "never-touch"]
    evidence: list[str]

    @field_validator("confidence")
    def confidence_in_range(cls, v):
        if not 0.0 <= v <= 1.0:
            raise ValueError("confidence must be 0.0-1.0")
        return v
```

If Pydantic validation fails (LLM emitted invalid schema), watchdog falls back to `classification: "needs-investigation"`, `action_tier: "escalate"`, and logs a parse error. This ensures the watchdog never silently drops an alert due to LLM output format failure.

---

## Research Question 3: Observation log schema for E2 pattern learning

### Finding

**Persistence approach:** SQLite via aiosqlite (async-compatible; pure Python; no new services). File path: `/data/watchdog_observations.db` on a PVC (same PVC as any future persistent state). File-based SQLite is appropriate at homelab scale (<100 observations/day). SQLite allows structured queries for pattern graduation analysis (Q5).

Alternative considered: append-only JSON Lines file. Rejected because JOIN queries for graduation threshold analysis are complex in JSONL; SQLite provides equivalent simplicity with query capability.

**Schema:**

```sql
CREATE TABLE observations (
    id TEXT PRIMARY KEY,                     -- UUID v4
    ts REAL NOT NULL,                        -- Unix timestamp (float)
    source TEXT NOT NULL,                    -- 'argocd' | 'temporal' | 'k8s' | 'loki'
    pattern_id TEXT,                         -- hardcoded detector pattern_id if triggered by detector
    alert_text TEXT NOT NULL,                -- raw alert embed body presented to LLM
    llm_classification TEXT NOT NULL,        -- TriageResponse.classification
    llm_confidence REAL NOT NULL,            -- TriageResponse.confidence (0.0-1.0)
    llm_hypothesis TEXT NOT NULL,            -- TriageResponse.hypothesis
    action_taken TEXT NOT NULL,              -- 'silent' | 'notify' | 'escalate' | 'no-action'
    model_used TEXT NOT NULL,                -- model/profile name sent in request
    provider_used TEXT,                      -- provider that actually served (from gateway degraded signal)
    latency_ms INTEGER,                      -- gateway round-trip ms
    input_tokens INTEGER,                    -- from gateway response usage field
    human_verdict TEXT,                      -- 'correct' | 'incorrect' | NULL (pending)
    pattern_graduated INTEGER DEFAULT 0,     -- bool: 1 if graduated to hardcoded detector
    graduated_at REAL,                       -- Unix timestamp of graduation
    created_at REAL NOT NULL DEFAULT (unixepoch('now', 'subsec'))
);

CREATE INDEX idx_observations_ts ON observations(ts);
CREATE INDEX idx_observations_classification ON observations(llm_classification);
CREATE INDEX idx_observations_pattern_id ON observations(pattern_id);
CREATE INDEX idx_observations_human_verdict ON observations(human_verdict);
```

**Human verdict wiring:** The Discord approve/reject interaction handler (on_interaction) updates `human_verdict` for the corresponding observation row. This is the feedback loop from human-in-the-loop (E4) into the pattern learning store (E2).

**Pattern graduation query (E2):**

```sql
-- Candidates for pattern graduation
SELECT
    llm_classification,
    COUNT(*) as occurrence_count,
    AVG(llm_confidence) as avg_confidence,
    SUM(CASE WHEN human_verdict = 'correct' THEN 1 ELSE 0 END) as confirmed_correct,
    SUM(CASE WHEN human_verdict = 'incorrect' THEN 1 ELSE 0 END) as confirmed_incorrect
FROM observations
WHERE ts > unixepoch('now') - (30 * 86400)   -- last 30 days
  AND pattern_graduated = 0
GROUP BY llm_classification
HAVING
    occurrence_count >= 5
    AND avg_confidence >= 0.90
    AND confirmed_incorrect = 0
    AND confirmed_correct = occurrence_count;  -- 100% human-confirmed
```

---

## Research Question 4: Discord reply wired to Temporal signal

### Finding

This is a pure wiring story using existing MVP1 infrastructure. No new dependencies required.

**How MVP1 Discord Gateway mode handles interactions:**

The MVP1 architecture uses discord.py 2.4+ in Gateway websocket mode (Decision H2: no public ingress). Button presses and message component interactions arrive as `InteractionType.component` events on the same Gateway websocket, handled by `on_interaction()` in the bot event loop.

**Wiring pattern:**

The watchdog Discord message includes action buttons. Each button's `custom_id` encodes the Temporal workflow ID and the action type:

```python
# When emitting the Discord alert embed:
view = discord.ui.View(timeout=None)
view.add_item(discord.ui.Button(
    label="Approve Sync",
    style=discord.ButtonStyle.green,
    custom_id=f"approve:{workflow_run_id}:{correlation_id}"
))
view.add_item(discord.ui.Button(
    label="Reject",
    style=discord.ButtonStyle.red,
    custom_id=f"reject:{workflow_run_id}:{correlation_id}"
))

# In on_interaction() handler:
async def on_interaction(self, interaction: discord.Interaction):
    if interaction.type != discord.InteractionType.component:
        return
    custom_id = interaction.data.get("custom_id", "")
    parts = custom_id.split(":")
    if len(parts) != 3:
        return
    action, workflow_run_id, correlation_id = parts
    
    handle = self.temporal_client.get_workflow_handle(workflow_run_id)
    await handle.signal("human_response", {"action": action, "correlation_id": correlation_id})
    await interaction.response.send_message(
        f"Signal `{action}` sent to workflow `{workflow_run_id}`", ephemeral=True
    )
```

**Temporal signal definition (in workflow):**

```python
@workflow.defn
class TriageWorkflow:
    def __init__(self):
        self._human_response: dict | None = None

    @workflow.signal
    async def human_response(self, payload: dict) -> None:
        self._human_response = payload

    @workflow.run
    async def run(self, alert: Alert) -> TriageResult:
        # ... classification activities ...
        if triage_result.action_tier == "escalate":
            await workflow.wait_condition(
                lambda: self._human_response is not None,
                timeout=timedelta(hours=24)
            )
        # ... handle response ...
```

**Key constraints:**
- The `workflow_run_id` must be stored in the Discord message (embed footer or button `custom_id`) so the bot can resolve the handle on button press — even after a watchdog pod restart
- Discord component interactions time out after 15 minutes unless the view is persistent (`timeout=None`). For 24-hour human approval windows, use persistent views registered at bot startup
- The existing Temporal client in the watchdog (`temporalio` gRPC SDK) is already initialized at startup — no new client creation needed in `on_interaction()`

---

## Research Question 5: Pattern graduation thresholds

### Finding

Graduation thresholds are governance criteria for promoting an LLM-classified observation pattern into a hardcoded `BaseDetector`. The goal is to remove LLM inference cost for patterns that are well-understood and reliably detectable with deterministic logic.

**Proposed graduation criteria (all must be met):**

| Criterion | Threshold | Rationale |
|---|---|---|
| Minimum occurrences | ≥ 5 within 30 days | Enough samples for statistical confidence without waiting months |
| Average LLM confidence | ≥ 0.90 | LLM is consistently certain; classification is not ambiguous |
| Human verdict correctness | 100% confirmed correct | Zero false positives required before automating |
| False positive count | 0 | Any incorrect human verdict resets the candidate |
| Pattern stability | Same `alert_text` structure across all occurrences | Avoid graduating patterns tied to a one-time infra change |

**Graduation process:**

1. Periodic (weekly) observation log query run by watchdog ops or automated cron in watchdog
2. Graduation candidates surfaced as a Discord report (not auto-graduated)
3. Human reviews: confirms the pattern is structurally repeatable, not a temporal coincidence
4. Engineer writes a `BaseDetector` subclass with `pattern_id` matching the LLM classification value
5. Unit tests added per MVP1 testing conventions (one test per alert path, suppression path, error path)
6. PR merged; watchdog chart upgraded with new image
7. Observation log rows with matching `llm_classification` marked `pattern_graduated = 1, graduated_at = now`
8. LLM triage path skips known-graduated patterns to avoid duplicate alerting

**Anti-graduation criteria (reset candidate):**

- Any `human_verdict = 'incorrect'` for a given `llm_classification` resets the occurrence counter
- Pattern occurs in a single burst (all 5 within 24 hours) — indicates an incident, not a recurring operational pattern

---

## Research Question 6: ArgoCD webhook vs polling for E3

### Finding

**Current MVP1 pattern:** ArgoCD poller (`argocd_poller.py`) calls the ArgoCD REST API on a configurable poll interval (default: 30s; configurable via `ARGOCD_POLL_INTERVAL_SECONDS`).

**ArgoCD Notifications webhook approach:**

ArgoCD provides an outbound webhook notification service (`argocd-notifications`). It sends HTTP POST requests from ArgoCD to an external endpoint when a trigger fires (e.g., `on-sync-failed`, `on-degraded`). Configuration requires:

1. `argocd-notifications` operator deployed to the cluster (separate Helm chart)
2. `argocd-notifications-cm` ConfigMap: defines the webhook URL, headers, and trigger templates
3. Subscriptions on each ArgoCD `Application` object via annotation
4. Watchdog exposes an HTTP endpoint to receive the webhook POST

**Trade-off analysis at homelab scale:**

| Factor | Polling | ArgoCD Notifications Webhook |
|---|---|---|
| Detection latency | 30s worst case (configurable to 5s) | Near-instant (ArgoCD fires on state change) |
| Infrastructure complexity | None (already working) | Requires ArgoCD Notifications operator + configmap + Application annotations |
| Watchdog changes required | None (E1 reuses existing poller) | Add HTTP receiver endpoint (aiohttp; port 9091 or reuse port 9090) |
| Ingress required? | No | No (webhook is from ArgoCD → watchdog, both in cluster; ClusterIP is sufficient) |
| Failure mode | Poller catches state on next poll cycle | Webhook delivery failure is silent unless ArgoCD Notifications retries are configured |
| Event completeness | Poll sees current state; may miss short-lived transients | Webhook fires on every state transition; captures transient states polling may miss |
| Homelab event volume | ~5-20 ArgoCD sync events/day | Same volume; webhook is not needed for throughput |

**Decision for MVP:**

Polling is the correct choice for E1 and E2. The existing poller already works; at homelab scale (< 50 apps, < 20 sync events/day), 30s detection latency is operationally acceptable. The LLM triage value is in classification quality, not event ingestion speed.

E3 (ArgoCD Notifications webhook integration) remains a valid post-MVP enhancement for teams that require faster detection latency (< 5s) and are willing to operate the ArgoCD Notifications operator. The E3 story should be scoped as an optional enhancement, not a core requirement, and deferred until E1 and E2 are validated.

**Note on webhook receiver implementation if E3 is pursued:**

Since watchdog already runs an `aiohttp` server on port 9090 (for `/healthz` and `/metrics`), adding a `/webhook/argocd` route to that server is a one-story addition. No new port or Ingress is required — the webhook call originates from within the cluster (argocd-notifications controller → watchdog ClusterIP). The receiver would parse the ArgoCD event body, construct an internal `Alert` object, and inject it into the `detection_loop()` state machine bypassing the poller interval.

---

## Summary of Findings

| Research Question | Finding | Implementation Complexity |
|---|---|---|
| Gateway consumption | Call `/v1/chat/completions` via existing `httpx`; profile-based routing handles failover; new ESO secret required | Low — wiring story only |
| LLM prompt and schema | `json_object` mode + Pydantic validation; 6-value classification enum; 4-value action tier; evidence array as chain-of-thought | Medium — prompt engineering + validation layer |
| Observation log | SQLite via `aiosqlite`; 17-field schema; PVC-backed at `/data/watchdog_observations.db`; JSON serialized alert blob | Low-Medium — new persistence layer |
| Discord → Temporal signal | `custom_id` encodes workflow run ID; `on_interaction()` calls `get_workflow_handle().signal()`; persistent views for 24h windows | Low — wiring story using existing clients |
| Pattern graduation | 5 occurrences + ≥0.90 confidence + 100% human confirmed + 0 false positives in 30-day window | Low — SQL query + ops process |
| ArgoCD webhook vs polling | Polling is correct for MVP; webhook is optional E3 post-MVP enhancement | N/A for MVP |

---

## Open Items / Risks

1. **Qwen JSON mode reliability:** Local Qwen 3.27B (running on `worker04`) must reliably produce valid JSON in `json_object` mode. Risk: smaller models occasionally emit text outside the JSON block. Mitigation: post-processing to extract the first valid JSON object from the response, followed by Pydantic validation, followed by fallback to `needs-investigation`.

2. **Gateway routing profile ownership:** The `triage` routing profile in `ROUTING_CONFIG_PATH` is owned by terminus-infra, not terminus-watchdog. A dependency story is needed to ensure the profile exists before E5 can be tested. The watchdog business plan should flag this as a cross-repo dependency.

3. **Temporal SDK version compatibility:** The `temporalio` Python SDK version in watchdog MVP1 must support `get_workflow_handle().signal()` with a payload dict. Confirmed as supported in `temporalio >= 1.3.0`. Implementation story should verify the pinned version.

4. **LLM triage cost at cloud fallback rates:** If Qwen is unavailable and all triage calls route to OpenAI (GPT-4o-mini), estimated cost is ~$0.002 per triage call at 3200 input tokens. At 30 calls/day worst case: ~$0.06/day. Acceptable but worth monitoring via observation log `input_tokens` field.

5. **Persistent view re-registration on pod restart:** Discord persistent views (`timeout=None`) must be re-registered at bot startup to handle button presses for messages sent before a pod restart. The E4 story must include a startup task that queries the observation log for open (non-resolved) escalations and re-registers their views.
