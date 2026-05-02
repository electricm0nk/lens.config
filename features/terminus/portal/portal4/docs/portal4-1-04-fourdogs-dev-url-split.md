# Story portal4-1-04: Add Fourdogs dev URL and split Fourdogs card metadata

**Epic:** 1 — Visual Fixes and Service Config
**Status:** backlog
**Initiative:** portal4
**Date:** 2026-05-02
**Estimate:** M

---

## User Story

As an **operator working with the Fourdogs stack**,
I want separate dev and prod links on the Fourdogs service card,
So that I can navigate to `central-dev.fourdogspetsupplies.com` during development without editing config manually.

---

## Context

**Repo:** `terminus-portal` (React/Vite SPA)
**Branch:** `portal4`

The current Fourdogs service entry in `services.js` likely has a single URL pointing to the prod instance. Portal4 adds a dev URL and may split the Fourdogs section into multiple cards (Central API, Central UI, dev vs. prod) or add a dev link field to the existing card.

**URLs:**
- Prod: `https://central.fourdogspetsupplies.com` (or `https://4dogs.trantor.internal`)
- Dev: `https://central-dev.fourdogspetsupplies.com/login`

The implementation approach (split cards vs. dev link field) should follow whatever pattern exists in the current portal for multi-environment services. If no such pattern exists, add a `devUrl` field to the service entry and render it as a secondary link.

---

## Acceptance Criteria

**Given** the portal dashboard loads
**When** the Fourdogs service card(s) render
**Then** both the prod URL and dev URL (`https://central-dev.fourdogspetsupplies.com/login`) are accessible as links

**Given** the dev URL is a separate link (not replacing prod)
**When** the operator clicks the dev link
**Then** the browser navigates to the dev login page (opens in new tab, consistent with other portal links)

**Given** the fix is applied
**When** `npm test` and `npm run build` are run
**Then** all pass with no regressions, including any tests that assert on the Fourdogs service config

---

## Dev Notes

- Check how other multi-environment services are handled in `services.js`; follow existing patterns
- If no multi-environment pattern exists: add a `devUrl` field to the Fourdogs entry and conditionally render it in `ServiceCard.jsx` as a secondary "Dev" button/link
- The dev URL should open in a new tab (`target="_blank"`, `rel="noopener noreferrer"`)
- Card metadata split (if needed): Fourdogs Central UI and Fourdogs Central API may warrant separate entries — use judgment based on how other compound services are handled

---

## Definition of Done

- [ ] Code merged to feature branch (`portal4`)
- [ ] No TypeScript/ESLint errors (`npm test` and `npm run build` passing)
- [ ] Manual smoke test: Fourdogs card shows both prod and dev links; dev link opens `central-dev.fourdogspetsupplies.com/login`
- [ ] No secrets committed to source code
