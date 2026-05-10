# Story 6.4 — System Prompt Update + Helm Configmap Patch

**Epic:** 6 — Order management tools  
**Track:** hotfix-express  
**Feature:** fourdogs-kaylee-order-update-bug  
**Depends on:** Story 6.3

---

## Context

Even with working tools, the system prompt still tells the LLM to "do it and confirm briefly" as if it has autonomous write access. The prompt needs tightening so the LLM:
1. Reports what the tool actually returned (not what it intended)
2. Understands it cannot modify submitted orders
3. Asks for clarification on ambiguous item matches rather than guessing

This is a configmap change — no Python change, no DB migration, no new release required beyond a helm/ArgoCD sync.

## Acceptance Criteria

**Given** an updated system prompt in `kaylee-agent-config` (both prod and dev namespaces)  
**When** the LLM receives an ambiguous item match from `order_update`  
**Then** it relays the ambiguity message rather than fabricating a choice

**Given** the system prompt update  
**When** the tool context says "Order is already submitted"  
**Then** the LLM does not try to "confirm" a write that didn't happen

**Given** a successful `order_update` tool result ("Updated: Blue Buffalo ... qty set to 5")  
**When** the LLM responds  
**Then** its response matches the tool result, not a hallucinated value

---

## TDD Plan (Article 7)

This story has no Python implementation to test. Testing is behavioural — validate via the dev environment after deploy.

**Pre-deploy checklist (replaces red-green for config-only changes):**
- [ ] Diff the proposed prompt change against current — confirm no ORDERING MODE capability removal (only constraint language added)
- [ ] Send a test message to dev Kaylee with a known ambiguous item: confirm LLM relays ambiguity
- [ ] Send a test with a submitted order (if one exists): confirm LLM reports it cannot modify
- [ ] Send a valid single-item update: confirm LLM response matches tool output exactly

---

## System Prompt Change

In the `ORDERING MODE` section, replace:

```
When Betsy tells you to adjust the order worksheet — add cases, remove items,
change quantities — do it and confirm briefly. If something seems off (adding a
lot of a slow-moving item, or the order already has plenty), mention it once and
then do what she asked unless she says otherwise. Don't second-guess her twice.
```

With:

```
When Betsy tells you to adjust the order worksheet — add cases, remove items,
change quantities — the adjustment runs in the background and you'll get back
the actual result. Confirm what the system actually saved, not what you intended
to save. If the result says the order is already submitted, tell her plainly and
don't pretend to make a change. If the result shows multiple possible items,
ask which one she means before touching anything. Don't second-guess her twice
once she's clarified.
```

---

## Helm / Configmap Patch

Target files in the `fourdogs-kaylee-agent` Helm chart (both environments):

```
charts/fourdogs-kaylee-agent/templates/configmap.yaml
```

or the environment-specific `values-prod.yaml` / `values-dev.yaml` if the prompt is stored there.

Procedure:
1. Locate where `system-prompt.txt` is rendered in the chart
2. Update the prompt text
3. Commit to `develop` branch of `fourdogs-kaylee-agent` repo
4. ArgoCD auto-syncs both namespaces (no pod restart needed — configmap reload triggers on next request via mounted file)

**Note:** The pod mounts the configmap at `/etc/kaylee/system-prompt.txt`. Changes to a mounted configmap are reflected within ~60s without a pod restart (k8s default configmap sync interval).

---

## Tasks

- [ ] Locate system-prompt configmap in Helm chart
- [ ] Apply prompt text change (exact replacement above)
- [ ] Commit to `develop` branch
- [ ] Push — ArgoCD syncs dev namespace
- [ ] Run behavioural validation against dev
- [ ] ArgoCD sync prod namespace
- [ ] Confirm prod Kaylee relays tool results faithfully on a test message
