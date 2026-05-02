# Story portal4-3-03: Build Release Pipeline tab component (ReleasePipelineTab.jsx)

**Epic:** 3 — Tab Navigation and Release Pipeline Tab
**Status:** backlog
**Initiative:** portal4
**Date:** 2026-05-02
**Estimate:** M

---

## User Story

As an **operator**,
I want a Release Pipeline tab that shows per-repo, per-branch release status for all tracked repos,
So that I can see at a glance which repos are on what version in dev vs. prod without visiting each GitHub repo individually.

---

## Context

**Repo:** `terminus-portal` (React/Vite SPA)
**Branch:** `portal4`
**Depends on:**
- portal4-3-01 merged (tab navigation exists; Release Pipeline tab renders in it)
- portal4-3-02 merged (sidecar GitHub proxy route must be available)
- portal4-3-04 merged (or done in same PR — `releasePipeline.js` config must exist for repo list)

**Architecture (ADR-6 in tech-plan):** React component using `useEffect` + `fetch` to call `/api/github/*` sidecar routes. Each repo row shows dev and main branch data.

---

## Acceptance Criteria

**Given** the Release Pipeline tab is selected
**When** the component mounts
**Then** it begins fetching data for all repos defined in `releasePipeline.js`

**Given** data is loading
**When** the fetch is in progress
**Then** a loading indicator (spinner or skeleton) is shown for each repo row

**Given** data has loaded for a repo
**When** the row renders
**Then** the dev branch (if configured) shows: tag, SHA (first 7 chars), commit message (truncated), age, CI status indicator
**Then** the main/prod branch shows the same fields

**Given** a repo is configured with only a prod branch (no `develop`)
**When** the row renders
**Then** only the prod branch data is shown; the dev column is empty/omitted

**Given** a repo returns `"tag": "no tag"`
**When** the row renders
**Then** the tag cell shows "no tag" (not blank, not an error)

**Given** a fetch returns a rate limit error
**When** the row renders
**Then** the row shows "rate limited — try again" state (not a crash or blank)

**Given** `npm test` and `npm run build` are run
**Then** both pass with no errors

---

## Dev Notes

- Create `src/components/ReleasePipelineTab.jsx`
- Import repo list from `releasePipeline.js` (story 3-04)
- Use `useEffect` with `Promise.allSettled` to fetch all repos in parallel — don't block one repo failure from showing others
- CI status: map `"success"` → green checkmark, `"failure"` → red X, `"pending"` → yellow spinner, `"unknown"` → grey dash
- Age: use the `age` field from the sidecar response directly (already human-readable)
- SHA: display first 7 characters, link to `https://github.com/:owner/:repo/commit/:sha`
- Table layout: one row per repo, columns: Repo, Dev Tag, Dev SHA, Dev CI, Prod Tag, Prod SHA, Prod CI
- Mobile viewport is not a design goal — table layout is fine for operator tooling

---

## Definition of Done

- [ ] Code merged to feature branch (`portal4`)
- [ ] All tracked repos show tag, SHA, CI status for configured branches
- [ ] Rate limit state handled gracefully (no crash)
- [ ] No-tag repos show "no tag" (not blank)
- [ ] Loading state visible during data fetch
- [ ] `npm test` passes; `npm run build` passes with no TypeScript/ESLint errors
- [ ] No secrets committed to source code
