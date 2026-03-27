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

### Article 4: Internal Network Namespace

**Rule:** The canonical internal DNS domain for all terminus homelab infrastructure is `trantor.internal`. All service hostnames, VM names, and internal DNS records within the terminus domain must use this domain suffix. No alternative internal domain suffix (including `.local`, `.lan`, or `.terminus.local`) may be introduced without an explicit constitutional amendment.

**Rationale:** A single, stable internal domain namespace eliminates ambiguity in DNS configuration, cloud-init templates, TLS certificate subject names, and service discovery rules. `.local` is reserved by mDNS (RFC 6762) and causes conflicts; `trantor.internal` uses the formally reserved `.internal` TLD (RFC 9476) and is scoped to the physical host that anchors the homelab.

**Evidence Required:** OpenTofu modules, cloud-init templates, and DNS configuration that provision or reference internal hostnames must use `trantor.internal` as the domain suffix.

**Gate:** informational
**Status:** active

---

### Article 5: Direct-to-Main Development for Infrastructure and Platform Repos

**Rule:** Implementation work in `terminus.infra` and `terminus.platform` target repositories may be committed directly to `main` without creating initiative, epic, or story feature branches, and without opening pull requests. The LENS `/dev` workflow's default branching and PR pattern (initiative branch → epic branch → story branch with story→epic→initiative PRs) is **overridden** for these two repositories. All implementation commits land on `main` directly.

**Rationale:** `terminus.infra` and `terminus.platform` are single-operator homelab infrastructure repositories. The full feature-branch and PR review ceremony provides no practical safety benefit in this context — there is no second reviewer, no parallel development stream that risks merge conflict, and no deployment pipeline gated on PR approval. The overhead of maintaining initiative/epic/story branches and merging PR chains is disproportionate to the risk and actively impedes delivery cadence. Planning discipline (LENS lifecycle phases, sprint planning, story definitions) is retained; only the target-repo branching and PR mechanics are relaxed.

**Scope:** This article applies exclusively to the following target repositories:
- `terminus.infra` (`TargetProjects/terminus/infra/terminus.infra`)
- `terminus.platform` (`TargetProjects/terminus/platform/terminus.platform`)

All other terminus-domain repositories follow the default LENS branching and PR model unless a separate constitutional article grants a similar exception.

**Evidence Required:** No evidence artifact required. The presence of this article in the effective constitution for an initiative whose target repo is `terminus.infra` or `terminus.platform` is sufficient authorization.

**Gate:** informational
**Status:** active

---

### Article 6: Base-to-Main Synchronization on Every Merge

**Rule:** When a pull request merges work into the `base` audience branch of any terminus-domain target repository, the same merged content must also be merged into `main` as part of the same lifecycle event. The `main` branch must never lag behind `base`. Synchronization merges into `main` must be non-destructive: the merge must be a standard merge commit that incorporates any divergent `main` content. Force-pushes, rebases onto `main`, and history rewrites that could discard commits already present on `main` are prohibited.

**Rationale:** The `base` audience branch accumulates finalized, reviewed work from all feature and service initiatives. If `main` is not kept current with `base`, future work branched from `main` diverges from the authoritative state, creating increasingly expensive merges, silent behavioral regressions, and ambiguous groundtruth for operators and developers. Synchronizing at each merge event — rather than periodically — prevents divergence from compounding. The non-destructive constraint protects `main` from losing hotfixes, operator commits, or any other commits that legitimately reach `main` outside the initiative cycle.

**Scope:** This article applies to all terminus-domain target repositories that use the LENS initiative branch model (including a `base` audience branch). It does not apply to `terminus.infra` and `terminus.platform`, which are governed by Article 5 of this constitution (direct-to-main commits; no `base` branch separation).

**Evidence Required:** For each PR that merges into a `base` audience branch, a corresponding merge commit or PR targeting `main` must exist and reference the source base merge. The merge to `main` must be a standard merge commit — no force-push or rebase.

**Gate:** informational
**Status:** active

---

## Ratification Record

| Date | Action | Summary |
|------|--------|---------|
| 2026-03-21T16:00:00Z | Ratified | Initial terminus domain constitution with repository-boundary governance |
| 2026-03-25T00:00:00Z | Amended | Article 4: establish `trantor.internal` as canonical internal DNS domain |
| 2026-03-26T00:00:00Z | Amended | Article 5: permit direct-to-main development for terminus.infra and terminus.platform |
| 2026-03-27T00:00:00Z | Amended | Added Article 6: Base-to-Main Synchronization on Every Merge |

---

## Inheritance

**Parent:** [org/constitution.md](/home/todd/Projects/TargetProjects/lens/lens-governance/constitutions/org/constitution.md)
**Children:** service constitutions under `constitutions/terminus/{service}/constitution.md` when created
**Inheritance rule:** All child constitutions must be strictly additive. No child may remove, weaken, or create exceptions to any org-level or terminus-domain article.
