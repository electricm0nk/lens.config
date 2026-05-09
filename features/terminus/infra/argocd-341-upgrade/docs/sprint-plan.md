---
title: "ArgoCD 3.4.1 Upgrade — Sprint Plan"
featureId: argocd-341-upgrade
domain: terminus
service: infra
track: express
phase: expressplan
author: Bob/SM (QuickPlan Phase 3)
created: 2026-05-09
inputDocuments:
  - docs/terminus/infra/argocd-341-upgrade/business-plan.md
  - docs/terminus/infra/argocd-341-upgrade/tech-plan.md
---

# Sprint Plan: ArgoCD 3.4.1 Upgrade

## Delivery Summary

This is an express-track infrastructure upgrade with a narrow, well-defined scope. The work sequences into three slices: code prep, execution, and verification + cleanup. Expected total effort: 1–2 working sessions.

## Prerequisites

- `kubectl` context configured to point at the K3s cluster with cluster-admin permissions
- GitHub PAT with `repo` scope available for the Ansible playbook `argocd_repo_password` extra-var
- Helm 3 CLI with `argo` repo added: `helm repo add argo https://argoproj.github.io/argo-helm && helm repo update`

## Slice Sequence

### Slice 1 — Code Prep and Version Pin (terminus.infra, argocd-341-upgrade-plan branch)

**Goal:** Produce a mergeable PR that adds the ArgoCD chart version pin and parameterizes the bootstrap playbook.

**Stories:**

**S1.1 — Add `argocd_chart_version` variable to group_vars**
- Add `argocd_chart_version: "9.5.13"` to `platforms/k3s/ansible/group_vars/all.yml`
- Mirror to `platforms/k3s/ansible/inventory/group_vars/all.yml`
- Commit and push to `argocd-341-upgrade-plan`

**S1.2 — Parameterize bootstrap-argocd.yml with chart version**
- Replace the `helm upgrade --install` command's bare invocation with `--version {{ argocd_chart_version }}`
- The `argocd_chart_version` variable resolves from group_vars
- Verify playbook YAML syntax with `ansible-playbook --syntax-check`
- Commit and push to `argocd-341-upgrade-plan`

**S1.3 — Values diff and confirm no values.yaml changes needed**
- Run: `helm show values argo/argo-cd --version 9.4.17 > /tmp/old.yaml && helm show values argo/argo-cd --version 9.5.13 > /tmp/new.yaml && diff /tmp/old.yaml /tmp/new.yaml`
- If any diff items conflict with `platforms/k3s/helm/argocd/values.yaml`, add explicit overrides to the values file
- Document the diff result as a comment in the PR description
- Commit any values.yaml changes to `argocd-341-upgrade-plan` (may be no-op)

**S1.4 — CRD schema delta check**
- Run: `helm show crds argo/argo-cd --version 9.5.13 | grep -E "^kind:|name:" | head -40`
- Confirm no new required fields in Application, ApplicationSet, or AppProject CRD schemas
- Document the result as a comment in the PR description

**Acceptance:** PR merged to `argocd-341-upgrade` base branch with all group_vars, playbook, and values changes; syntax-check passing.

---

### Slice 2 — Upgrade Execution (live cluster)

**Goal:** Run the parameterized Ansible playbook against the K3s cluster and verify the ArgoCD upgrade succeeds.

**Dependencies:** Slice 1 PR is merged to `argocd-341-upgrade` base branch. All Slice 2 stories must run from the merged `argocd-341-upgrade` branch, not the plan branch. The `argocd_chart_version` variable introduced in S1.1 must be present in the checked-out working tree before S2.3 runs the playbook.

**Stories:**

**S2.0 — Verify or restore `terminus-infra-k3s-root` app-of-apps**
- Check if the ArgoCD app-of-apps managing infra deployments exists:
  ```bash
  kubectl get app terminus-infra-k3s-root -n argocd
  ```
- If absent (it was deleted during a prior incident to clear corrupted state): re-apply from its source manifest in `platforms/k3s/argocd/apps/`
- Confirm the app-of-apps reaches Synced / Healthy before proceeding
- **This is a hard gate: do not proceed to S2.1 if the app-of-apps cannot be recovered**

**Stories:**

**S2.1 — Pre-upgrade cluster state snapshot**
- Record pre-upgrade state:
  ```bash
  helm ls -n argocd -o json
  kubectl get app -n argocd -o wide
  kubectl get pods -n argocd
  ```
- Confirm no apps are in active sync operations before starting

**S2.2 — Apply CRDs via server-side apply**
- Extract and apply CRD manifests from the target chart:
  ```bash
  helm show crds argo/argo-cd --version 9.5.13 | kubectl apply --server-side --force-conflicts -f -
  ```
- Verify no errors; confirm existing Application/ApplicationSet resources are not affected

**S2.3 — Run bootstrap-argocd.yml upgrade**
- Run the playbook from the terminus.infra repo root:
  ```bash
  ansible-playbook platforms/k3s/ansible/playbooks/bootstrap-argocd.yml \
    -i platforms/k3s/ansible/inventory/hosts.yml \
    --extra-vars "argocd_repo_password=<GITHUB_PAT>"
  ```
- The playbook waits for deployments to be Available and asserts the release is in `deployed` state
- Expected duration: 3–8 minutes

**S2.4 — Verify upgrade success**
- Check version: `helm ls -n argocd -o json` → expect `app_version: v3.4.1`, `chart: argo-cd-9.5.13`
- Check pods: `kubectl get pods -n argocd` → all Running, no CrashLoopBackOff
- Check controller logs: `kubectl logs -n argocd deploy/argocd-application-controller --tail=50` → no nil-pointer errors
- Check repo connections: verify ArgoCD repository credentials are intact
- Check app sync: `kubectl get app -n argocd` → all Synced / Healthy within 10 minutes

**Acceptance:** ArgoCD reports v3.4.1; all apps Synced / Healthy; no nil-pointer errors in controller logs.

---

### Slice 3 — Auto-Sync Re-Enablement and Cleanup

**Goal:** Restore full GitOps automation on the affected apps and close the workaround gap.

**Dependencies:** Slice 2 verified healthy.

**Stories:**

**S3.1 — Confirm auto-sync policy for kaylee apps**
- Read canonical auto-sync policy from the **source app definitions** (not live cluster state):
  ```bash
  # Find the canonical app YAML in the control repo
  find platforms/k3s/argocd/apps/ -name "*kaylee*"
  # Read syncPolicy from the source YAML, e.g.:
  grep -A5 'syncPolicy' platforms/k3s/argocd/apps/fourdogs-kaylee-agent.yaml
  ```
- The live cluster has already been patched (auto-sync disabled as a workaround). The source YAML in the repo is the canonical policy to restore.
- Confirm the intended policy (expected: `automated: {prune: true, selfHeal: true}` or as defined in the source YAML)

**S3.2 — Re-enable auto-sync on fourdogs-kaylee-agent**
  ```bash
  kubectl -n argocd patch app fourdogs-kaylee-agent --type merge \
    -p '{"spec":{"syncPolicy":{"automated":{"prune":true,"selfHeal":true}}}}'
  ```
- Wait ≥5 minutes; confirm app stays Synced / Healthy with no spurious OutOfSync events
- Check controller logs: no nil-pointer errors after sync

**S3.3 — Re-enable auto-sync on fourdogs-kaylee-agent-dev**
  ```bash
  kubectl -n argocd patch app fourdogs-kaylee-agent-dev --type merge \
    -p '{"spec":{"syncPolicy":{"automated":{"prune":true,"selfHeal":true}}}}'
  ```
- Confirm same post-sync health as S3.2

**S3.4 — Scan for other apps with disabled auto-sync**
- `kubectl get app -n argocd -o json | jq '.items[] | select(.spec.syncPolicy.automated == null) | .metadata.name'`
- For any app that had auto-sync disabled as a v3.3.6 workaround (not by design), re-enable using the same patch pattern
- Document findings

**S3.5 — Close this upgrade in Lens lifecycle**
- Commit runbook summary as `docs/terminus/infra/argocd-341-upgrade/runbook.md` (post-execution)
- Merge `argocd-341-upgrade-plan` to `argocd-341-upgrade` base branch via PR
- Trigger `/finalizeplan` in Lens to generate epics, stories, and implementation-readiness bundle

**Acceptance:** Both kaylee apps running with auto-sync re-enabled for ≥30 minutes without OutOfSync events; no nil-pointer errors in controller logs; all other affected apps restored; Lens lifecycle advanced.

---

## Dependency Map

```
Slice 1 (code prep) → Slice 2 (upgrade execution) → Slice 3 (re-enablement)
                              ↑
                  requires merged PR from Slice 1
```

## Unresolved Risks Preserved for FinalizePlan

1. **CRD schema delta** — must be checked during S1.4; if new required fields appear, additional Application/ApplicationSet patching may be needed (not yet scoped)
2. **Other auto-sync disabled apps** — S3.4 scans for additional workaround apps; count and re-enablement steps not yet finalized
3. **Values.yaml delta** — S1.3 diff may surface new chart defaults requiring explicit overrides; not yet evaluated

## Open Questions

- Are there other fourdogs ArgoCD apps (beyond kaylee-agent and kaylee-agent-dev) that had auto-sync disabled during the v3.3.6 incident period? (Answered during S3.4)
- Should the upgrade playbook be extracted into a dedicated `upgrade-argocd.yml` playbook to cleanly separate bootstrap (first-install) from upgrade (subsequent) semantics? (Defer to FinalizePlan consideration)
