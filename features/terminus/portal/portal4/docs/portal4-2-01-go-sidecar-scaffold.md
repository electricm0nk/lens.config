# Story portal4-2-01: Scaffold Go sidecar module with health endpoint

**Epic:** 2 — Go Sidecar Foundation and Helm Wiring
**Status:** backlog
**Initiative:** portal4
**Date:** 2026-05-02
**Estimate:** M

---

## User Story

As a **developer implementing the portal4 sidecar**,
I want a minimal Go module scaffold with a working `/api/healthz` endpoint,
So that subsequent sidecar stories have a clean, deployable binary to build on.

---

## Context

**Repo:** `terminus-portal` (new `sidecar/` subdirectory)
**Branch:** `portal4`

The Go sidecar is a second container in the portal pod. nginx proxies all `/api/*` requests to it on `localhost:8080`. This story creates the scaffold only — no GitHub proxy, k8s proxy, or Prometheus proxy yet.

**Architecture (ADR-1 in tech-plan):**
- Go module at `sidecar/` in the terminus-portal repo root
- Binary name: `portal-sidecar`
- Listens on `:8080`
- Routes: `GET /api/healthz` → `{"status":"ok"}`
- Reads PAT from env var `GITHUB_PAT` (set by ESO ExternalSecret in story 2-04) but does NOT require it for healthz
- Structured logging using `log/slog` (stdlib); never logs env vars or auth headers

**Security constraint (ADR-2, M-F1 from finalizeplan-review):**
The sidecar must never log the PAT value, request Authorization headers, or environment variable contents in any code path. Panic handlers must not dump env. `log/slog` with explicit field selection only.

---

## Acceptance Criteria

**Given** the sidecar is built with `go build ./...`
**When** the binary is run
**Then** it listens on `:8080` and returns `HTTP 200 {"status":"ok"}` on `GET /api/healthz`

**Given** `go vet ./...` is run against `sidecar/`
**Then** it passes with no errors

**Given** `go build ./...` is run against `sidecar/`
**Then** the binary compiles successfully

**Given** the sidecar starts with `GITHUB_PAT` env var set
**When** any request is made
**Then** the value of `GITHUB_PAT` is never emitted to logs in any code path (including startup)

**Given** `curl http://localhost:8080/api/healthz`
**Then** response is `{"status":"ok"}` with Content-Type `application/json`

---

## Dev Notes

- Create `sidecar/` directory in terminus-portal repo root
- `go mod init github.com/electricm0nk/terminus-portal/sidecar` (or match org convention)
- Use Go 1.22+; use `net/http` stdlib for the HTTP server — no framework needed for this story
- Structure: `sidecar/main.go` (entry point, server setup), `sidecar/handlers/health.go`
- Structured logging: use `log/slog` with `slog.New(slog.NewJSONHandler(os.Stdout, nil))` — fields must be explicit; no `fmt.Sprintf("%+v", os.Environ())` anywhere
- A `sidecar/go.sum` must be committed alongside `go.mod`
- This binary will be built into a distroless container in story 2-02; keep it static-binary friendly (`CGO_ENABLED=0 GOOS=linux GOARCH=amd64`)

---

## Definition of Done

- [ ] Code merged to feature branch (`portal4`)
- [ ] `go vet ./...` and `go build ./...` passing in `sidecar/`
- [ ] `GET /api/healthz` returns `{"status":"ok"}` confirmed by local test
- [ ] No env var or auth header logging in any code path
- [ ] No secrets committed to source code
