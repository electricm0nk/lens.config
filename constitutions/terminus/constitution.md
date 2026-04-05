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

### Article 7: Local-First AI Data Sovereignty

**Rule:** All AI inference and processing within the terminus domain must operate on a local-first basis. Confidential, personal, or sensitive data must not leave the internal network (`trantor.internal`) for inference, embedding, or processing by any external AI service provider. Requests containing confidential data must be routed exclusively to locally hosted models. Routing logic that could leak confidential data to external providers must be identified and gated in architecture review. An exception to this rule requires a valid AI safety exception under org Article 15 with explicit data-flow justification.

**Rationale:** Terminus is a homelab platform that may handle personal and operational data. The inference gateway's entire value proposition is local execution. Allowing confidential data to reach external providers would contradict the architectural intent, expose data outside operator control, and violate the data-sovereignty commitment to services that consume the gateway.

**Evidence Required:** Architecture and implementation artifacts for AI components must include explicit data-flow statements identifying all processing paths and confirming confidential data is scoped to `trantor.internal` infrastructure. Infrastructure configuration must not permit external AI API calls on routes that handle confidential data without a committed and valid org Article 15 exception.

**Gate:** informational
**Status:** active

---

### Article 8: AI Component Promotion Gate

**Rule:** Any change to an AI-facing interface, model configuration, routing policy, batch execution policy, or inference behavior requires a promotion gate confirmation before advancing to the next audience tier. This gate must verify: (a) all route profiles are present and compliant with service-layer requirements; (b) no regression in gateway contract tests; (c) all active AI safety exceptions under org Article 15 are approved and unexpired. Changes that affect only non-AI components follow the standard terminus promotion path.

**Rationale:** AI behavioral changes can have subtle, non-obvious effects on downstream consumers. A dedicated promotion gate for AI-affecting changes ensures that contract stability, safety obligations, and exception hygiene are verified before reaching a wider audience — independent of the general release gate.

**Evidence Required:** Promotion artifacts for AI-affecting changes must include confirmation that route profiles are complete, gateway contract tests pass, and no unapproved or expired AI safety exceptions are on record for the initiative.

**Gate:** informational
**Status:** active

---

### Article 9: AI Gateway Contract Stability

**Rule:** The inference gateway within the terminus domain exposes one stable API contract to platform consumers. Provider-specific logic, model selection, retry strategies, timeout behavior, and fallback mechanics must be encapsulated in provider adapters behind the gateway interface. Platform consumers may not depend on provider-specific behaviors, response structures, or error formats. A breaking change to the gateway contract requires a formal contract version increment and a compatibility window documented in the initiative artifacts before deployment.

**Rationale:** Contract instability at the inference gateway propagates breakage to all platform consumers simultaneously. A single stable contract with provider logic isolated in adapters allows models, providers, and routing strategies to evolve without impacting consumers. It also enables provider migration and cost substitution without consumer code changes.

**Evidence Required:** Architecture artifacts for inference-adjacent platform features must reference the gateway contract as the integration boundary. No consumer code may import or reference provider-specific types, endpoints, or response shapes directly. When a breaking gateway contract change is introduced, initiative artifacts must document the version increment and compatibility window.

**Gate:** informational
**Status:** active

---

### Article 10: AI Telemetry and Audit Baseline

**Rule:** All AI inference components within the terminus domain must emit structured telemetry for every request, including at minimum: route identifier, model used, request start time, duration, token counts (input and output where measurable), outcome (success, fallback, or error), and cost tag. Batch operations must additionally emit checkpoint events at each processing checkpoint. Telemetry must be retained locally and queryable. No inference request may silently fail or silently fall back without a logged outcome record.

**Rationale:** Inference components are opaque execution environments. Without telemetry, capacity planning, cost attribution, failure diagnosis, and safety auditing are impossible. Silent failures and silent fallbacks are especially dangerous in AI contexts because they can produce plausible-looking incorrect output without any visible error signal.

**Evidence Required:** Implementation artifacts for inference components must identify the telemetry emission points, the log destination, and the retention mechanism. Reviews must confirm that all route outcomes — including fallbacks and errors — produce logged records. Batch implementations must commit checkpoint telemetry to persistent storage in a form that allows resume-on-failure.

**Gate:** informational
**Status:** active

---

### Article 11: AI Infrastructure Separation

**Rule:** Infrastructure services own the AI substrate: GPU passthrough configuration, model weights storage, hardware scheduling, runtime containers, and cluster-level resource policies. Inference services own AI behavior: route definitions, model selection policy, prompt templates, guardrails, fallback logic, and request lifecycle controls. Neither layer may absorb ownership responsibilities belonging to the other. Infra must expose substrate capabilities through documented contracts. Inference must consume substrate capabilities via those contracts without reaching into infra internals.

**Rationale:** Mixing substrate ownership with behavioral ownership creates two failure modes: infra changes break inference behavior silently, and inference changes require infra modifications beyond their scope. Separation of substrate and behavior allows each layer to evolve, be replaced, or be upgraded independently. It also clarifies maintenance responsibility: if a GPU fails, that is an infra incident; if a route returns wrong results, that is an inference incident.

**Evidence Required:** Architecture artifacts for AI initiatives must explicitly map each component to its owning layer (infra or inference) and must not place behavioral logic in infra-owned assets or substrate configuration in inference-owned assets. Infra-to-inference capability contracts must be documented in architecture files before implementation begins.

**Gate:** informational
**Status:** active

---

### Article 12: k3s Deployment Completeness

**Rule:** For any service initiative within the terminus domain that introduces a network-reachable service endpoint, the following deployment artifacts must be committed and applied to the k3s cluster before the initiative is considered complete:

1. **ArgoCD `Application` manifest** — committed to `terminus.infra` under `argocd/apps/`, pointing at the service's Helm chart in its source repository. The application must target the correct destination namespace and sync policy.
2. **Namespace manifest** — a `Namespace` resource committed to `terminus.infra` under `argocd/namespaces/` or inline in the app-of-apps kustomization, ensuring the namespace exists before the application syncs.
3. **Traefik `IngressRoute`** — an `IngressRoute` resource with a `*.trantor.internal` hostname (Article 4 of this constitution), committed to `terminus.infra` or bundled in the service's Helm chart under an `ingressRoute` toggle.
4. **`cert-manager` `Certificate`** — a `Certificate` resource targeting the same `*.trantor.internal` hostname, referencing the cluster `ClusterIssuer`, committed alongside the `IngressRoute`.
5. **GHCR `imagePullSecret` documented** — if the service image is hosted on a private container registry, the `imagePullSecret` name and its provisioning procedure (manual `kubectl create secret docker-registry` or External Secrets operator path) must be documented in the service's `runbook.md` before the sprint closes.
6. **Smoke-test acceptance criterion** — at least one story in the deployment sprint must include an AC of the form: `GET https://{service}.trantor.internal/{healthz-path}` returns `200 OK` from within the cluster network.

**Rationale:** This article is the terminus-domain implementation of org Article 18 (Service Delivery Includes Deployment). The k3s cluster uses ArgoCD for GitOps delivery, Traefik as the ingress controller, and cert-manager for TLS. Without these six artifacts, a service merged to `main` is unreachable. The explicit artifact checklist removes ambiguity about what "deployed" means in this environment and prevents the gateway-style gap where a complete, tested service sits unrouted for an indefinite period.

**Scope:** Applies to all terminus-domain service and feature initiatives deploying to the k3s cluster. Does not apply to initiatives that are exclusively infra-substrate changes (cluster provisioning, Vault configuration, DNS setup) with no new service endpoint.

**Exception:** If any of the six artifacts cannot be completed in the same sprint, the exception mechanism in org Article 18 applies: a `.todo` entry must be filed before sprint close identifying the missing artifact, the dependency, the owner, and a target date.

**Evidence Required:** Sprint artifacts must include stories covering each of the six items above. Each story's acceptance criteria must reference the specific artifact being created. The `runbook.md` for the service must include a section titled "Deployment Checklist" enumerating all six items with confirmation status.

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
| 2026-04-03T00:00:00Z | Amended | Added Articles 7–11: AI governance for the terminus domain — local-first data sovereignty, AI component promotion gate, gateway contract stability, telemetry/audit baseline, and AI infrastructure separation |
| 2026-04-04T00:00:00Z | Amended | Added Article 12: k3s Deployment Completeness — terminus-domain implementation of org Article 18; mandates ArgoCD Application, Namespace, IngressRoute, Certificate, imagePullSecret runbook entry, and smoke-test AC before a service initiative is complete |

---

## Inheritance

**Parent:** [org/constitution.md](/home/todd/Projects/TargetProjects/lens/lens-governance/constitutions/org/constitution.md)
**Children:** service constitutions under `constitutions/terminus/{service}/constitution.md` when created
**Inheritance rule:** All child constitutions must be strictly additive. No child may remove, weaken, or create exceptions to any org-level or terminus-domain article.
