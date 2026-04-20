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

# fourdogs / eTailPet API Service Constitution

Inherits from the `fourdogs` domain constitution.
Lower-level constitutions (repo) may add constraints but may not remove those defined here.

## Scope

Applies to all repositories within the `etailpet-api` service of the `fourdogs` domain.

## Tracks

Inherits all permitted tracks from the domain: `quickplan`, `full`, `hotfix`, `tech-change`.
Repo-level constitutions may restrict this list further.

## Artifacts

Inherits domain-level artifact requirements:
- **Planning phase:** `business-plan`
- **Dev phase:** `stories`

Add service-specific artifact requirements below as needed.

## Review

Peer review is enforced. Additional participants may be named at the repo level.

## Notes

This constitution was initialized with service defaults inherited from the `fourdogs` domain.
Update it to reflect the specific governance needs of the `etailpet-api` service.
