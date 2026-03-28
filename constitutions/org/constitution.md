# Org Constitution: electricm0nk

**Layer:** org
**Name:** electricm0nk
**Ratified:** 2026-03-21T16:00:00Z
**Status:** active

---

## Preamble

This constitution governs all software initiatives under the electricm0nk organization. It ensures consistent lifecycle discipline, artifact standards, and quality gates across all domains and services. All child domain, service, and repo constitutions must remain additive to these rules.

---

## Articles

### Article 1: Track Declaration Required

**Rule:** Every initiative must declare a LENS lifecycle track (full, feature, tech-change, hotfix, or spike) before any phase work begins.

**Rationale:** Ensures consistent scoping and gates are applied from the start.

**Evidence Required:** `initiative.yaml` contains a non-empty `track` field.

**Gate:** informational
**Status:** active

---

### Article 2: Phase Artifacts Before Gate

**Rule:** Required artifacts for each phase must exist and be non-empty before a phase completion gate is approved.

**Rationale:** Prevents empty planning artifacts from slipping through lifecycle gates.

**Evidence Required:** Phase artifact files exist and contain substantive content at the time of gate evaluation.

**Gate:** informational
**Status:** active

---

### Article 3: Architecture Documentation Required

**Rule:** Any initiative introducing new infrastructure, services, or integration patterns must include architecture documentation in the techplan phase.

**Rationale:** Ensures design decisions are captured and reviewable, not just implemented.

**Evidence Required:** `architecture.md` exists and is non-empty in the techplan phase artifacts.

**Gate:** informational
**Status:** active

---

### Article 4: No Confidential Data Exfiltration

**Rule:** No initiative may introduce a dependency that transmits confidential or personal data outside the local network without explicit documented justification.

**Rationale:** Data privacy is a non-negotiable org-wide constraint. No cloud SaaS dependency may handle confidential or personal data without explicit review.

**Evidence Required:** Architecture documentation contains explicit data flow statements identifying all external data transmission paths and their justification.

**Gate:** informational
**Status:** active

---

### Article 5: Git Discipline

**Rule:** All initiative work occurs on named initiative branches. Direct commits to `main` are prohibited. The only permitted path to `main` is a reviewed and approved pull request.

**Rationale:** Maintains traceability, review discipline, and rollback capability for all production changes.

**Evidence Required:** PR-based merge is the only path to `main`; branch protection rules enforce this.

**Gate:** informational
**Status:** active

---

### Article 6: Additive Governance

**Rule:** Child constitutions — at domain, service, or repo layers — may only add governance rules. They may never remove, weaken, narrow, or create exceptions to any article ratified at a higher layer. An article title is immutable after ratification. Amendments that contradict a parent article are void.

**Rationale:** The inheritance chain derives its authority from permanence. If child layers could undermine org rules, the org constitution would offer no real protection. Governance must be reliable across all governed work.

**Evidence Required:** Any constitution amendment or new child constitution must include an inheritance validation record confirming no parent rule was weakened or removed.

**Gate:** informational
**Status:** active

---

### Article 7: TDD Red-Green Discipline

**Rule:** Test-Driven Development (TDD) red-green discipline is mandatory for all production code. Every unit of new or modified behavior must begin with a failing test (Red), followed by the minimum production code to make it pass (Green), followed by refactoring with tests remaining green. No production code may be written without a prior failing test for that behavior. Skipping this cycle — including writing tests after the fact — is a violation of this article.

**Rationale:** Red-green TDD is not a quality preference; it is a design discipline. It forces clarity of intent before implementation, catches regressions at the moment they are introduced, and ensures that the test suite is causally linked to the code it governs. Post-hoc tests cannot provide these guarantees.

**Evidence Required:** Commit history must demonstrate the red-green cycle (failing test commit precedes passing implementation commit) for each story or task. CI gates must include a test coverage check showing all new paths are covered.

**Gate:** informational
**Status:** active

---

### Article 8: BDD Acceptance Criteria — Fully Implemented Tests

**Rule:** Every story's acceptance criteria must have a corresponding, fully implemented BDD test. Tests must be written before or in lockstep with implementation following red-green practices. Stub tests, skipped tests, pending tests, placeholder tests, or tests with empty bodies are prohibited. A story is not done until all its acceptance criteria are verified by passing, non-stub BDD tests that exercise real production behavior.

**Rationale:** Acceptance criteria represent the contract between product and engineering. If that contract is expressed as test stubs, it is not verified — it is deferred. Deferred verification is deferred risk. The organization has determined that "done" means the contract is provably satisfied.

**Evidence Required:** BDD test suite must have one or more fully implemented, passing scenarios per acceptance criterion. No pending, skip, xit, xdescribe, empty `it()`, or equivalent stub constructs may appear in the accepted test suite. Code review must confirm no stubs bypass this rule.

**Gate:** informational
**Status:** active

---

### Article 9: Security First

**Rule:** Security is a first-order requirement in every initiative. Secrets must never be stored in source control. All implementations must follow established security best practices by default. Shortcuts that weaken credential handling, access control, encryption, auditability, or other core security controls are prohibited.

**Rationale:** Security failures are high-impact, hard to contain, and often irreversible once exposed. Treating security as optional or deferrable creates systemic risk across every domain, service, and repository in the organization.

**Evidence Required:** Architecture and implementation artifacts must identify secret handling, trust boundaries, credential storage approach, and applicable security controls. Code review must confirm that no secrets are committed and that the implementation follows documented security best-practice patterns appropriate to the technology in use.

**Gate:** informational
**Status:** active

---

---

### Article 10: Repository Is the Agent's Source of Truth

**Rule:** Any context that an agent needs to perform initiative work — decisions, rationale, architectural patterns, operating principles, conventions, and domain knowledge — must be committed to the repository. Context that lives only in chat threads, verbal discussions, external documents, or human memory is invisible to agents and therefore does not govern their output. When a decision is made or a pattern is established, it must be encoded in repo-local, versioned artifacts before it can be considered authoritative.

**Corollary — Maps Over Manuals:** Agent-facing knowledge files (such as `AGENTS.md`, `copilot-instructions.md`, or equivalent) must function as navigational maps — short, stable entry points with pointers to deeper sources of truth — not as monolithic instruction manuals. A knowledge file that attempts to encode every rule in a single blob will crowd out task context, become unmaintainable, and rot into inaccuracy. Knowledge must be distributed across appropriately scoped, cross-linked, versioned artifacts.

**Corollary — Knowledge Rot Is a Defect:** Documentation or guidance that no longer reflects real code behavior is a defect, not a cosmetic issue. Stale agent-facing knowledge actively harms output quality by causing agents to operate on false premises. Keeping agent-visible knowledge current is engineering work of the same priority class as keeping code correct.

**Rationale:** Agents can only reason over what is in their context at runtime. Any knowledge not encoded in the repository effectively does not exist to an agent — the same way undocumented decisions are invisible to a new hire who joins months later. Distributing knowledge into structured, repo-local artifacts makes it discoverable, versionable, reviewable, and durable across model upgrades and team changes. This is the foundation that allows agent-assisted work to remain coherent and aligned with human intent over time.

**Evidence Required:** Initiative artifacts and architecture documentation must be committed to the repository. Significant decisions that shape implementation — including rationale for patterns, exceptions, and tradeoffs — must appear in committed documents rather than external sources. Agent-facing knowledge files must use a map-and-pointer structure rather than a single monolithic file.

**Gate:** informational
**Status:** active

---

### Article 11: Prefer Source-Correction; Always Maintain a Runbook

**Rule:** When an agent is asked to build, fix, or implement any component, it must prefer correcting the authoritative source artifact (code, configuration, template, script) over workarounds, patches, or ad-hoc instructions. When a source correction is not possible — due to external constraints, upstream dependencies, or environment-specific state — the gap must be recorded in a `pre-repave-manifest.md` file committed to the active branch before the session ends. In addition, every infrastructure or service component must be covered by a `runbook.md` that documents the component's purpose, its dependencies, and the exact commands to rebuild it from scratch. The runbook must be reviewed and updated at the end of every session that touches the component, even if only to confirm its accuracy.

**Rationale:** The organization runs with heavy agent assistance. An agent that works around a broken source leaves the repository in a state that will confuse all future agents and engineers. An agent that fixes the root cause extends the canonical knowledge baseline. Where root-cause fixes are not possible, recorded gaps are the minimum acceptable residue — undocumented gaps compound silently. Runbooks serve the same principle applied to operational knowledge: a component that cannot be rebuilt from committed documentation is a single-point-of-failure for human and agent alike.

**Evidence Required:**
- Implementation commits must target source artifacts (templates, scripts, configs) rather than one-off environment patches where possible.
- When a source fix is not possible, a `pre-repave-manifest.md` file committed to the active branch must describe: what each gap is, why a source fix was not made, and what manual steps were taken. Resolved gaps remain in the file under a **Resolved Gaps** section.
- A `runbook.md` file must exist for each infrastructure or service component introduced or materially changed by an initiative. It must contain: the component's purpose, its dependencies, and a rebuild-from-scratch command sequence.
- The `runbook.md` must include a `Last Verified` timestamp updated each session. It must cross-reference the `pre-repave-manifest.md` when one exists.
- The `pre-repave-manifest.md` must cross-reference the `runbook.md`.

**Gate:** informational
**Status:** active

---

## Ratification Record

| Date | Action | Summary |
|------|--------|---------|
| 2026-03-21T16:00:00Z | Ratified | Initial constitution — 8 articles |
| 2026-03-21T16:00:00Z | Amended | Added Article 9: Security First |
| 2026-03-27T00:00:00Z | Amended | Added Article 10: Repository Is the Agent's Source of Truth |
| 2026-03-28T00:00:00Z | Amended | Added Article 11: Prefer Source-Correction; Always Maintain a Runbook |
| 2026-03-28T01:00:00Z | Amended | Article 11: named gap manifest file as `pre-repave-manifest.md`; added cross-reference requirements between runbook.md and pre-repave-manifest.md; added Resolved Gaps retention rule |

---

## Inheritance

**Parent:** none (org is root)
**Children:** all domain constitutions under `constitutions/{domain}/constitution.md`
**Inheritance rule:** All child constitutions must be strictly additive. No child may remove, weaken, or create exceptions to any article above.
