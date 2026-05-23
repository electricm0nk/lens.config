---
feature: hermes-platform-honcho-setup
doc_type: retrospective
status: approved
author: Todd Hintzmann
date: "2026-05-23"
---

# Retrospective — Honcho Setup (hermes/platform)

**Feature:** hermes-platform-honcho-setup  
**Domain / Service:** hermes / platform  
**Track:** express  
**Epic:** E1 — Deploy Honcho Memory Service on hermes VM  
**Stories Completed:** HP-1 through HP-8 (8/8, 6.5 pts)  
**Target Repo:** terminus.infra  
**Retrospective Date:** 2026-05-23

---

## Outcome Summary

**Status: Success.** All 8 stories completed. Honcho is running as a native Python service on `hermes.trantor.internal`, backed by PostgreSQL with pgvector, managed by Ansible/systemd, monitored via Grafana Alloy/Loki, and verified with a passing smoke test suite. Every goal from the sprint goal was met.

---

## What Went Well

### 1. Scope stayed clean
All 8 stories were scoped right. Nothing was added mid-sprint, nothing was deferred. The dependency chain (HP-1 → HP-2 → HP-3 → HP-4 → HP-5 → {HP-6, HP-7, HP-8}) held up exactly as planned.

### 2. Ansible role structure was idempotent from the start
The role decomposition (honcho-install, honcho-systemd, honcho-postgres-provision) was clean and reusable. The second CI run on the same environment produced `0 changed` on all state tasks.

### 3. Security posture was right by design
Loopback-only binding (`127.0.0.1:8000`), env file at mode `0600`, secrets via Vault, no external exposure, no auth bypass. The architecture held up without surprises.

### 4. Runbook and monitoring shipped as part of the feature
HP-7 (runbook) and HP-8 (Alloy monitoring wiring) were not deferred. They shipped in the same sprint as the service itself. This is the correct pattern.

### 5. Sprint plan and story files were accurate
The pre-sprint estimates (6.5 pts total) reflected actual scope reasonably. The story files gave the implementation agent enough context to work without needing to re-discover requirements.

---

## What Could Have Been Better

### 1. honcho-ai (client SDK) vs honcho server (GitHub repo) confusion — 3 CI failures
**What happened:** The initial install role used `uv pip install honcho-ai==2.1.1`. This installed the client SDK, not the Honcho server. The server (`plastic-labs/honcho`) must be cloned from GitHub and run with `fastapi run`.

**Impact:** Three CI run failures before the root cause was identified. Required a full redesign of the honcho-install role.

**Root cause:** PyPI search returned `honcho-ai` prominently; the packaging boundary between client SDK and server was not documented prominently in the Honcho README.

**Action:** Document this pattern — when a service has a PyPI SDK *and* a server, always verify the deployment path from the source repo's `Dockerfile` or `README` before writing the install role.

### 2. pydantic-settings nested env var naming was not obvious
**What happened:** Honcho uses pydantic-settings with a nested `DBSettings` model. The database connection URL env var is `DB__CONNECTION_URI` (double-underscore separator), not `DATABASE_URL`. This pattern is pydantic-settings-specific and is not surfaced in most web docs.

**Impact:** One CI failure after the server was correctly cloned and running — alembic migrations silently connected to `localhost:5432` instead of the configured Postgres host.

**Root cause:** `no_log: true` on the migration task hid the error output. The pydantic-settings env var naming convention was not checked during architecture review.

**Action:** Add a check to the architecture review process: for any Python service using pydantic-settings, read `src/config.py` (or equivalent) to identify the actual env var names before writing the `.env` template.

### 3. pgvector not installed on Postgres VM — out-of-band manual fix required
**What happened:** The Postgres VM (`postgres-primary`) did not have `postgresql-17-pgvector` installed. The `CREATE EXTENSION vector` task in the Ansible playbook failed with `Extension vector is not available`.

**Impact:** One CI failure. Required SSH access to postgres-primary and manual `apt install postgresql-17-pgvector` to unblock CI.

**Root cause:** The `terminus.infra` postgres VM provisioning does not install pgvector by default. This is an infra prerequisite that was not documented or automated.

**Action item (open):** Add `postgresql-17-pgvector` to the postgres-primary VM provisioning playbook/role so it is available for any future feature that requires vector storage.

### 4. `no_log: true` on migration task masked root cause
**What happened:** The alembic migration task had `no_log: true`, which suppressed the error output. Debugging required a manual test run with the env var set to observe the actual alembic connection behavior.

**Action:** Use `no_log: true` only on tasks that handle plaintext secrets (e.g., password writes). Migration tasks that use connection strings should log normally, or use `no_log: false` explicitly when debugging is likely to be needed.

---

## Lessons Learned

| # | Lesson | Category |
|---|--------|----------|
| 1 | For any Python service, verify deployment method from source `Dockerfile`/`README` before writing install role. Never assume PyPI package = server. | Infra/Ansible |
| 2 | For pydantic-settings apps, read `src/config.py` to identify actual env var names (double-underscore nested model separators). | Infra/Ansible |
| 3 | pgvector is an infra prerequisite — add to postgres-primary VM provisioning so future features don't hit this. | Platform Infra |
| 4 | Use `no_log: true` only on tasks that write plaintext secrets, not on migration or health-check tasks. | Ansible discipline |
| 5 | Loopback + systemd + uv venv is a viable, clean deployment pattern for small API services on hermes. No Docker/k3s overhead needed. | Architecture |

---

## Action Items

| Priority | Action | Owner | Status |
|---|---|---|---|
| High | Add `postgresql-17-pgvector` to postgres-primary VM provisioning role | Todd | Open |
| Medium | Pin `honcho_repo_tag` to a specific version tag (currently `main`) for reproducibility | Todd | Open |
| Low | Remove `no_log: true` from non-secret Ansible tasks (migration, health-check) | Todd | Open |

---

## Feature Success Criteria — Final Verification

| Criterion | Status |
|---|---|
| Honcho service `active (running)` via `systemctl status honcho` | ✅ Verified |
| `curl http://127.0.0.1:8000/health` returns HTTP 200 | ✅ Verified |
| Full smoke test (app, user, session, message write/read) passes | ✅ Verified |
| Alloy ships `{vm="hermes", service="honcho"}` logs to Loki | ✅ Verified |
| Ansible playbook idempotent (second run `0 changed`) | ✅ Verified |
| Runbook covers all standard operating procedures | ✅ Delivered (HP-7) |
