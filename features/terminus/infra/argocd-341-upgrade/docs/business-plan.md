---
title: "ArgoCD 3.4.1 Upgrade — Business Plan"
featureId: argocd-341-upgrade
domain: terminus
service: infra
track: express
phase: expressplan
author: John/PM (QuickPlan Phase 1)
created: 2026-05-09
---

# Business Plan: ArgoCD 3.4.1 Upgrade

## Problem Statement

The Terminus infrastructure cluster is running ArgoCD v3.3.6, which contains a confirmed nil-pointer dereference crash bug in its Helm multi-source template rendering engine. This bug manifests as:

- ArgoCD application controllers crashing when processing `ApplicationSet` or `Application` resources that use multiple Helm chart sources
- Apps reporting `OutOfSync` status even when deployed workloads are running correctly and health checks pass
- Operator intervention required to manually restore sync state after each controller restart

The bug is actively impacting production workloads. The fourdogs-kaylee-agent and fourdogs-kaylee-agent-dev ArgoCD apps have had automated sync permanently disabled as a temporary workaround, increasing operational risk and reducing the cluster's self-healing capability.

## Target Users / Stakeholders

| Stakeholder | Impact |
|---|---|
| **Platform operators (Todd Hintzmann)** | Eliminating repeated manual intervention and restoring confidence in GitOps automation |
| **Development teams (fourdogs)** | Restoring reliable auto-sync for all multi-source Helm apps; reducing deployment friction |
| **On-call rotation** | Removing a recurring incident trigger tied to ArgoCD controller restarts |

## Business Goals

1. **Restore GitOps reliability**: Re-enable automated sync on all fourdogs ArgoCD apps after the upgrade, closing the workaround gap opened during the v3.3.6 incident period.
2. **Eliminate a recurring operational incident**: The nil-pointer crash is a known upstream bug fixed in v3.4.1. The upgrade removes the root cause, not just the symptom.
3. **Preserve multi-source Helm compatibility**: All existing ApplicationSet and Application resources that use multi-source Helm must continue to render and sync correctly after the upgrade.
4. **Maintain zero data-loss upgrade path**: ArgoCD state (applications, app-of-apps topology, repository credentials, RBAC) must be preserved across the upgrade.
5. **Keep cluster at a supported upstream release**: v3.3.6 is no longer the latest patch; staying current reduces exposure to additional unpatched issues.

## Scope

### In Scope

- Upgrade ArgoCD Helm chart from v3.3.6 to v3.4.1 in the terminus infra K3s cluster
- Validate multi-source Helm rendering against existing fourdogs Application resources post-upgrade
- Re-enable automated sync on fourdogs-kaylee-agent and fourdogs-kaylee-agent-dev after confirmed upgrade health
- Update any pinned chart version references in the terminus.infra control repo
- Document the upgrade procedure and verification steps as a runbook entry

### Out of Scope

- Migration to a different GitOps operator (e.g., Flux)
- Changes to ArgoCD RBAC, SSO, or repository credential configuration
- Upgrading Kubernetes itself or K3s
- Changes to fourdogs Application or ApplicationSet definitions (unless required for compatibility)
- High-availability ArgoCD configuration (single-node deployment remains in scope)

## Dependencies

| Dependency | Status | Notes |
|---|---|---|
| ArgoCD v3.4.1 Helm chart availability | Confirmed | Available in official ArgoCD Helm repo |
| terminus.infra repo access | Active | Branch `argocd-341-upgrade-plan` created |
| K3s cluster admin access | Active | Available via kubectl with kubeconfig |
| Existing ArgoCD app definitions (fourdogs) | In use | Must be validated post-upgrade |
| External Secrets Operator (ESO) | Running | Not affected by ArgoCD upgrade |
| fourdogs-kaylee-agent auto-sync re-enablement | Post-upgrade gate | Re-enable only after verified upgrade health |

## Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Breaking change in v3.4.1 CRD schema | Low | High | Review v3.4.1 release notes for CRD changes; apply `kubectl apply --server-side` with force-conflicts if needed |
| Helm chart value API changes | Low-Medium | Medium | Diff current values against new chart defaults before applying |
| ArgoCD controller restart gap during upgrade | Certain | Low | Expected transient; apps remain running, sync paused during upgrade window |
| Git authentication credential loss | Low | High | Verify repository connections immediately after upgrade |
| App-of-apps topology disruption | Low | High | Verify root app sync status within 5 minutes of upgrade completion |

## Success Criteria

- [ ] ArgoCD reports version 3.4.1 in the UI and via `argocd version`
- [ ] All ArgoCD-managed apps return to `Synced / Healthy` status within 10 minutes of upgrade completion
- [ ] fourdogs-kaylee-agent and fourdogs-kaylee-agent-dev have automated sync re-enabled and remain healthy for ≥30 minutes
- [ ] No nil-pointer dereference errors appear in ArgoCD controller logs
- [ ] Multi-source Helm template rendering completes without errors for all affected apps
- [ ] No application credential or repository connections are lost

## Delivery Expectations

This is an express-track infrastructure upgrade. The scope is narrow, the fix is upstream-confirmed, and the delivery window is a single controlled change to the Helm chart version in the terminus.infra repo. Expected end-to-end delivery: 1–2 working sessions including upgrade execution and post-upgrade verification.
