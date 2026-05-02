# Story portal4-1-02: Fix ArgoCD and Semaphore icon slugs

**Epic:** 1 — Visual Fixes and Service Config
**Status:** backlog
**Initiative:** portal4
**Date:** 2026-05-02
**Estimate:** S

---

## User Story

As an **operator viewing the portal dashboard**,
I want the ArgoCD and Semaphore service cards to display their correct icons,
So that services are visually identifiable at a glance without broken image placeholders.

---

## Context

**Repo:** `terminus-portal` (React/Vite SPA)
**Branch:** `portal4`
**Depends on:** portal4-1-01 merged (or can be done in parallel on same branch)

ArgoCD and Semaphore service cards are currently showing broken images or a default placeholder icon instead of the correct branded icon. The icons are referenced by a slug or asset path in `src/services.js` (or equivalent config). The issue is likely that the icon slug doesn't match the available icon assets, or the asset hasn't been added to the project.

---

## Acceptance Criteria

**Given** the portal dashboard loads
**When** the ServiceGrid renders the ArgoCD card
**Then** the ArgoCD icon is displayed correctly (not a broken image or generic placeholder)

**Given** the portal dashboard loads
**When** the ServiceGrid renders the Semaphore CI card
**Then** the Semaphore icon is displayed correctly (not a broken image or generic placeholder)

**Given** the fix is applied
**When** `npm test` and `npm run build` are run
**Then** all pass with no regressions

---

## Dev Notes

- Check `src/services.js` for the `icon` field on the ArgoCD and Semaphore service entries
- Check `src/components/ServiceCard.jsx` (or equivalent) for how the icon field is rendered — is it an img src, an SVG import, or a CSS class slug?
- If icon assets are missing: SVGs for ArgoCD and Semaphore CI are available in their respective project repos or CDNs. Add the SVG to `src/assets/icons/` and reference it. If asset sourcing takes too long, a text-only fallback card is acceptable for this sprint.
- Icon slugs should follow whatever convention is already used by working cards (e.g. Grafana, Temporal)

---

## Definition of Done

- [ ] Code merged to feature branch (`portal4`)
- [ ] No TypeScript/ESLint errors (`npm test` and `npm run build` passing)
- [ ] Manual smoke test: ArgoCD and Semaphore cards show correct icons in dev
- [ ] No secrets committed to source code
