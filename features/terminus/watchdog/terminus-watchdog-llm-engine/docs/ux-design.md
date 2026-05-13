---
feature: terminus-watchdog-llm-engine
doc_type: ux-design
status: draft
goal: "Define the Discord operator interface for LLM-enriched triage alerts: embed formats, interaction patterns, information hierarchy, and state transitions."
key_decisions:
  - Discord is the sole operator UI surface — no web dashboard, no separate notification channel
  - Embed color coding by action tier (gray/blue/orange/red) for at-a-glance triage
  - Approval buttons shown only for escalate tier; never-touch shows no approve option
  - Confidence displayed as percentage (e.g., 87%) not raw decimal (0.87)
  - Evidence array displayed as inline bullet list, max 4 items
  - 24-hour approval window with message edit on expiry
  - Silent tier produces no Discord output — observation is Temporal-only
open_questions: []
depends_on:
  - prd.md (E1, E4, E2)
blocks: []
updated_at: 2026-05-13
---

# UX Design Specification — Watchdog LLM Engine

**Feature:** `terminus-watchdog-llm-engine`  
**UX Surface:** Discord (sole operator interface)  
**Owner:** Todd Hintzmann  
**Date:** 2026-05-13

---

## 1. Design Context

### Operator Profile

Single operator (Todd) monitoring a k3s homelab. The Discord server is the existing, always-open operational dashboard. Alerts from MVP1 already arrive here. This feature adds LLM-enriched message formats and interactive approval buttons to the same channel.

**Operator context when receiving an alert:**

- Working at a computer with Discord visible
- May be mid-task, not focused on infra
- Needs: enough context to make an approve/reject decision without switching to a terminal
- Wants: brevity over completeness; one hypothesis, not a dissertation

### Design Principles

1. **At-a-glance scanability** — the action tier and classification must be readable in under 2 seconds without reading body text
2. **Sufficient context, not complete context** — hypothesis + 2-4 evidence items; not full log dumps
3. **One decision per message** — approve or reject; no multi-step flows in Discord
4. **Honest confidence** — display confidence as percentage; add a visual indicator for low-confidence results (< 70%)
5. **Recoverable from silence** — if the operator ignores a message and it expires, a clear update indicates what happened and the status

---

## 2. Alert Message Templates

### Template 1 — Notify Tier (no interaction required)

Used when `action_tier = notify`. LLM has a hypothesis but no safe automated action is available. Operator is informed; no decision required.

```
┌────────────────────────────────────────────────────────────────────┐
│ 🔵  WATCHDOG  ·  config-drift                       confidence: 87% │
├────────────────────────────────────────────────────────────────────┤
│ app/fourdogs-central · ArgoCD                                        │
│                                                                      │
│ Hypothesis                                                           │
│ fourdogs-central is OutOfSync due to an image tag change in the     │
│ values file that has not been approved for sync in ArgoCD.          │
│                                                                      │
│ Evidence                                                             │
│ • argocd: fourdogs-central Health=Healthy Status=OutOfSync          │
│ • loki: "image tag differs from deployed" (2 occurrences, last 5m)  │
│ • no recent ArgoCD sync operation in last 6h                        │
│                                                                      │
│ Recommended Action                                                   │
│ Review and approve the pending sync in ArgoCD                       │
│                                                                      │
│ watchdog-triage · 2026-05-13 14:32:01 UTC                           │
└────────────────────────────────────────────────────────────────────┘
```

**Embed fields:**
- **Color:** Blue (`#2196F3`) — informational, no action required
- **Title:** `🔵  WATCHDOG  ·  {classification}` + right-aligned `confidence: {N}%`
- **Author line:** `{pattern_id} · {alert_source}` (e.g., `app/fourdogs-central · ArgoCD`)
- **Hypothesis:** One sentence, imperative voice, plain English
- **Evidence:** 2–4 bullet items from LLM evidence array; truncate to 80 chars each
- **Recommended Action:** One imperative sentence
- **Footer:** `watchdog-triage · {timestamp UTC}`

---

### Template 2 — Escalate Tier (approval required)

Used when `action_tier = escalate`. Watchdog can execute a remediation action but requires human confirmation.

```
┌────────────────────────────────────────────────────────────────────┐
│ 🟠  WATCHDOG  ·  config-drift                       confidence: 92% │
├────────────────────────────────────────────────────────────────────┤
│ app/fourdogs-central · ArgoCD                                        │
│                                                                      │
│ Hypothesis                                                           │
│ fourdogs-central is OutOfSync. The drift appears to be a known-safe │
│ image tag bump. Approving sync is the recommended action.           │
│                                                                      │
│ Evidence                                                             │
│ • argocd: fourdogs-central Health=Healthy Status=OutOfSync          │
│ • loki: "image tag differs from deployed" (3 occurrences, last 10m) │
│ • last sync: 2026-05-13 08:14 UTC (6h ago, completed successfully)  │
│ • no resource deletion or namespace changes detected                │
│                                                                      │
│ Remediation                                                          │
│ Approve sync for fourdogs-central in ArgoCD                         │
│                                                                      │
│ ⚠️  Approval window: 24 hours                                        │
│ watchdog-triage · 2026-05-13 14:32:01 UTC                           │
│                                                                      │
│  [ ✅ Approve ]          [ ❌ Reject ]                               │
└────────────────────────────────────────────────────────────────────┘
```

**Embed fields:**
- **Color:** Orange (`#FF9800`) — action required
- **Title:** `🟠  WATCHDOG  ·  {classification}` + `confidence: {N}%`
- **Body:** same structure as Template 1
- **Remediation:** replaces "Recommended Action" label with "Remediation" to signal that the watchdog will execute on Approve
- **Approval window indicator:** `⚠️  Approval window: 24 hours`
- **Buttons:**
  - `✅ Approve` — style: Success (green); `custom_id: approve:{workflow_run_id}`
  - `❌ Reject` — style: Danger (red); `custom_id: reject:{workflow_run_id}`

**Low-confidence variant (confidence < 70%):**

Same template; confidence percentage shown in amber with warning prefix:
- Title becomes: `🟠  WATCHDOG  ·  needs-investigation` + `confidence: 58% ⚠️`
- Hypothesis prefixed with: *"Low confidence — LLM could not classify with certainty:"*

---

### Template 3 — Never-Touch Tier (escalate, no approval)

Used when `action_tier = never-touch`. Resource is human-protected; watchdog will not execute any remediation regardless of classification.

```
┌────────────────────────────────────────────────────────────────────┐
│ 🔴  WATCHDOG  ·  needs-investigation                confidence: 74% │
├────────────────────────────────────────────────────────────────────┤
│ pvc/temporal-postgres-data · k8s                                     │
│                                                                      │
│ Hypothesis                                                           │
│ The Temporal Postgres PVC is nearing capacity. Disk pressure may    │
│ cause database writes to fail within 4–8 hours.                     │
│                                                                      │
│ Evidence                                                             │
│ • k8s: temporal-postgres pod CPU normal, filesystem usage 91%       │
│ • loki: "disk usage high" warning (1 occurrence, last 15m)          │
│ • no automated action available for PVC expansion                   │
│                                                                      │
│ ⛔  Human action required — this resource is never-touch             │
│ watchdog-triage · 2026-05-13 14:32:01 UTC                           │
└────────────────────────────────────────────────────────────────────┘
```

**Embed fields:**
- **Color:** Red (`#F44336`) — urgent, human-only
- **Title:** `🔴  WATCHDOG  ·  {classification}` + `confidence: {N}%`
- **Body:** same structure as Template 1 with Recommended Action replaced by:
  - `⛔  Human action required — this resource is never-touch`
- **No buttons** — never-touch tier has no approval path

---

### Template 4 — Approved State (post-interaction update)

The original escalate embed is **edited in-place** after the operator clicks Approve.

```
┌────────────────────────────────────────────────────────────────────┐
│ ✅  WATCHDOG  ·  config-drift  ·  APPROVED          confidence: 92% │
├────────────────────────────────────────────────────────────────────┤
│ app/fourdogs-central · ArgoCD                                        │
│                                                                      │
│ [original hypothesis and evidence — unchanged]                      │
│                                                                      │
│ ✅  Approved by operator at 2026-05-13 14:38:12 UTC                 │
│ Action: ArgoCD sync for fourdogs-central — executed                 │
└────────────────────────────────────────────────────────────────────┘
```

- **Color:** Green (`#4CAF50`)
- Buttons removed
- Footer updated with approval timestamp and action confirmation
- Original content preserved for audit

---

### Template 5 — Rejected State (post-interaction update)

```
┌────────────────────────────────────────────────────────────────────┐
│ ❌  WATCHDOG  ·  config-drift  ·  REJECTED          confidence: 92% │
├────────────────────────────────────────────────────────────────────┤
│ app/fourdogs-central · ArgoCD                                        │
│                                                                      │
│ [original hypothesis and evidence — unchanged]                      │
│                                                                      │
│ ❌  Rejected by operator at 2026-05-13 14:38:55 UTC                 │
│ No action taken.                                                     │
└────────────────────────────────────────────────────────────────────┘
```

- **Color:** Gray (`#9E9E9E`)
- Buttons removed

---

### Template 6 — Expired State (24-hour timeout edit)

```
┌────────────────────────────────────────────────────────────────────┐
│ ⏰  WATCHDOG  ·  config-drift  ·  EXPIRED           confidence: 92% │
├────────────────────────────────────────────────────────────────────┤
│ app/fourdogs-central · ArgoCD                                        │
│                                                                      │
│ [original hypothesis and evidence — unchanged]                      │
│                                                                      │
│ ⏰  Approval window expired at 2026-05-13 14:32:01 UTC              │
│ No action was taken. Review manually if the condition persists.     │
└────────────────────────────────────────────────────────────────────┘
```

- **Color:** Gray (`#9E9E9E`)
- Buttons removed

---

## 3. Interaction Flow

### Happy Path — Escalate → Approve

```
[Watchdog detects novel alert]
    │
    ▼
[LLM classifies: config-drift, 92%, escalate tier]
    │
    ▼
[Discord: Template 2 embed sent with Approve/Reject buttons]
    │
    ▼ (operator clicks Approve within 24h)
[Discord: on_interaction() fires → Temporal signal "human_response" → verdict: approve]
    │
    ▼
[Temporal workflow executes remediation action]
    │
    ▼
[Discord: Template 2 edited to Template 4 (Approved state)]
    │
    ▼
[Temporal workflow completes, observation search attributes updated: HumanVerdict=correct]
```

### Reject Path

```
[Operator clicks Reject]
    │
    ▼
[Discord: on_interaction() fires → Temporal signal "human_response" → verdict: reject]
    │
    ▼
[Temporal workflow closes without executing action]
    │
    ▼
[Discord: Template 2 edited to Template 5 (Rejected state)]
    │
    ▼
[Observation: HumanVerdict=incorrect]
```

### Expiry Path

```
[Temporal workflow approval-wait timer fires at 24h]
    │
    ▼
[Temporal workflow closes with status=expired]
    │
    ▼
[Discord: Template 2 edited to Template 6 (Expired state)]
    │
    ▼
[Observation: HumanVerdict=expired]
```

### Silent Path (no Discord output)

```
[Watchdog detects novel alert]
    │
    ▼
[LLM classifies: no-action OR transient-glitch with silent tier]
    │
    ▼
[No Discord message]
    │
    ▼
[Temporal TriageWorkflow started with observation data]
    │
    ▼ (7 days pass, no contradiction)
[Temporal workflow auto-confirms: HumanVerdict=auto-confirmed]
```

---

## 4. Information Hierarchy

### Priority Ranking in Embed

The operator's scanning pattern (top-to-bottom, 2-second glance):

| Rank | Information | Location | Rationale |
|------|-------------|----------|-----------|
| 1 | Action tier icon + classification | Title | Answers "do I need to act?" immediately |
| 2 | Confidence percentage | Title (right-aligned) | Calibrates trust without reading body |
| 3 | Alert source + resource | Author line | Answers "which service?" |
| 4 | Hypothesis (one sentence) | First body field | Answers "what's wrong?" |
| 5 | Evidence (2-4 bullets) | Second body field | Answers "why does the LLM think so?" |
| 6 | Remediation / Recommended Action | Third body field | Answers "what should I do?" |
| 7 | Buttons | Bottom | Answers "how do I respond?" |

### Field Constraints

| Field | Max Length | Notes |
|---|---|---|
| Title | 80 chars | truncate classification if needed |
| Author line | 60 chars | truncate pattern_id |
| Hypothesis | 200 chars | LLM must produce within this budget; prompt instruction |
| Evidence item | 80 chars each, max 4 items | truncate with `…` |
| Remediation | 150 chars | |

---

## 5. Discord Channel Strategy

**Recommended channel layout (additive to existing MVP1 setup):**

| Channel | Purpose |
|---|---|
| `#watchdog-alerts` (existing) | All MVP1 alerts + all LLM notify/escalate/never-touch embeds |
| No new channels required | Silent-tier observations are Temporal-only; no new channel needed |

**Why no separate triage channel:**  
At homelab scale (< 30 triage events/day), consolidating all alert types in one channel is less cognitively demanding than context-switching between channels. Color coding and tier icons provide sufficient visual differentiation.

---

## 6. Edge Cases and Error States

### Gateway Unreachable (MVP1 Fallback)

When the inference gateway is unreachable, the watchdog falls back to an MVP1-style alert (no LLM enrichment):

```
┌────────────────────────────────────────────────────────────────────┐
│ ⚪  WATCHDOG  ·  unclassified  [LLM unavailable]                    │
├────────────────────────────────────────────────────────────────────┤
│ fourdogs-central · ArgoCD                                            │
│                                                                      │
│ An alert was detected but LLM triage is currently unavailable.      │
│ Review manually.                                                     │
│                                                                      │
│ watchdog-triage · 2026-05-13 14:32:01 UTC                           │
└────────────────────────────────────────────────────────────────────┘
```

- **Color:** Gray (`#9E9E9E`)
- No buttons
- Distinguishable from escalate/never-touch by gray color and `[LLM unavailable]` suffix in title

### Pod Restart — View Re-registration

When the watchdog pod restarts, all open RUNNING TriageWorkflows have their Discord persistent views re-registered at startup. The operator sees no visible change in Discord — existing messages retain their buttons. No new messages are sent for re-registration.

### Duplicate Button Click

If a second user (future consideration) clicks Approve after the first approval has been processed, the Temporal workflow rejects the duplicate signal silently. The Discord message is already in the Approved state (Template 4) — the button is non-functional (disabled state applied on first approval).

---

## 7. Accessibility and Usability

- All status distinctions rely on **both color and icon** — not color alone (colorblind-safe)
- Confidence percentage uses numeric representation — not subjective labels like "high" or "medium"
- Evidence bullets use plain English, not internal log codes
- Hypothesis is written by the LLM in operator-facing language; the system prompt instructs the model to avoid technical jargon
- Timestamp always shown in UTC with full ISO format to avoid timezone ambiguity

---

## 8. Implementation Notes for TechPlan

- Discord embeds are built using `discord.Embed` with `discord.ui.View` for button rows
- `custom_id` format: `{action}:{workflow_run_id}` (e.g., `approve:terminus-triage-2026-05-13-fourdogs-central-abc123`)
- `custom_id` length limit: 100 chars — `workflow_run_id` must be kept short; use hash or truncated timestamp+pattern combination
- Embed edit on state transition: use `await message.edit(embed=updated_embed, view=None)` to clear buttons
- View persistence: `discord.ui.View(timeout=None)` registered via `bot.add_view()` at startup
- Color values should be defined as constants in a `watchdog.ui.colors` module for maintainability
- The `confidence: N%` display in the title uses Discord's inline title limitation — if Discord truncates, move confidence to a footer field
