---
feature: portal4
doc_type: epics
status: draft
goal: "Break portal4 requirements into four implementation epics aligned with the sprint plan"
key_decisions:
  - Four epics map 1:1 to four sprints for clear milestone tracking
  - Each epic is self-contained with a deploy gate before the next begins
open_questions: []
depends_on: []
blocks: []
updated_at: "2026-05-02T00:00:00Z"
---

# Epics: portal4

## Overview

Portal4 transforms the Terminus portal from a static SPA to an operational command centre. The work is decomposed into four epics, each corresponding to one sprint.

---

## Epic 1 — Visual Fixes and Service Config

**Goal:** All known visual regressions fixed; services.js complete and accurate; portal renders correctly with no card overflow or broken icons.

**Rationale:** Fix known regressions before adding any new complexity. Sprint 1 de-risks the foundation that all subsequent sprints render on top of.

**Stories:** portal4-1-01, portal4-1-02, portal4-1-03, portal4-1-04

**Exit criteria:**
- ServiceGrid panel no longer overflows on standard viewport
- ArgoCD and Semaphore cards display correct icons
- Temporal appears as a service card
- Fourdogs card includes a dev link to `central-dev.fourdogspetsupplies.com`

---

## Epic 2 — Go Sidecar Foundation and Helm Wiring

**Goal:** A Go sidecar binary is deployed in the portal pod, accessible via nginx `/api/*` proxy. RBAC, ESO ExternalSecret for PAT, and ServiceAccount are all wired. Sidecar responds to `/api/healthz`.

**Rationale:** The sidecar is the runtime dependency for all data features in Epics 3 and 4. Must be deployed and stable before writing any proxy routes.

**Stories:** portal4-2-01, portal4-2-02, portal4-2-03, portal4-2-04, portal4-2-05

**Pre-conditions:**
- Vault PAT provisioned at `terminus/github::Portal_PAT` (confirmed present)
- GHCR org available for sidecar image

**Exit criteria:**
- `portal-sidecar` binary deployed; `/api/healthz` returns 200 from browser
- nginx proxy forwards `/api/*` to sidecar on localhost:8080
- ESO ExternalSecret `portal-github-pat` syncs from Vault
- ServiceAccount `portal-sidecar-sa` with RBAC wired for pod listing

---

## Epic 3 — Tab Navigation and Release Pipeline Tab

**Goal:** React SPA ships with three-tab navigation (Overview, Release Pipeline, Pods placeholder). Release Pipeline tab is fully functional showing per-repo, per-branch tag/SHA/CI status for all 9 configured repos.

**Rationale:** Release Pipeline tab is the highest business-value deliverable in the portal4 feature. Provides the single-pane release status view currently missing from operational workflow.

**Pre-conditions:** Epic 2 deployed (PAT ExternalSecret wired for private repo access)

**Stories:** portal4-3-01, portal4-3-02, portal4-3-03, portal4-3-04

**Exit criteria:**
- Three tabs render in SPA: Overview, Release Pipeline, Pods
- All 9 repos in releasePipeline.js show tag/SHA/CI status per branch
- Private repos resolve via PAT-proxied GitHub API
- Version watermark displayed in portal header/footer

---

## Epic 4 — Deep Health Signals and Prometheus Metrics

**Goal:** All 5 Fourdogs microservices show appropriate health signals; Prometheus metrics are live; Pods tab is fully functional.

**Rationale:** Completes the operational command-centre vision. Prometheus metrics and the Pods tab give the operator a live cluster view directly in the portal.

**Pre-conditions:** Epic 2 deployed (RBAC for pod listing); Epic 3 deployed (Pods tab renders in tab nav)

**Stories:** portal4-4-01, portal4-4-02, portal4-4-03, portal4-4-04, portal4-4-05

**Conditional dependency:** Story 4-04 (emailfetcher health signal) is conditionally blocked on `fourdogs-central /v1/health/emailfetcher` endpoint existing. If not available, emailfetcher cell shows "no signal" and story is marked partial-complete.

**Exit criteria:**
- Pods tab shows all cluster pods grouped by namespace with status and restart counts
- Overview tab MetricsPanel shows CPU load, memory usage, pods-running from live Prometheus
- FourDogsHealthPanel shows health signals for all 5 microservices
- Prometheus proxy uses internal CA TLS (not skip-verify)
