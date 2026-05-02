# Story portal4-2-02: Add sidecar Dockerfile and Semaphore CI build job

**Epic:** 2 — Go Sidecar Foundation and Helm Wiring
**Status:** backlog
**Initiative:** portal4
**Date:** 2026-05-02
**Estimate:** M

---

## User Story

As a **developer deploying the portal sidecar**,
I want a Dockerfile and Semaphore CI job that build and push the sidecar image to GHCR on commit,
So that the sidecar image is always up to date and available for the Helm chart to deploy.

---

## Context

**Repo:** `terminus-portal`
**Branch:** `portal4`
**Depends on:** portal4-2-01 merged (Go sidecar scaffold must exist)

This story adds the container build pipeline for the sidecar. The sidecar image is separate from the existing portal nginx image and is pushed to GHCR under the same org.

**Architecture (ADR-1, H2 resolution from expressplan review):**
- Dockerfile at `sidecar/Dockerfile`
- Multi-stage build: `golang:1.22-alpine` builder → `gcr.io/distroless/static-debian12` runtime
- Image tag: `ghcr.io/electricm0nk/terminus-portal-sidecar:<sha>` 
- Semaphore CI: new build job in the existing Semaphore pipeline for terminus-portal; triggers on commits to `portal4` branch and `develop`/`main`

---

## Acceptance Criteria

**Given** `sidecar/Dockerfile` exists
**When** `docker build -f sidecar/Dockerfile sidecar/` is run locally
**Then** the build completes and produces a valid image that starts and responds on `:8080`

**Given** a commit is pushed to the `portal4` feature branch
**When** the Semaphore CI pipeline runs
**Then** the sidecar build job runs: `go build`, `go vet`, produces image, and pushes to `ghcr.io/electricm0nk/terminus-portal-sidecar:<sha>`

**Given** the distroless runtime image is used
**When** the container is inspected
**Then** no shell, no package manager, and no unnecessary binaries are present in the runtime layer

**Given** a `GITHUB_PAT` build arg or similar is NOT required at build time
**Then** the Dockerfile does not embed any secrets; runtime env only

---

## Dev Notes

- Dockerfile multi-stage pattern:
  ```dockerfile
  FROM golang:1.22-alpine AS builder
  WORKDIR /build
  COPY go.mod go.sum ./
  RUN go mod download
  COPY . .
  RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o portal-sidecar .

  FROM gcr.io/distroless/static-debian12
  COPY --from=builder /build/portal-sidecar /portal-sidecar
  EXPOSE 8080
  ENTRYPOINT ["/portal-sidecar"]
  ```
- Semaphore job: add to existing terminus-portal pipeline YAML. Check `.semaphore/semaphore.yml` in the terminus-portal repo for the current job structure
- Image naming: check GHCR naming convention used by other terminus services (e.g. `terminus-inference-gateway`) and follow it
- CI build job must use `docker buildx` or `docker build` — confirm with existing CI pattern in the repo
- Image tag: use `$SEMAPHORE_GIT_SHA` (or equivalent Semaphore env var) for the SHA tag

---

## Definition of Done

- [ ] Code merged to feature branch (`portal4`)
- [ ] `docker build -f sidecar/Dockerfile sidecar/` succeeds locally
- [ ] Semaphore CI job defined and validated (CI job YAML reviewed; ideally triggered on a test push to confirm it runs)
- [ ] Sidecar image pushed to GHCR on CI run
- [ ] No secrets in Dockerfile or CI job YAML
- [ ] No TypeScript/ESLint errors in portal; `npm run build` still passing
