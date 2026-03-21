# Domain Constitution: terminus

**Layer:** domain
**Name:** terminus
**Parent:** electricm0nk
**Ratified:** 2026-03-21T16:00:00Z
**Status:** active

---

## Preamble

This constitution governs all initiatives within the terminus domain. It inherits and preserves all org-level rules from the electricm0nk org constitution and adds domain-specific governance for service boundaries, repository strategy, and implementation structure.

---

## Inheritance Validation Record

This domain constitution is strictly additive relative to the org constitution. It does not remove, narrow, weaken, or create exceptions to any org-level article. All articles below are domain-specific extensions only.

---

## Articles

### Article 1: Repository Boundaries Default to Service Boundaries

**Rule:** Within the terminus domain, repository boundaries default to service boundaries, not feature boundaries. A service may contain multiple features, scripts, manifests, operational tooling, and supporting implementation assets within a single repository.

**Rationale:** Services are the primary unit of runtime ownership, deployment responsibility, and operational maintenance. Defaulting repository boundaries to services reduces repo sprawl, minimizes duplicated CI/CD and governance overhead, and keeps tightly related implementation assets together.

**Evidence Required:** Repository planning and architecture artifacts must show that features are grouped under their owning service unless an explicit exception has been approved. Service-level implementation assets such as scripts, manifests, Helm charts, migrations, and bootstrap tooling may live in the same repository when they support that service.

**Gate:** informational
**Status:** active

---

### Article 2: Features Do Not Imply Repositories

**Rule:** In the terminus domain, a feature initiative does not by itself justify creation of a separate repository. Features are planning, delivery, and ownership slices unless a stronger repository-boundary justification is documented and approved.

**Rationale:** Feature-level lifecycle management is useful for planning and delivery, but using each feature as a repository boundary creates unnecessary fragmentation, coordination cost, and documentation drift. Planning structure and repository structure serve different purposes and must not be conflated.

**Evidence Required:** For any new feature initiative, initiative artifacts must treat the feature as part of its parent service repository by default. If a separate repository is proposed, the proposal must include explicit written justification satisfying Article 3 of this constitution.

**Gate:** informational
**Status:** active

---

### Article 3: New Repositories Require Explicit Justification

**Rule:** A new repository may be created in the terminus domain only when explicitly justified by one or more of the following: independent deployment lifecycle, independent release/versioning needs, materially different runtime or toolchain constraints, distinct ownership or access-control boundaries, or clear cross-service reuse that would otherwise create harmful coupling.

**Rationale:** New repositories create long-term operational cost. They should exist only when they solve a real boundary problem, not as a default organizational habit.

**Evidence Required:** Any proposal for a new repository must document the reason for separation, the expected operational benefit, the ownership model, and why the code cannot remain within the parent service repository without creating material risk or friction.

**Gate:** informational
**Status:** active

---

## Ratification Record

| Date | Action | Summary |
|------|--------|---------|
| 2026-03-21T16:00:00Z | Ratified | Initial terminus domain constitution with repository-boundary governance |

---

## Inheritance

**Parent:** [org/constitution.md](/home/todd/Projects/TargetProjects/lens/lens-governance/constitutions/org/constitution.md)
**Children:** service constitutions under `constitutions/terminus/{service}/constitution.md` when created
**Inheritance rule:** All child constitutions must be strictly additive. No child may remove, weaken, or create exceptions to any org-level or terminus-domain article.
