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

### Article 12: Agent Entry Point Parity

**Rule:** When multiple AI coding assistants are actively used in this workspace, each must have its session-start entry point file present at the workspace root. These files must maintain structural parity: identical module reference paths, activation sequences, and skill tables. An entry point file present for one assistant but absent for another is a knowledge rot defect. A diverged entry point file is an equal defect.

**Corollary — Preflight Is the Sync Gate:** The release module (`lens.core/`) maintains canonical versions of all workspace-root agent entry point files. Preflight is responsible for syncing these files from the release module to the workspace root on every run. Changes to one entry point file in the release module must include a corresponding update to all peer entry point files in the same commit or PR.

**Rationale:** Different AI coding assistants discover session context from different entry point filenames (`AGENTS.md` for GitHub Copilot/Codex, `CLAUDE.md` for Claude Code, and equivalents for other tools). If each assistant begins a session from a diverged entry point, agents will have inconsistent awareness of project structure, active initiative, constitutional obligations, and available commands. Structural parity ensures every assistant session begins with identical foundational context. This is a direct application of the Knowledge Rot corollary of Article 10: a missing or diverged entry point file is stale documentation that actively harms output quality.

**Evidence Required:**
- All actively used assistant entry point files exist at the workspace root.
- Content comparison between peer files confirms identical module reference paths, activation sequences, and skill references.
- Preflight sync covers all entry point files from the release module.
- When a canonical entry point file is updated in the release module, all peer files are updated in the same commit or PR.

**Gate:** informational
**Status:** active

---

### Article 13: IDE Adapter Installation and Verification

**Rule:** When a LENS Workbench module is installed in a workspace, the IDE-specific command adapter for each actively used AI coding assistant must be installed. For Claude Code, the adapter is the `.claude/commands/` directory containing command stub files generated by `lens.core/_bmad/lens-work/scripts/install.sh --ide claude`. For other IDEs, equivalent adapter directories apply. Preflight must verify that all expected adapters are present on every run and must re-execute the installer in idempotent mode when any adapter directory is missing.

**Corollary — Adapters Enable Commands:** IDE adapters translate LENS Workbench lifecycle commands into native IDE slash-command affordances. Without them, phase commands such as `/status`, `/constitution`, and `/dev` are not discoverable through normal IDE interaction. An agent that cannot surface its own commands is not fully operational, regardless of whether its entry point files (Article 12) are correct.

**Corollary — Installer Is the Authority:** The `install.sh` script in the release module (`lens.core/_bmad/lens-work/scripts/install.sh`) is the single authority for adapter content. Manually created or modified command stubs are not authoritative. The installer is idempotent and safe to re-run on every preflight.

**Rationale:** During the terminus-infra-postgres initiative, the Claude Code adapter (`.claude/commands/`) was absent from the workspace. Preflight reported clean and activation succeeded — but no slash commands were discoverable through the IDE. The diagnosis required manual inspection; the fix required a manual installer invocation. A workspace where agents appear ready but commands are unreachable through IDE affordances is in a degraded operational state that the constitution must prevent.

**Evidence Required:**
- `.claude/commands/` exists and contains command stub files when Claude Code is an active assistant.
- Preflight log confirms an adapter presence check was performed on the session.
- When an adapter was absent before a preflight run, the preflight log records that the installer was invoked.

**Gate:** informational
**Status:** active

---

### Article 14: Automated Phase PR Acceptance (Solo Operator)

**Rule:** For solo-operator organizations, lens-work planning lifecycle phase branch PRs must be automatically created and merged without requiring manual review or intervention. The `auto-merge-phase.sh` script must be used in place of `create-pr.sh` for all phase completion steps.

**Rationale:** Requiring manual PR review for planning artifact gate merges in a single-contributor environment introduces unnecessary ceremony with no review benefit. The artifact content is collaboratively produced with the agent during the phase workflow; the PR is a state-management record, not a human review gate. Blocking lifecycle advancement on a manual merge step in a solo context defeats the purpose of automation.

**Scope:** Applies exclusively to lens-work planning lifecycle phase branches (`{initiative-root}-{audience}-{phase}` → `{initiative-root}-{audience}`). Does not apply to production code, infrastructure changes, or target project repositories.

**Implementation:** `_bmad/lens-work/scripts/auto-merge-phase.sh`. Uses GitHub API + PAT when available; falls back to local `git merge --no-ff` + push.

**Evidence Required:** `_bmad-output/lens-work/personal/profile.yaml` is present (confirms this is a managed workspace, not an ad-hoc environment).

**Gate:** informational
**Status:** active

---

### Article 15: AI Safety Primacy

**Rule:** Safety and security are first-order requirements for all AI-related work. No speed, convenience, cost, feature velocity, or delivery timeline consideration may override a known or unmitigated safety or security risk. The default position when risk is unknown or unmitigated is: **do not proceed**. This default may only be overridden by an approved exception with complete exception artifacts committed to the project repository.

**Exception Standard:** An exception to this article — where known or unmitigated risk is accepted — is valid only when all of the following artifacts are committed to the project repository before the work proceeds:

1. **Decision statement:** explicit description of the risk being accepted and the reason the work must proceed despite it.
2. **Threat and impact analysis:** documented assessment of the threat model, failure modes, and potential impact scope.
3. **Mitigations:** concrete measures in place or planned before the excepted code or configuration ships.
4. **Expiry date:** a specific date after which the exception must be re-evaluated or revoked.
5. **Named approver:** the person who authorized the exception.
6. **Rollback plan:** the approach to revert the change if the risk materializes.
7. **Verification plan:** how and when the mitigations will be confirmed as effective.

**Corollary — Incomplete Exceptions Are Void:** An exception lacking any of the seven required artifacts is automatically void and does not authorize the work to proceed. Partial exception artifacts provide no protection.

**Promotion Gate:** Work may not be promoted to any audience tier while an unapproved or expired AI safety exception exists on the initiative. Exception artifacts must be current and complete before promotion unblocks.

**Rationale:** AI systems introduce distinct risk profiles — including opaque failure modes, data leakage surfaces, adversarial input vulnerabilities, and unbounded compute behaviors — that make a default-safe posture essential. The cost of an AI safety failure is typically asymmetric: cheap to prevent early, expensive or irreversible once materialized. Treating safety as optional or deferrable, even under schedule pressure, converts technical risk into organizational risk.

**Evidence Required:** Where an AI safety exception has been invoked, all seven exception artifacts must exist as committed files (or committed sections of existing architecture or decision files) in the active initiative branch. Architecture and design documents for AI components must include explicit safety and data-handling sections. Code review must confirm that no known unmitigated risks were introduced without an approved exception on record.

**Gate:** informational
**Status:** active

---

### Article 16: Internal Infrastructure Primacy

**Rule:** All initiatives must default to the organization's internally built and operated infrastructure for compute, orchestration, storage, data persistence, networking, secrets management, AI inference, and operational tooling. Externally hosted services, SaaS platforms, and third-party providers are non-default options. Any decision to use an external service or tool in place of an available internal equivalent must be documented as a `.todo` entry in the initiative's `.todo` folder (see Article 17) before that decision is acted upon. The entry must remain open and unresolved until explicitly reviewed and accepted.

**Rationale:** The terminus domain exists to provide a self-hosted infrastructure stack under full operator control. Defaulting to internal infrastructure preserves data sovereignty, reinforces platform investment, and prevents silent accumulation of external dependencies that erode the organization's operational independence. Requiring a `.todo` review entry for each deviation creates a lightweight but durable audit trail without blocking delivery.

**Evidence Required:** Architecture and implementation artifacts must enumerate all infrastructure dependencies and confirm each is sourced from the internal terminus stack where an internal equivalent exists. Where an external service is used, a corresponding `.todo` entry must exist documenting: the external dependency chosen, the internal alternative considered, and the reason for the deviation. The entry must reference this article and must not be marked resolved until it has been reviewed.

**Gate:** informational
**Status:** active

---

### Article 17: .todo Folder as Workflow Discovery Log

**Rule:** Every domain, service, and feature initiative must maintain a `.todo` folder at its initiative artifact root (`_bmad-output/lens-work/initiatives/{domain}/`, `_bmad-output/lens-work/initiatives/{domain}/{service}/`, or `_bmad-output/lens-work/initiatives/{domain}/{service}/{feature}/` as appropriate). This folder is the designated capture point for unresolved findings discovered during any phase of the lifecycle — including defects, half-formed ideas, enhancement requests, architecture concerns, deferred decisions, and internal infrastructure deviation notices (see Article 16). Each entry is a separate `.md` file named with a short slug and creation date (e.g., `2026-04-03-vault-auth-edge-case.md`). Entries accumulate throughout the lifecycle and must be reviewed at each phase gate. An initiative may not reach `done` state while any `.todo` entry is marked unreviewed.

**Entry Format:** Each `.todo` entry must contain at minimum:
- **Label:** one of `defect`, `idea`, `enhancement`, `architecture-concern`, `deferred-decision`, or `infra-deviation`
- **Date:** ISO 8601 date the entry was added
- **Description:** one or more sentences describing the finding
- **Status:** `open` or `reviewed` (with reviewer note when reviewed)

**Rationale:** Discoveries happen throughout development — not only during planning. A universal, low-friction capture point ensures observations are recorded at the moment they arise rather than being lost in chat threads or forgotten between sessions. By giving every initiative level a `.todo` folder and anchoring review to phase gates, the organization prevents deferred thoughts from silently dropping out of the workflow.

**Evidence Required:** The `.todo/` folder must be present within the initiative artifact path for any initiative in active development. Phase gate artifacts must include a `.todo` review summary confirming all entries have been assessed and their statuses updated. Entries added by agents during implementation must cite the source story or task.

**Gate:** informational
**Status:** active

---

### Article 18: Service Delivery Includes Deployment

**Rule:** A service initiative is not complete until the service can receive traffic in its target environment. Deployment artifacts — including cluster manifests, ingress configuration, secrets provisioning, and any required infrastructure wiring — must be scoped into the same sprint as the service functionality they support. Deferring deployment to a follow-on initiative or a separate infrastructure ticket is prohibited unless an explicit, documented exception is approved and committed to the initiative artifacts before the service sprint closes.

**Rationale:** A service that is "done" but not reachable provides no value and creates a false sense of delivery completeness. The gap between a tested, merged service and a live, routable endpoint is not a trivial handoff — it requires coordination, environment-specific configuration, and validation that can surface integration issues invisible to unit and contract tests. When deployment is treated as someone else's problem, those issues are discovered later, at higher cost, and often by the wrong person. The only honest definition of "done" for a service is: traffic can reach it.

**Scope:** Applies to all service and feature initiatives that introduce or modify a network-reachable service endpoint. Infrastructure-only initiatives (e.g., cluster provisioning, secret store setup) are exempt from this article. Existing services receiving only internal refactoring changes that do not alter their deployment model are also exempt.

**Exception Standard:** If deployment cannot be completed in the same sprint — due to an unresolved infrastructure dependency, environment readiness gate, or operator decision — a `.todo` entry (see Article 17) must be filed before the sprint closes specifying: the missing deployment component, the dependency blocking it, the owner, and a target date. The entry must remain open until deployment is confirmed.

**Evidence Required:** Initiative sprint artifacts must include at least one story or task scoped to deployment completeness for each service endpoint introduced. The story must cover: cluster/namespace manifests, ingress or route configuration, secrets provisioning runbook entries, and a smoke-test acceptance criterion confirming the endpoint is reachable in the target environment. Where a deployment exception has been filed, the `.todo` entry must be present and unresolved at sprint close.

**Gate:** informational
**Status:** active

---

### Article 19: Semaphore UI — Imperative Job Runner in the Terminus Platform

**Rule:** Semaphore UI is the designated imperative job runner in the terminus ecosystem and must be used exclusively for work that cannot be expressed as a continuously reconcilable manifest. ArgoCD is exclusively responsible for all GitOps reconciliation — detecting desired-state manifests in Git and applying them to the cluster. These two tools are not interchangeable and must not be used in each other's roles.

**The Reconciliation Test:** Before assigning ownership of any task, apply the following test: *Can ArgoCD reconcile this back to desired state if it drifts?* If yes — the task belongs in a manifest and ArgoCD owns it. If no — the task is inherently imperative and Semaphore owns it.

**Semaphore's Canonical Responsibilities:**
- Seeding secrets into Vault (before ESO/ArgoCD can take over)
- Running database schema migrations (ordered, one-time, imperative)
- Seeding initial or reference data into databases
- External service configuration that cannot live in a manifest (webhooks, API keys, firewall rules, DNS entries)
- Operational day-2 jobs (backups, restores, cleanup, smoke tests)

**Standard Workload Deployment Sequence:**
1. Semaphore — run schema migrations
2. Semaphore — seed secrets into Vault
3. Git — Helm chart present in `apps/<workload>/`
4. ArgoCD — detects chart, deploys container
5. ESO — pulls secret from Vault into k8s Secret
6. Pod starts with schema and secrets available

**What Semaphore Must NOT Do:**
- Deploy containers directly to k3s (ArgoCD owns that)
- Watch Git and react to commits (ArgoCD owns that)
- Replace Crossplane for infrastructure provisioning (Crossplane owns that)

**Blast-and-Repave Rule:** All Semaphore job definitions must live in the repository. All migration files must live under `apps/<workload>/migrations/`. Nothing Semaphore does may require tribal knowledge to reconstruct. A Semaphore-owned operation that cannot be reproduced from committed artifacts alone is a violation of this article.

**Rationale:** Without an explicit boundary, Semaphore and ArgoCD responsibilities silently overlap — teams reach for whichever tool is familiar rather than applying the correct ownership model. Imperative steps executed outside GitOps become invisible to reconciliation, producing cluster state that diverges from declared intent without ArgoCD detecting it. Equally, attempting to force inherently imperative operations (secrets seeding, migrations) into declarative ArgoCD sync waves creates ordering fragility and hidden dependencies. The reconciliation test provides a deterministic decision rule that keeps the division of labor clean and auditable.

**Evidence Required:** Architecture and implementation artifacts for any workload that uses Semaphore must enumerate each Semaphore-managed operation and confirm it passes the reconciliation test (i.e., ArgoCD cannot own it). All Semaphore job definitions must be committed to the repository. Where Semaphore and ArgoCD interact in a deployment sequence, the sequence must be explicitly documented in the workload's `runbook.md` (see Article 11).

**Gate:** informational
**Status:** active

---

### Article 20: Go Is the Default Runtime for Platform Services and Temporal Workers

**Rule:** Go is the default runtime for all platform services and Temporal workers. Any initiative that introduces a new platform service or Temporal worker using a different runtime must include explicit documented justification in its architecture artifacts, and that justification must be approved before the techplan phase gate is passed.

**Rationale:** Runtime consistency reduces operational overhead, lowers cognitive burden for contributors moving between services, and ensures a uniform set of tooling, observability, and build practices across the platform. Exceptions fragment the platform without proportionate benefit and must therefore be explicitly justified — not defaulted to.

**Evidence Required:** Architecture documentation for any non-Go platform service or Temporal worker must contain a language decision record identifying the chosen runtime, the rationale for the deviation, and the approver. Initiatives using Go require no additional evidence for this rule.

**Gate:** informational
**Status:** active

---

## Ratification Record

| Date | Action | Summary |
|------|--------|---------|
| 2026-03-21T16:00:00Z | Ratified | Initial constitution — 8 articles |
| 2026-04-02T00:00:00Z | Amended | Articles 9–13 added; Article 14 added — Automated Phase PR Acceptance (Solo Operator) |
| 2026-03-21T16:00:00Z | Amended | Added Article 9: Security First |
| 2026-03-27T00:00:00Z | Amended | Added Article 10: Repository Is the Agent's Source of Truth |
| 2026-03-28T00:00:00Z | Amended | Added Article 11: Prefer Source-Correction; Always Maintain a Runbook |
| 2026-03-28T01:00:00Z | Amended | Article 11: named gap manifest file as `pre-repave-manifest.md`; added cross-reference requirements between runbook.md and pre-repave-manifest.md; added Resolved Gaps retention rule |
| 2026-03-27T00:00:00Z | Amended | Added Article 12: Agent Entry Point Parity — requires structural alignment between all assistant entry point files; preflight is the enforcement gate |
| 2026-03-28T02:00:00Z | Amended | Added Article 13: IDE Adapter Installation and Verification — requires `.claude/commands/` (and equivalent) be installed and verified by preflight on every run |
| 2026-03-28T00:00:00Z | Amended | Added Article 14: Epic-Scoped PR Discipline — PRs created per epic, not per story; all stories committed to single epic branch; PR opened only when epic is complete |
| 2026-03-28T00:00:00Z | Reverted | Removed Article 14: Epic-Scoped PR Discipline — superseded by terminus domain constitution Article 5 (2026-03-26), which already authorizes direct-to-main development for terminus.infra and terminus.platform; Article 14 was added in error and contradicted existing domain governance |
| 2026-04-04T00:00:00Z | Amended | Added Article 18: Service Delivery Includes Deployment — a service initiative is not complete until the service can receive traffic; deployment artifacts must be scoped in the same sprint as the service functionality |
| 2026-04-03T00:00:00Z | Amended | Added Article 15: AI Safety Primacy — safety/security is first principle for all AI work; default is do not proceed when risk is unknown or unmitigated; exceptions require seven committed artifacts; incomplete exceptions are void; promotion gated while any unapproved or expired exception exists |
| 2026-04-03T00:00:00Z | Amended | Added Article 16: Internal Infrastructure Primacy — all initiatives default to internally built infrastructure; external service deviations must be documented as open .todo entries before proceeding |
| 2026-04-03T00:00:00Z | Amended | Added Article 17: .todo Folder as Workflow Discovery Log — every initiative level requires a .todo/ folder for capturing defects, ideas, enhancements, and deferred decisions; reviewed at each phase gate; initiative cannot reach done with unreviewed entries |
| 2026-04-04T00:00:00Z | Amended | Added Article 19: Semaphore UI — Imperative Job Runner in the Terminus Platform — establishes Semaphore as the exclusive imperative job runner, defines the reconciliation test for ArgoCD vs Semaphore ownership, enumerates Semaphore's canonical responsibilities and the standard workload deployment sequence, enforces blast-and-repave rule requiring all job definitions and migration files in the repository |

---

## Inheritance

**Parent:** none (org is root)
**Children:** all domain constitutions under `constitutions/{domain}/constitution.md`
**Inheritance rule:** All child constitutions must be strictly additive. No child may remove, weaken, or create exceptions to any article above.
