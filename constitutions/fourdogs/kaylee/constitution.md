---
permitted_tracks: [full, express, quickdev, hotfix-express, spike]
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

# kaylee Service Constitution

This constitution defines governance rules for the **kaylee** service within the `fourdogs` domain.

## Scope

Applies to all repositories within the `fourdogs/kaylee` service.
Inherits domain-level constraints and may add further restrictions.

## Tracks

All lifecycle tracks are permitted: `full`, `express`, `quickdev`, `hotfix-express`, `spike`.

## Artifacts

- **Planning phase:** a `business-plan` is required before promotion to dev.
- **Dev phase:** at least one story file must exist before dev work begins.

## Review

Peer review is enforced for all features in this service.
Additional participants may be named at the repo level.

## Notes

This constitution was initialized with service defaults.
Update it to reflect the specific governance needs of the kaylee service.
