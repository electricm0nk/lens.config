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

# Watchdog Service Constitution

This constitution defines governance rules for the **Watchdog** service within the `terminus` domain.

## Scope

Applies to all repositories within the `terminus/watchdog` service.
Inherits domain-level constraints and may add further restrictions.

## Tracks

All lifecycle tracks are permitted: `full`, `express`, `quickdev`, `hotfix-express`, `spike`.

## Artifacts

- **Planning phase:** a `business-plan` is required before promotion to dev.
- **Dev phase:** at least one story file must exist before dev work begins.

## Review

Peer review is enforced for all features in this service.
Additional participants may be named at the repo level.

## Deployment Model

**Production only — no dev environment.**

The Watchdog service runs a single production deployment. There is no dev namespace, dev ArgoCD
application, or dev values file in use. `values-dev.yaml` in `terminus.infra` must not be
configured or promoted to.

## Release Flow

All code changes must be delivered via **PR → merge to `main`**. Direct commits to `main` are not
permitted for feature or fix work. After merging, a release is triggered by pushing a semver tag:

```
git tag v<major>.<minor>.<patch>
git push origin v<major>.<minor>.<patch>
```

The release workflow (`release.yml`) builds and pushes the Docker image to GHCR, then promotes
the semver tag into `terminus.infra/platforms/k3s/helm/terminus-watchdog/values.yaml` on `main`.
ArgoCD syncs automatically from there.

**No Temporal ReleaseWorkflow** is used for this service. ArgoCD is the sole deployment mechanism.

## Notes

This constitution was initialized with service defaults.
Update it to reflect the specific governance needs of the Watchdog service.
