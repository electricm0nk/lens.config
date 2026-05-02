# Story portal4-3-04: Replace githubActions.js with releasePipeline.js config

**Epic:** 3 — Tab Navigation and Release Pipeline Tab
**Status:** backlog
**Initiative:** portal4
**Date:** 2026-05-02
**Estimate:** S

---

## User Story

As a **developer configuring the Release Pipeline tab**,
I want a `releasePipeline.js` config file that defines all tracked repos and their branch topology,
So that adding or removing a repo from the Release Pipeline view is a one-line config change.

---

## Context

**Repo:** `terminus-portal` (React/Vite SPA)
**Branch:** `portal4`

The portal currently has a `githubActions.js` (or similar) config file for GitHub Actions display from portal3. This story replaces it with `releasePipeline.js` that defines the full repo list for portal4's Release Pipeline tab.

**Repos to include (9 total):**

| Repo | Dev Branch | Prod Branch |
|------|-----------|------------|
| `electricm0nk/terminus-portal` | `develop` | `main` |
| `electricm0nk/terminus.infra` | — | `main` |
| `electricm0nk/terminus.platform` | — | `main` |
| `electricm0nk/lens.config` | — | `main` |
| `electricm0nk/fourdogs-central` | `develop` | `main` |
| `electricm0nk/fourdogs-central-ui` | `develop` | `main` |
| `electricm0nk/fourdogs-kaylee-agent` | `develop` | `main` |
| `electricm0nk/terminus-inference-gateway` | `develop` | `main` |
| `electricm0nk/terminus-inference-qwen-warmup` | — | `main` |

---

## Acceptance Criteria

**Given** `src/config/releasePipeline.js` (or `src/releasePipeline.js`) exists
**When** the file is imported
**Then** it exports an array of 9 repo objects with at minimum: `{ owner, repo, displayName, branches: { dev?, prod } }` fields

**Given** a repo has no dev branch
**When** the config is read
**Then** the `dev` field is `null` or absent — not an empty string

**Given** the old `githubActions.js` file is replaced or renamed
**When** `npm test` and `npm run build` are run
**Then** both pass with no import errors

---

## Dev Notes

- Create `src/config/releasePipeline.js` (or follow existing config file location convention)
- Example entry:
  ```js
  export const REPOS = [
    {
      owner: "electricm0nk",
      repo: "terminus-portal",
      displayName: "Terminus Portal",
      branches: { dev: "develop", prod: "main" },
    },
    {
      owner: "electricm0nk",
      repo: "terminus.infra",
      displayName: "Terminus Infra",
      branches: { prod: "main" },  // no dev branch
    },
    // ... 7 more
  ];
  ```
- If `githubActions.js` is still used by existing portal3 components, keep it and add `releasePipeline.js` as a new file; only remove `githubActions.js` if no other component imports it
- Update any imports in `ReleasePipelineTab.jsx` (story 3-03) to use the new config

---

## Definition of Done

- [ ] Code merged to feature branch (`portal4`)
- [ ] `releasePipeline.js` exports 9 repo entries with correct branch topology
- [ ] Repos with no dev branch have `dev: null` or no `dev` field (not empty string)
- [ ] `npm test` and `npm run build` pass with no import errors
- [ ] No secrets committed to source code
