---
permitted_tracks: [quickplan, full, express, hotfix, tech-change]
required_artifacts:
  planning:
    - business-plan
  dev:
    - stories
gate_mode: informational
sensing_gate_mode: informational
additional_review_participants: []
enforce_stories: true
enforce_review: true
---

# fourdogs Domain Constitution

This constitution defines governance rules for the **fourdogs** domain.

## Scope

Applies to all services and repositories within the `fourdogs` domain.
Lower-level constitutions (service, repo) may add constraints but may not remove those defined here.

## Tracks

All standard tracks are permitted: `quickplan`, `full`, `hotfix`, `tech-change`.
Service-level constitutions may restrict this list further.

## Artifacts

- **Planning phase:** a `business-plan` is required before promotion to dev.
- **Dev phase:** at least one story file must exist before dev work begins.

## Review

Peer review is enforced for all features in this domain.
Additional participants may be named at the service or repo level.

## Notes

This constitution was initialized with domain defaults.
Update it to reflect the specific governance needs of the fourdogs domain.

## Domain Articles (Four Dogs)

### Article FD-1: Develop-Branch Documentation Sync

Any change merged to or committed on a `develop` branch in the `fourdogs` domain must include an explicit documentation sync check for:

- `docs/fourdogs/app-arch.md`
- `docs/fourdogs/index.md`

Compliance requirement (must satisfy one):

1. update one or both files to reflect API, service, workflow, or architecture changes, or
2. add a no-op rationale in the PR description or commit message footer using:
  - `No-Doc-Sync-Required: <reason>`

Accepted no-op rationale categories:

- `(internal refactor)` with no route/topology/workflow change
- `(test-only change)`
- `(tooling/CI-only change)`

Scope trigger (changes considered doc-impacting by default):

- HTTP route additions/removals/contract changes
- service/workload additions, removals, or runtime topology changes
- data-flow, auth-flow, or operator workflow changes

Audit and ownership:

- service CODEOWNER reviewer is responsible for verifying path updates or no-op rationale
- CI must fail PR merges to `develop` when neither documentation updates nor no-op rationale are present
- missing compliance blocks merge until corrected

Intent: keep implementation and domain documentation aligned, even when this adds modest delivery friction.

### Article FD-2: Loki Logging Requirement

All new services and all new deployable feature components in the `fourdogs` domain must emit structured logs that are collectible by Loki in deployed environments.

Definitions:

- service: any deployable workload with its own runtime process (API, worker, agent, scheduled job)
- deployable component: feature code introducing a new long-running process, job, or network-facing handler

Minimum expectation:

- service emits JSON structured logs to stdout/stderr with at least: `timestamp`, `level`, `service`, `message`
- required logging fields are top-level and case-sensitive
- `level` values are one of: `DEBUG`, `INFO`, `WARN`, `ERROR`
- deployment path preserves collection into Loki via platform-approved log ingestion
- operational events are present at minimum for: `startup_ready`, `health_degraded`, and `critical_failure`

Acceptance criteria:

- `timestamp` must be UTC ISO 8601
- `startup_ready` must be emitted during startup of each deployable workload
- Loki visibility check must confirm at least one startup log line is queryable within 5 minutes of deployment start

Audit and ownership:

- service PR reviewer verifies logging fields and event coverage
- platform/infrastructure reviewer verifies Loki collection path for new service workloads
- release readiness fails if a deployable service cannot be observed in Loki

Intent: preserve observability and accelerate problem diagnosis.

### Article FD-3: API Contract Documentation Requirement

All `fourdogs` HTTP APIs must include and maintain machine-readable Swagger/OpenAPI documentation.

Definitions:

- API route: any HTTP endpoint implemented by fourdogs services
- externally consumed route: any route not matching `/internal/*`, `/health*`, or `/metrics*`, and not explicitly marked internal-only in service docs

Minimum expectation:

- each externally consumed API route is represented in OpenAPI/Swagger
- request, response, and error envelopes are documented for each route; minimum coverage includes all intentionally returned HTTP status codes for the route and example response bodies for common 4xx/5xx cases
- route contract changes include OpenAPI/Swagger updates in the same change set
- Swagger UI must be protected uniformly across fourdogs services by a shared ingress-level internal authentication mechanism
- Swagger UI must not be anonymously exposed

Specification requirement:

- API docs must be OpenAPI v3.x at one of:
  - `api/openapi.yaml`, or
  - `docs/api.openapi.yaml`
  - `docs/api/<service-name>.openapi.yaml`
- when multiple OpenAPI files exist, the service README or build configuration must declare the canonical source
- Swagger UI must render from the maintained OpenAPI source and stay in sync with the same route contract changes
- acceptable Swagger UI auth mechanisms are:
  - Traefik Basic Auth backed by Vault-managed credentials
  - an approved internal forward-auth provider

Acceptance criteria:

- versioned APIs must either maintain version-specific OpenAPI documents or a single multi-version spec with version-specific differences documented
- deprecated routes must be marked deprecated in the OpenAPI definition before code removal
- Swagger UI usability means the UI loads, renders documented operations, and supports direct request execution for authorized users in the intended internal environment
- service-local bespoke Swagger-only auth must not be introduced unless a documented exception is approved under the exception handling section

Audit and ownership:

- service PR reviewer verifies route-to-spec parity for changed endpoints
- service PR reviewer verifies Swagger UI remains reachable and usable behind the shared ingress-level internal auth mechanism
- CI must include OpenAPI validation/lint checks (for example Spectral) and fail on missing or invalid spec updates
- CI must validate ingress or Traefik auth middleware presence for Swagger UI exposure before allowing merge
- merge to `develop` is blocked when API contract changes are not documented
- unresolved FD-3 review blocks must be escalated to the domain owner after 5 business days
- post-deployment smoke testing must confirm Swagger UI requires authentication before the change is considered release-ready

## Exception Handling

Temporary exceptions to FD-1/FD-2/FD-3 require all of:

1. an issue link describing reason, scope, and remediation plan
2. explicit expiry date
3. approval by domain owner or delegated CODEOWNER

Expired exceptions are non-compliant and block further merges until resolved.

Intent: enforce contract clarity for operators, UI clients, and integration consumers.
