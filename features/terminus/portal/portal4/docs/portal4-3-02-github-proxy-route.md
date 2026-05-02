# Story portal4-3-02: Implement /api/github/repos/:owner/:repo/branches/:branch sidecar route

**Epic:** 3 — Tab Navigation and Release Pipeline Tab
**Status:** backlog
**Initiative:** portal4
**Date:** 2026-05-02
**Estimate:** L

---

## User Story

As a **developer building the Release Pipeline tab**,
I want a sidecar route that proxies GitHub branch data (tag, SHA, commit message, age, last CI status) for a given repo and branch,
So that the React SPA can display per-branch release status for all tracked repos without directly exposing the PAT to the browser.

---

## Context

**Repo:** `terminus-portal` (sidecar Go code)
**Branch:** `portal4`
**Depends on:** portal4-2-04 merged (PAT ExternalSecret must be wired before GitHub API calls work)

**Architecture (ADR-1, ADR-2 in tech-plan):**
- Route: `GET /api/github/repos/:owner/:repo/branches/:branch`
- Sidecar calls GitHub API using `Authorization: Bearer $GITHUB_PAT`
- Returns a JSON response to the SPA; PAT never visible to browser

**GitHub API strategy (ADR-7 in tech-plan):**
1. Try GitHub Releases API (`GET /repos/:owner/:repo/releases/latest`) for the most recent release on the branch
2. Fall back to `GET /repos/:owner/:repo/git/matching-refs/tags` sorted by semver, then filter by branch context
3. If no releases/tags: return `"tag": "no tag"` (not an error)

**Response shape:**
```json
{
  "branch": "main",
  "sha": "abc1234",
  "message": "chore: bump version",
  "age": "2 hours ago",
  "tag": "v1.2.3",
  "ciStatus": "success",  // or "failure" | "pending" | "unknown"
  "ciRunUrl": "https://github.com/org/repo/actions/runs/12345"
}
```

---

## Acceptance Criteria

**Given** the sidecar is running with a valid `GITHUB_PAT`
**When** `GET /api/github/repos/electricm0nk/terminus-portal/branches/main` is called
**Then** the response is HTTP 200 with a valid JSON body matching the response shape above

**Given** a repo has no GitHub Releases
**When** the branch route is called for that repo
**Then** the response includes `"tag": "no tag"` and does not return an error

**Given** the GitHub API returns a 403 or 429 (rate limit)
**When** the sidecar route handles it
**Then** the sidecar returns HTTP 429 to the SPA with a `{"error": "rate limited"}` body (not a panic, not a 500)

**Given** the branch does not exist
**When** the sidecar route is called
**Then** the sidecar returns HTTP 404 with a `{"error": "branch not found"}` body

**Given** `go vet ./...` and `go build ./...` are run
**Then** both pass

---

## Dev Notes

- Add `sidecar/handlers/github.go` with the route handler
- Register route in `sidecar/main.go`: `mux.HandleFunc("GET /api/github/repos/{owner}/{repo}/branches/{branch}", githubHandler)`
- Use `net/http` `PathValue` (Go 1.22 routing) to extract path params
- GitHub API calls: use `net/http` client with a 10s timeout; set `Authorization: Bearer %s` header from `os.Getenv("GITHUB_PAT")`
- Never log the PAT value; set the Authorization header programmatically without logging the header value
- CI status: call `GET /repos/:owner/:repo/commits/:sha/check-runs` and summarise the check conclusion into `"success"/"failure"/"pending"/"unknown"`
- Age calculation: parse `committer.date` from the commit response and format as human-readable relative time
- Sidecar response should be cached briefly (60s in-memory cache per route + params) to avoid GitHub rate limits on repeated dashboard refreshes — optional but recommended for homelab rate limit headroom

---

## Definition of Done

- [ ] Code merged to feature branch (`portal4`)
- [ ] `go vet ./...` and `go build ./...` passing
- [ ] `GET /api/github/repos/electricm0nk/terminus-portal/branches/main` returns valid JSON locally (with real PAT)
- [ ] No-tag repo returns `"tag": "no tag"` without error
- [ ] Rate limit (403/429) returns HTTP 429 `{"error":"rate limited"}` — not a crash
- [ ] PAT never appears in sidecar logs
- [ ] No secrets committed to source code
