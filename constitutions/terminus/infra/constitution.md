# Service Constitution: terminus-infra

**Layer:** service
**Name:** infra
**Parent:** terminus
**Ratified:** 2026-03-21T16:00:00Z
**Status:** active

---

## Preamble

This constitution governs the `infra` service within the terminus domain. It inherits all org-level and terminus-domain rules and adds service-specific governance for infrastructure implementation, operational assets, and stateful substrate ownership.

---

## Inheritance Validation Record

This service constitution is strictly additive relative to the org and terminus domain constitutions. It does not remove, narrow, weaken, or create exceptions to any parent article.

---

## Articles

### Article 1: Infra Repository Is the Default Home for Infra Features

**Rule:** Infrastructure features owned by the `infra` service must default to the shared infra repository unless a separate repository is explicitly justified under the terminus domain repository-boundary rules.

**Rationale:** The infra service owns tightly related operational assets including cluster substrate, secret systems, stateful backing services, and provisioning logic. Keeping these assets together reduces fragmentation and preserves operational context.

**Evidence Required:** Planning and architecture artifacts for infra features must identify the infra service as the owning repository boundary unless an approved exception exists.

**Gate:** informational
**Status:** active

---

### Article 2: Operational Tooling Belongs with the Infra Service

**Rule:** Scripts, manifests, Helm charts, provisioning definitions, bootstrap helpers, migration tooling, backup jobs, restore helpers, and runbooks that support the infra service must live with the infra service by default.

**Rationale:** Operational infrastructure is not only application code. It includes execution assets required to provision, repair, back up, recover, rotate, and audit the environment. Splitting these artifacts away from the infra service creates brittle operational gaps.

**Evidence Required:** Infra implementations must store service-supporting operational assets alongside the infra service unless a documented reuse case requires extraction.

**Gate:** informational
**Status:** active

---

### Article 3: Shared Substrate Services Are Owned Centrally

**Rule:** Shared substrate capabilities such as Kubernetes foundations, secrets management, data stores, ingress, certificate management, and other cross-service infrastructure must be owned as infra service capabilities, not duplicated inside consuming services.

**Rationale:** Shared substrate duplication leads to inconsistent security posture, inconsistent operations, and higher recovery risk. Cross-service infrastructure should have one owning service with explicit consumers.

**Evidence Required:** Architecture artifacts for infra and consuming services must identify a single owner for each shared substrate capability and must not duplicate that ownership in sibling services.

**Gate:** informational
**Status:** active

---

## Ratification Record

| Date | Action | Summary |
|------|--------|---------|
| 2026-03-21T16:00:00Z | Ratified | Initial infra service constitution |

---

## Inheritance

**Parent:** [terminus/constitution.md](/home/todd/Projects/TargetProjects/lens/lens-governance/constitutions/terminus/constitution.md)
**Children:** repo constitutions under `constitutions/terminus/infra/{repo}/constitution.md` when created
**Inheritance rule:** All child constitutions must be strictly additive. No child may remove, weaken, or create exceptions to any org-, domain-, or service-level article.
