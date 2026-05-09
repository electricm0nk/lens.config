---
title: "ArgoCD 3.4.1 Upgrade — Technical Plan"
featureId: argocd-341-upgrade
domain: terminus
service: infra
track: express
phase: expressplan
author: Winston/Architect (QuickPlan Phase 2)
created: 2026-05-09
inputDocuments:
  - docs/terminus/infra/argocd-341-upgrade/business-plan.md
---

# Technical Plan: ArgoCD 3.4.1 Upgrade

## Architecture Context

### Current State

ArgoCD is deployed via Helm to the `argocd` namespace on the K3s cluster:

```
Helm release:   argocd
Chart:          argo/argo-cd @ 9.4.17 (app_version: v3.3.6)
Namespace:      argocd
Revision:       3
Deployed:       2026-03-28
Install method: bootstrap-argocd.yml Ansible playbook
Version pin:    None — playbook uses `helm upgrade --install` without `--version`
```

### Target State

```
Chart:          argo/argo-cd @ 9.5.13 (app_version: v3.4.1)
Version pin:    Added — playbook passes `--version 9.5.13` explicitly
```

### Deployment Topology

- **Single-node K3s cluster** — no HA ArgoCD; controller restarts cause a brief sync pause but no data loss
- **Helm-managed** — all ArgoCD configuration is in `platforms/k3s/helm/argocd/values.yaml` and `platforms/k3s/ansible/playbooks/bootstrap-argocd.yml`
- **No `--version` pin currently** — the playbook installs latest matching the repo; the upgrade formalizes the chart version pin
- **Values file** — `platforms/k3s/helm/argocd/values.yaml` uses standard Prometheus ServiceMonitor labels and insecure-mode server; no custom upstream CRD extensions
- **Repository credentials** — stored as a Kubernetes Secret (type `repository`) in the `argocd` namespace; not managed by Helm directly; must survive the upgrade intact

## Implementation Boundaries

### What Changes

| File | Change |
|---|---|
| `platforms/k3s/ansible/playbooks/bootstrap-argocd.yml` | Add `--version 9.5.13` to the `helm upgrade --install` command |
| `platforms/k3s/ansible/group_vars/all.yml` | Add `argocd_chart_version: "9.5.13"` variable |
| `platforms/k3s/ansible/inventory/group_vars/all.yml` | Mirror: add `argocd_chart_version: "9.5.13"` |
| `docs/terminus/infra/argocd-341-upgrade/` | Feature planning and runbook artifacts (this feature) |

### What Does Not Change

- `platforms/k3s/helm/argocd/values.yaml` — no values changes expected; diff against 9.5.13 chart defaults first
- ArgoCD Application and ApplicationSet resources in `platforms/k3s/argocd/apps/`
- Namespace, RBAC, PriorityClass manifests
- Any fourdogs application repository configurations

## Technical Approach

### Upgrade Mechanism

The upgrade uses the existing Ansible playbook as its delivery path. The `helm upgrade --install` command in `bootstrap-argocd.yml` is idempotent — it will upgrade the running release to the pinned version.

**Step sequence:**
1. Pin chart version in `group_vars/all.yml` and `inventory/group_vars/all.yml`
2. Parameterize the Helm command in `bootstrap-argocd.yml` to use `--version {{ argocd_chart_version }}`
3. Perform a values diff (`helm show values argo/argo-cd --version 9.5.13`) to detect any new defaults that affect the current configuration
4. Run the playbook against the cluster: `ansible-playbook platforms/k3s/ansible/playbooks/bootstrap-argocd.yml -i ... --extra-vars "argocd_repo_password=<PAT>"`
5. Wait for all ArgoCD deployments to reach `Available` condition (the playbook already includes this wait step)
6. Verify version, sync, and re-enable auto-sync on disabled apps

### CRD Handling

ArgoCD chart upgrades can modify CRDs. The `helm upgrade` command does not update CRDs by default (Helm behavior post-v3.2). The playbook must apply CRDs explicitly before or after the upgrade if schema changes are present in 9.5.13.

**CRD upgrade approach:**
- Download the CRD manifests from the 9.5.13 chart: `helm show crds argo/argo-cd --version 9.5.13`
- Apply with `kubectl apply --server-side --force-conflicts` to update schemas without disrupting existing custom resources
- Confirm no new required fields are introduced in `Application`, `ApplicationSet`, or `AppProject` CRD schemas

### Values Diff Gate

Before running the upgrade, extract and diff values:
```bash
helm show values argo/argo-cd --version 9.4.17 > /tmp/argocd-values-current.yaml
helm show values argo/argo-cd --version 9.5.13 > /tmp/argocd-values-target.yaml
diff /tmp/argocd-values-current.yaml /tmp/argocd-values-target.yaml
```

Any new keys that do not conflict with `platforms/k3s/helm/argocd/values.yaml` are safe to accept. Conflicting defaults must be evaluated and explicitly set in the values file if they deviate from the desired configuration.

### Post-Upgrade Auto-Sync Re-Enablement

After the upgrade is verified, re-enable automated sync on the two apps that were disabled as a workaround:

```bash
kubectl -n argocd patch app fourdogs-kaylee-agent --type merge \
  -p '{"spec":{"syncPolicy":{"automated":{"prune":true,"selfHeal":true}}}}'

kubectl -n argocd patch app fourdogs-kaylee-agent-dev --type merge \
  -p '{"spec":{"syncPolicy":{"automated":{"prune":true,"selfHeal":true}}}}'
```

These patches restore the auto-sync policies that were present before the v3.3.6 incident. The canonical `prune` and `selfHeal` values **must be read from the source app definitions in `platforms/k3s/argocd/apps/`** before applying — do not read from live cluster state, which may reflect the workaround patching applied during the v3.3.6 incident.

## Data / Artifact Contracts

| Artifact | Owner | Notes |
|---|---|---|
| `platforms/k3s/ansible/group_vars/all.yml` | terminus.infra main branch | Version pin variable added |
| `platforms/k3s/ansible/inventory/group_vars/all.yml` | terminus.infra main branch | Mirror of group_vars |
| `platforms/k3s/ansible/playbooks/bootstrap-argocd.yml` | terminus.infra main branch | Parameterized `--version` flag |
| `argocd` Helm release in cluster | K3s cluster | Upgraded in-place; state preserved via Helm revision history |
| `argocd-repo-terminus-infra` Secret | K3s `argocd` namespace | Must survive upgrade; verified post-upgrade |

## Testing Strategy

| Test | Method | Pass Condition |
|---|---|---|
| Version verification | `helm ls -n argocd -o json` | `app_version: v3.4.1`, `chart: argo-cd-9.5.13` |
| ArgoCD controller health | `kubectl get pods -n argocd` | All pods Running, no CrashLoopBackOff |
| App sync status | `kubectl get app -n argocd` | All apps show Synced / Healthy within 10 min |
| Multi-source Helm rendering | `argocd app diff fourdogs-kaylee-agent` | No rendering errors; diff shows no drift |
| Auto-sync re-enabled | `kubectl get app fourdogs-kaylee-agent -o json` | `syncPolicy.automated` is non-null |
| Controller logs clean | `kubectl logs -n argocd deploy/argocd-application-controller` | No `nil pointer dereference` errors |
| Repo-server logs clean | `kubectl logs -n argocd deploy/argocd-repo-server` | No rendering errors or panics |
| ApplicationSet controller logs clean | `kubectl logs -n argocd deploy/argocd-applicationset-controller` | No errors |
| Multi-source hard-refresh render | `argocd app diff fourdogs-kaylee-agent --hard-refresh` | No rendering errors; no nil-pointer in repo-server logs |
| Repository connections | ArgoCD UI → Settings → Repositories | All repos show Connected |

## Rollout Considerations

- **Maintenance window:** Schedule during low-activity period. Controller restart causes a ~2-5 minute sync pause. All existing pods continue running normally during this window.
- **Rollback path:** Use a dynamic revision lookup to avoid hardcoded revision numbers:
  ```bash
  helm rollback argocd $(helm history argocd -n argocd -o json | jq '.[length-2].revision') -n argocd
  ```
  Apply only if the upgrade fails verification and the v3.3.6 workaround (disabled auto-sync) can be re-applied.
- **No pre-upgrade backup needed:** ArgoCD application state is defined in Git (terminus.infra). The cluster is the execution surface, not the source of truth. Repository credential secret should be confirmed healthy immediately post-upgrade as the one runtime secret not in Git.

## Known Technical Risks

| Risk | Mitigation |
|---|---|
| CRD schema change requires `--server-side` apply | Documented in implementation; apply CRDs explicitly before chart upgrade |
| New chart defaults activate unexpected behavior | Values diff gate run before upgrade; conflicts resolved in `values.yaml` |
| Repository secret lost during upgrade | Verify immediately post-upgrade with `kubectl get secret argocd-repo-terminus-infra -n argocd` |
| Auto-sync re-enablement reverts wrong policy | Confirm original `prune`/`selfHeal` flags from app definitions before patching |
| ArgoCD controller stays in restart loop on new version | Check `kubectl describe pod` in `argocd` namespace; rollback if restart loop persists beyond 5 minutes |

## Unresolved Items for FinalizePlan

- Confirm exact `prune` and `selfHeal` values for fourdogs-kaylee-agent and fourdogs-kaylee-agent-dev from the canonical app definitions before writing the re-enablement runbook
- Confirm CRD schema delta between 9.4.17 and 9.5.13 (no new required fields expected, but must be checked during dev execution)
- Determine whether any other fourdogs apps also had auto-sync disabled as a workaround and should be re-enabled post-upgrade
