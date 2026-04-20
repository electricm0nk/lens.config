---
permitted_tracks: [quickplan, full, hotfix, tech-change]
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
