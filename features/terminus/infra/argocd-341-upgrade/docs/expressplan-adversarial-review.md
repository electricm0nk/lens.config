---
title: "ArgoCD 3.4.1 Upgrade — ExpressPlan Adversarial Review"
featureId: argocd-341-upgrade
domain: terminus
service: infra
track: express
phase: expressplan
source: phase-complete
---

# Adversarial Review: argocd-341-upgrade / expressplan

**Reviewed:** 2026-05-09T16:25:00Z
**Source:** phase-complete
**Overall Rating:** pass-with-warnings

---

## Summary

The expressplan artifact set (`business-plan.md`, `tech-plan.md`, `sprint-plan.md`) correctly identifies the root cause (ArgoCD v3.3.6 nil-pointer dereference in Helm multi-source rendering), the target fix (chart upgrade to 9.5.13 / v3.4.1), and the primary remediation path (add a version pin to the Ansible playbook and run `helm upgrade`). The scope is narrow and the technical approach is sound. Two high-severity issues were found: undefined execution sequencing (feature branch vs. merged main) and an unaccounted app-of-apps deletion from a prior incident. These are documented and must be resolved before Slice 2 cluster execution begins. No critical findings block FinalizePlan.

---

## Findings

### Critical

*None.*

---

### High

| # | Dimension | Finding | Recommendation |
|---|-----------|---------|----------------|
| H1 | Logic Flaws | **Execution sequencing is undefined.** Sprint plan Slice 2 (cluster execution via Ansible playbook) runs before Slice 3 story S3.5 merges the feature branch to the base branch via PR. The `bootstrap-argocd.yml` playbook and `argocd_chart_version` variable exist only in the feature branch during Slice 2. If the operator runs the playbook from an unmerged branch, ArgoCD's GitOps loop (which reads from `main`) would immediately drift from the cluster state after the upgrade. The plan must explicitly state whether Slice 2 runs from the feature branch (with a follow-up merge) or after merging. | Clarify in sprint plan: either (a) merge the feature branch before running Slice 2, or (b) run from the feature branch and immediately merge the PR as part of S2.3. Add a merge-first gate to S2.3. |
| H2 | Coverage Gaps | **`terminus-infra-k3s-root` app-of-apps deletion is unaddressed.** Session history documents that `terminus-infra-k3s-root` (the ArgoCD app-of-apps managing the infra deployment) was deleted earlier to clear corrupted state. If this app has not been re-applied, the cluster has no reconciliation loop for the infra ArgoCD apps after the upgrade. This is not mentioned in any planning artifact. | Add a pre-upgrade verification step in Slice 2: check if `terminus-infra-k3s-root` exists in ArgoCD. If not, re-apply it from its source manifest before proceeding with the upgrade. Document in tech plan and sprint plan. |

---

### Medium

| # | Dimension | Finding | Recommendation |
|---|-----------|---------|----------------|
| M1 | Complexity and Risk | **CRD pre-apply step is a manual kubectl command, not integrated into the playbook.** Sprint plan S2.2 runs `helm show crds ... \| kubectl apply --server-side -f -` as a manual step outside the Ansible playbook. Future upgrades will require repeating this pattern from memory. If the operator skips it, they may succeed (older CRDs may still work) but leave the cluster on a stale schema. | Add a `Apply ArgoCD CRDs from target chart version` task to `bootstrap-argocd.yml` before the `helm upgrade --install` task. Use `shell` module with `helm show crds ... \| kubectl apply --server-side --force-conflicts -f -`. |
| M2 | Logic Flaws | **Rollback command uses a hardcoded Helm revision number.** Tech plan states `helm rollback argocd 3 -n argocd`. Revision 3 was the current state at time of writing. After any upgrade run, the revision increments. The rollback command will silently roll back to the wrong state or fail if referenced blindly. | Change to a dynamic rollback: `helm rollback argocd $(helm history argocd -n argocd -o json \| jq '.[length-2].revision') -n argocd`. Document this pattern in the runbook and tech plan. |
| M3 | Coverage Gaps | **Test strategy checks only `argocd-application-controller` logs.** The nil-pointer crash bug also affects the `argocd-repo-server` (which handles Helm multi-source rendering) and the `argocd-applicationset-controller`. The upgrade could succeed without fixing the bug in those components if their logs are not checked. | Add log verification for `argocd-repo-server` and `argocd-applicationset-controller` to the test strategy in tech-plan.md. The log check should trigger a test render: `argocd app diff fourdogs-kaylee-agent --hard-refresh` to force a repo-server rendering pass. |
| M4 | Assumptions and Blind Spots | **Canonical auto-sync policy source is ambiguous.** Sprint plan S3.1 reads the sync policy from the live cluster (`kubectl get app ... -o jsonpath`). But the live cluster has already been patched (auto-sync disabled). The canonical sync policy should come from the source app YAML in `platforms/k3s/argocd/apps/`. Reading from the live cluster risks re-enabling a patched-away policy rather than the intended GitOps definition. | Identify the source app definitions in `platforms/k3s/argocd/apps/` and use those as the canonical policy to restore, not the live cluster state. Add a note to S3.1. |

---

### Low

| # | Dimension | Finding | Recommendation |
|---|-----------|---------|----------------|
| L1 | Coverage Gaps | **No upgrade window notification step.** The sprint plan has no step to notify team members (Slack, etc.) that ArgoCD will be unavailable for 2-5 minutes during Slice 2 execution. | Add a pre-execution notification step to S2.1. |
| L2 | Coverage Gaps | **TLS/certificate layer not explicitly verified post-upgrade.** ArgoCD runs in `--insecure` mode with TLS terminated at Traefik. The upgrade could affect the Traefik ingress route or ArgoCD's own TLS configuration. No test step verifies that the ArgoCD UI and API are accessible through the Traefik ingress after the upgrade. | Add to test strategy: verify `https://argocd.{cluster_domain}` returns HTTP 200 after upgrade. |

---

## Accepted Risks

- **ArgoCD controller restart gap during upgrade** (Certain / Low impact): All ArgoCD-managed apps will briefly show Unknown/OutOfSync during the controller restart. Apps are running normally on the cluster; only sync state is temporarily unavailable. Accepted without mitigation beyond scheduling during a low-activity window.
- **Helm does not update CRDs automatically** (Known behavior): Documented in tech plan; CRD pre-apply step addresses this. If CRD pre-apply is skipped, the risk is stale schema validation, not data loss.

---

## Party-Mode Challenge

**Santiago (Platform Engineer):** The sprint plan has Slice 2 cluster execution happening before the merge-to-base-branch story in S3.5. The `argocd_chart_version` variable you're adding only exists in the feature branch during Slice 2. Are you running the Ansible playbook from an unmerged branch, or merging first? If you run from the feature branch, ArgoCD will drift as soon as it reconciles from `main`. This sequencing is undefined and it's the most operationally dangerous gap in the plan.

**Petra (Site Reliability):** You disabled auto-sync on kaylee-agent and kaylee-agent-dev, but the session notes confirm `terminus-infra-k3s-root` was deleted to clear corrupted state. If that app-of-apps hasn't been re-applied, your cluster has no reconciliation loop for the infra apps after the upgrade. Your plan has no story to verify or restore it before starting. And `helm rollback argocd 3 -n argocd` — revision 3 is stale the moment you run this upgrade. That rollback command will age badly.

**Zara (Security/Ops):** Your test strategy checks `argocd-application-controller` logs for nil-pointer errors. But the Helm multi-source rendering runs in the `argocd-repo-server` and surfaces through the `argocd-applicationset-controller`. You can have a clean application-controller and still have the rendering bug active. Force a hard-refresh diff of a multi-source app after the upgrade and check the repo-server logs. Otherwise you might declare victory on the wrong signal.

---

## Gaps You May Not Have Considered

1. Is `terminus-infra-k3s-root` (the infra app-of-apps) currently missing from ArgoCD? If so, re-creating it is a Slice 2 prerequisite, not a post-upgrade cleanup.
2. Should the PR for the feature branch be merged before or after running the Ansible upgrade playbook? The answer changes whether the GitOps loop (reading from main) is in sync with the cluster state immediately after upgrade.
3. Are there fourdogs apps beyond the two kaylee apps that had auto-sync disabled during the v3.3.6 incident? S3.4 scans for this — should it also check for apps with `Degraded` sync status that may have accumulated drift?
4. Is there a monitoring alert configured for ArgoCD controller restart events? If the upgrade triggers a restart loop, how quickly would the team know?
5. Does the `argocd-repo-server` retain cached Helm rendering results across restarts? If so, the nil-pointer fix might not surface until the cache is invalidated after the upgrade.

---

## Open Questions Surfaced

1. **H1 resolution:** What is the merge-before-execute policy? Merge feature branch to base, then run playbook from `main`? Or run from branch and merge immediately after verification?
2. **H2 resolution:** What is the current state of `terminus-infra-k3s-root` in the cluster? Run `kubectl get app terminus-infra-k3s-root -n argocd` before Slice 2 begins.
3. **M4 resolution:** Where are the canonical source YAML definitions for `fourdogs-kaylee-agent` and `fourdogs-kaylee-agent-dev` app manifests? In `platforms/k3s/argocd/apps/`? What is their original `spec.syncPolicy`?

---

## FinalizePlan Guidance

FinalizePlan should integrate the following before generating epics and stories:

1. Add a `terminus-infra-k3s-root` pre-check story to Slice 2
2. Clarify merge-before-execute sequencing (H1) — update sprint-plan.md
3. Update tech-plan.md rollback command to use dynamic revision lookup (M2)
4. Add `argocd-repo-server` and `argocd-applicationset-controller` log checks to tech-plan.md test strategy (M3)
5. Update sprint S3.1 to use GitOps source YAML for canonical syncPolicy (M4)
