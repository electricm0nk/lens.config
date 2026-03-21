# Service Constitution: terminus-platform

**Layer:** service
**Name:** platform
**Parent:** terminus
**Ratified:** 2026-03-21T16:00:00Z
**Status:** active

---

## Preamble

This constitution governs the `platform` service within the terminus domain. It inherits all org-level and terminus-domain rules and adds service-specific governance for orchestration, runtime glue, schedules, and shared execution patterns.

---

## Inheritance Validation Record

This service constitution is strictly additive relative to the org and terminus domain constitutions. It does not remove, narrow, weaken, or create exceptions to any parent article.

---

## Articles

### Article 1: Platform Repository Is the Default Home for Platform Features

**Rule:** Platform features owned by the `platform` service must default to the shared platform repository unless a separate repository is explicitly justified under the terminus domain repository-boundary rules.

**Rationale:** Platform capabilities such as workflow orchestration, scheduling, shared execution conventions, and service integration glue are operationally related and should remain co-located by default.

**Evidence Required:** Planning and architecture artifacts for platform features must identify the platform service as the owning repository boundary unless an approved exception exists.

**Gate:** informational
**Status:** active

---

### Article 2: Runtime Orchestration Assets Stay with the Platform Service

**Rule:** Scheduler definitions, workflow orchestration code, platform bootstrap logic, worker registration glue, execution policies, and runtime helper scripts that support the platform service must live with the platform service by default.

**Rationale:** Platform behavior emerges from orchestration assets as much as from application code. Separating these assets from the service that owns execution policy weakens traceability and operational correctness.

**Evidence Required:** Platform implementations must store orchestration-supporting assets alongside the platform service unless a documented reuse case requires extraction.

**Gate:** informational
**Status:** active

---

### Article 3: Platform Consumes Shared Substrate, It Does Not Re-Own It

**Rule:** The platform service may consume shared substrate capabilities provided by infra, but it may not duplicate ownership of secrets management, core data stores, cluster foundations, or other infra-owned substrate services.

**Rationale:** Platform depends on substrate services, but dependency does not imply ownership. Clear ownership boundaries prevent duplicated provisioning logic and conflicting operational models.

**Evidence Required:** Platform architecture artifacts must identify consumed infra capabilities and must not restate those capabilities as platform-owned infrastructure.

**Gate:** informational
**Status:** active

---

## Ratification Record

| Date | Action | Summary |
|------|--------|---------|
| 2026-03-21T16:00:00Z | Ratified | Initial platform service constitution |

---

## Inheritance

**Parent:** [terminus/constitution.md](/home/todd/Projects/TargetProjects/lens/lens-governance/constitutions/terminus/constitution.md)
**Children:** repo constitutions under `constitutions/terminus/platform/{repo}/constitution.md` when created
**Inheritance rule:** All child constitutions must be strictly additive. No child may remove, weaken, or create exceptions to any org-, domain-, or service-level article.
