# Service Constitution: terminus-agent

**Layer:** service
**Name:** agent
**Parent:** terminus
**Ratified:** 2026-03-21T16:00:00Z
**Status:** active

---

## Preamble

This constitution governs the `agent` service within the terminus domain. It inherits all org-level and terminus-domain rules and adds service-specific governance for agent feature boundaries, deployable workers, and service-local supporting assets.

---

## Inheritance Validation Record

This service constitution is strictly additive relative to the org and terminus domain constitutions. It does not remove, narrow, weaken, or create exceptions to any parent article.

---

## Articles

### Article 1: Agent Features Default to the Shared Agent Repository Until Deployment Boundaries Are Real

**Rule:** Agent features must default to the shared agent service repository until they become independently deployed, independently versioned, or otherwise satisfy the explicit new-repository justification rules of the terminus domain constitution.

**Rationale:** Agent features are often conceptually distinct before they are operationally distinct. Premature repository separation increases coordination cost without delivering operational benefit.

**Evidence Required:** Agent planning and architecture artifacts must identify whether a feature remains in the shared agent repository or has satisfied the documented criteria for repository separation.

**Gate:** informational
**Status:** active

---

### Article 2: Independently Deployed Agents May Become Separate Repositories

**Rule:** An agent feature may become a separate repository when it is a real independently deployed worker or service with its own runtime lifecycle, release cadence, ownership surface, or compliance boundary.

**Rationale:** Some agent features mature into standalone operational units. When the runtime boundary becomes real, the repository boundary may appropriately follow.

**Evidence Required:** Any proposal to split an agent feature into its own repository must document deployment independence, runtime ownership, release/versioning needs, and why remaining in the shared agent repository would create material operational friction.

**Gate:** informational
**Status:** active

---

### Article 3: Agent-Local Tooling Stays with the Owning Agent Service

**Rule:** Prompts, classifiers, schema files, worker glue, test fixtures, evaluation assets, scripts, delivery templates, and other supporting implementation assets that serve an agent feature must remain with the owning agent service or agent repository by default.

**Rationale:** Agent behavior depends on more than executable code. Keeping all agent-local implementation assets together preserves reproducibility, reviewability, and operational understanding.

**Evidence Required:** Agent implementations must keep feature-supporting assets in the owning service or agent repository unless a documented shared-library or cross-service reuse case justifies extraction.

**Gate:** informational
**Status:** active

---

## Ratification Record

| Date | Action | Summary |
|------|--------|---------|
| 2026-03-21T16:00:00Z | Ratified | Initial agent service constitution |

---

## Inheritance

**Parent:** [terminus/constitution.md](/home/todd/Projects/TargetProjects/lens/lens-governance/constitutions/terminus/constitution.md)
**Children:** repo constitutions under `constitutions/terminus/agent/{repo}/constitution.md` when created
**Inheritance rule:** All child constitutions must be strictly additive. No child may remove, weaken, or create exceptions to any org-, domain-, or service-level article.
