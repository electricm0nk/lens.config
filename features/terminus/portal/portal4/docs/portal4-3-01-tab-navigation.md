# Story portal4-3-01: Implement tab navigation bar in App.jsx (useState)

**Epic:** 3 — Tab Navigation and Release Pipeline Tab
**Status:** backlog
**Initiative:** portal4
**Date:** 2026-05-02
**Estimate:** M

---

## User Story

As an **operator using the portal**,
I want a tab navigation bar at the top of the portal with tabs for Overview, Release Pipeline, and Pods,
So that I can switch between the operational views without a page reload.

---

## Context

**Repo:** `terminus-portal` (React/Vite SPA)
**Branch:** `portal4`
**Depends on:** Sprint 1 complete (no hard dependency, but Sprint 1 fixes the base layout this tab nav sits on top of)

**Architecture (ADR-6 in tech-plan):**
- `useState` in `App.jsx` to track `activeTab` — values: `"overview"`, `"release-pipeline"`, `"pods"`
- Default tab: `"overview"` (preserves current dashboard on load)
- Tab bar renders above the main content area
- Each tab renders its component on selection; other tabs are unmounted (or hidden with CSS — follow existing portal pattern for any already-mounted components like MetricsPanel)

---

## Acceptance Criteria

**Given** the portal loads
**When** the page renders
**Then** a tab bar is visible with three tabs: "Overview", "Release Pipeline", "Pods"

**Given** the "Overview" tab is active
**When** the page renders
**Then** the existing dashboard content (ServiceGrid, MetricsPanel, etc.) is displayed unchanged

**Given** the "Release Pipeline" tab is clicked
**When** the tab activates
**Then** a placeholder "Release Pipeline — coming soon" is shown (full implementation in story 3-03)

**Given** the "Pods" tab is clicked
**When** the tab activates
**Then** a placeholder "Pods — coming soon" is shown (full implementation in story 4-02)

**Given** `npm test` is run
**Then** all existing tests pass; new tab navigation smoke test added

**Given** `npm run build` is run
**Then** no TypeScript/ESLint errors

---

## Dev Notes

- Modify `src/App.jsx`: add `const [activeTab, setActiveTab] = useState("overview")`
- Add a `<TabBar>` component (can be inline in App.jsx or a separate `src/components/TabBar.jsx`)
- Tab bar styling: follow the existing portal theme (dark background, consistent with service cards); keep it minimal — no external UI library required
- The tab bar should be sticky/fixed at the top or below the portal header (match existing header treatment)
- Version watermark: add `VITE_APP_VERSION` env var display in the portal footer or header — this is also part of this story (referenced in Sprint 3 AC). Read it as `import.meta.env.VITE_APP_VERSION` and display it in a small footer text element.

---

## Definition of Done

- [ ] Code merged to feature branch (`portal4`)
- [ ] Three tabs render: Overview, Release Pipeline, Pods
- [ ] Overview tab shows existing dashboard content unchanged
- [ ] Release Pipeline and Pods tabs show placeholder content
- [ ] Version watermark displayed in portal header or footer
- [ ] `npm test` passes (including tab navigation smoke test)
- [ ] `npm run build` passes with no TypeScript/ESLint errors
- [ ] No secrets committed to source code
