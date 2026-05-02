# Story portal4-1-03: Add Temporal service card to services.js

**Epic:** 1 — Visual Fixes and Service Config
**Status:** backlog
**Initiative:** portal4
**Date:** 2026-05-02
**Estimate:** S

---

## User Story

As an **operator viewing the portal dashboard**,
I want a Temporal service card in the Platform section,
So that I can navigate to the Temporal UI directly from the portal.

---

## Context

**Repo:** `terminus-portal` (React/Vite SPA)
**Branch:** `portal4`

Temporal is deployed in the `terminus` cluster but does not yet have a service card in the portal. It should appear in the Platform section alongside Grafana, Prometheus, etc.

The `temporal` feature in governance is at `preplan` phase — the Temporal UI may not be fully deployed yet. If Temporal is not reachable at the configured URL, the card should show a "UNREACHABLE" or similar status gracefully (same as other unreachable services). The card is additive only.

**Temporal UI URL (dev/prod):** To be confirmed by operator; use `https://temporal.trantor.internal` as the expected hostname pending confirmation.

---

## Acceptance Criteria

**Given** the portal dashboard loads
**When** the ServiceGrid renders the Platform section
**Then** a Temporal card appears with the correct service name, icon (or text fallback), and link URL

**Given** the Temporal UI is not yet reachable
**When** the Temporal card is rendered
**Then** the card displays gracefully (not a crash; a status indicator or greyed-out link is acceptable)

**Given** the fix is applied
**When** `npm test` and `npm run build` are run
**Then** all pass with no regressions

---

## Dev Notes

- Add a new entry in `src/services.js` (or equivalent) for Temporal:
  - `name`: "Temporal"
  - `section`: "Platform" (or whatever section Grafana is in)
  - `url`: `https://temporal.trantor.internal` (operator to confirm before S1 deploy)
  - `icon`: use the Temporal icon slug if available; text fallback acceptable
- Do not change the card layout — just add the config entry
- If service health checks run against the URL, the card will show UNREACHABLE if not deployed — that is acceptable and expected for now

---

## Definition of Done

- [ ] Code merged to feature branch (`portal4`)
- [ ] No TypeScript/ESLint errors (`npm test` and `npm run build` passing)
- [ ] Manual smoke test: Temporal card appears in the Platform section; if Temporal UI is not reachable, card shows graceful status
- [ ] No secrets committed to source code
