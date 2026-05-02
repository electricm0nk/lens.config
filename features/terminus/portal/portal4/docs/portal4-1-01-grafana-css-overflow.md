# Story portal4-1-01: Fix Grafana card CSS overflow in ServiceGrid

**Epic:** 1 — Visual Fixes and Service Config
**Status:** backlog
**Initiative:** portal4
**Date:** 2026-05-02
**Estimate:** S

---

## User Story

As an **operator viewing the portal dashboard**,
I want the Grafana service card to display correctly without overflowing or truncating its content,
So that all service cards on the Overview tab are visually consistent and readable.

---

## Context

**Repo:** `terminus-portal` (React/Vite SPA)
**Branch:** `portal4`
**This is Story 1.1 — the first story of portal4.**

The ServiceGrid panel containing service cards has a CSS issue where the Grafana card overflows its container on standard viewport sizes (1280px+). The exact overflow vector (height, width, or content spillover) should be confirmed by running the portal locally before patching.

Source files likely involved:
- `src/components/ServiceGrid.jsx` or `src/components/ServiceGrid.css`
- Possibly `src/services.js` if the Grafana entry has an unusually long description or URL

---

## Acceptance Criteria

**Given** the portal dashboard is loaded in a standard browser viewport (≥1280px wide)
**When** the Overview tab renders the ServiceGrid
**Then** the Grafana card displays within its grid cell without any content spilling outside the card boundary

**Given** the fix is applied
**When** the developer runs `npm test`
**Then** all existing tests pass with no regressions

**Given** the fix is applied
**When** the developer runs `npm run build`
**Then** the build completes with no TypeScript/ESLint errors

---

## Dev Notes

- Check `src/components/ServiceGrid.jsx` and its CSS for overflow rules; likely a `max-height`, `overflow: hidden`, or grid row sizing issue
- Do not change card layout structure — keep the existing grid composition
- Minimal CSS change preferred: fix the specific overflow, not a grid refactor
- If the issue is in `services.js` (e.g. a very long description), truncate the description field instead
- Smoke test: load the portal dev server locally, verify Grafana card renders cleanly alongside neighboring cards

---

## Definition of Done

- [ ] Code merged to feature branch (`portal4`)
- [ ] No TypeScript/ESLint errors (`npm test` and `npm run build` passing)
- [ ] Manual smoke test: Grafana card displays correctly in dev at ≥1280px viewport
- [ ] No secrets committed to source code
