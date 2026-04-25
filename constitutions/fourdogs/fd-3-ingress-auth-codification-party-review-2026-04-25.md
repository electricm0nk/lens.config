# FD-3 Ingress Auth Codification Party-Mode Review

Date: 2026-04-25
Scope: Codification of uniform ingress-level internal authentication for Swagger UI across Four Dogs services.

## Blind Hunter (Risk)

- Resolved: policy now requires a shared ingress-level mechanism rather than vague per-service protection.
- Resolved: anonymous Swagger UI exposure is disallowed.
- Residual warning: approved forward-auth provider governance still depends on domain-level human approval rather than a machine-enforced registry.

## Edge Case Hunter (Coverage)

- Resolved: bespoke service-local Swagger-only auth is now prohibited except through documented exception handling.
- Resolved: CI validation and post-deployment smoke testing are both called out.
- Residual warning: provider re-validation cadence is still operational policy rather than constitutional text.

## Acceptance Auditor (Enforceability)

- Resolved: auth middleware presence is now a CI concern before merge.
- Resolved: release readiness now requires a post-deployment auth smoke test.
- Resolved: exception path already exists in the domain constitution and applies to FD-3.

## Verdict

pass-with-warnings

## Warnings

1. Approved forward-auth provider membership is still a human-governed list, not a schema-validated registry.
2. Credential rotation cadence is not specified in FD-3 itself.
3. The 5-business-day escalation remains a manual governance timer.

## Recommendation

Accept this codification. If drift appears later, add a provider registry reference and credential rotation article at the domain or service-constitution level.
