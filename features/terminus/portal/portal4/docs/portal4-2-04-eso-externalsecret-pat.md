# Story portal4-2-04: Wire PAT via Vault ESO ExternalSecret in Helm chart

**Epic:** 2 — Go Sidecar Foundation and Helm Wiring
**Status:** backlog
**Initiative:** portal4
**Date:** 2026-05-02
**Estimate:** L

---

## User Story

As a **developer deploying the portal sidecar**,
I want the GitHub PAT injected into the sidecar via a Vault-backed ESO ExternalSecret,
So that the sidecar can authenticate to the GitHub API for private repo access without any secrets in code or Helm values.

---

## Context

**Repo:** `terminus-portal`
**Branch:** `portal4`
**Depends on:** portal4-2-03 merged (Helm chart must exist with sidecar container placeholder env)

**Pre-condition (operator):** Vault PAT provisioned at `terminus/github::Portal_PAT` — confirmed present before finalizeplan entry.

**Architecture (ADR-2 in tech-plan):**
- ESO `ExternalSecret` `portal-github-pat` in `terminus-portal` namespace
- Syncs from `ClusterSecretStore` (ESO cluster-level, already configured in cluster)
- Vault path: `terminus/github`, key: `Portal_PAT`
- Produces k8s Secret `portal-github-pat` with key `GITHUB_PAT`
- Sidecar container mounts the Secret as env var `GITHUB_PAT`

**Security constraint (ADR-2):** Sidecar must never log `GITHUB_PAT` or any env var contents. This was explicitly added to the sidecar scaffold story (2-01) and must be re-verified when wiring the env var.

---

## Acceptance Criteria

**Given** the `ExternalSecret` manifest is applied to the cluster
**When** ESO reconciles it
**Then** a k8s Secret `portal-github-pat` appears in the `terminus-portal` namespace with a non-empty `GITHUB_PAT` key

**Given** the sidecar container is deployed with the updated Helm chart
**When** the container is running
**Then** `GITHUB_PAT` env var is available inside the sidecar container (verify via `kubectl exec` or sidecar startup log)

**Given** the sidecar is running with `GITHUB_PAT` set
**When** sidecar logs are inspected via `kubectl logs`
**Then** the value of `GITHUB_PAT` does not appear in any log line

**Given** `helm template` is run
**Then** the ExternalSecret and sidecar env var mount appear in rendered output without hardcoded secret values

---

## Dev Notes

- Add an ESO `ExternalSecret` manifest to the Helm chart (as a template or as a separate manifest in `argocd/` depending on existing pattern)
  ```yaml
  apiVersion: external-secrets.io/v1beta1
  kind: ExternalSecret
  metadata:
    name: portal-github-pat
    namespace: terminus-portal
  spec:
    refreshInterval: 1h
    secretStoreRef:
      name: vault-cluster-secret-store   # confirm actual ClusterSecretStore name from cluster
      kind: ClusterSecretStore
    target:
      name: portal-github-pat
      creationPolicy: Owner
    data:
      - secretKey: GITHUB_PAT
        remoteRef:
          key: terminus/github
          property: Portal_PAT
  ```
- Update sidecar container in Helm chart deployment template:
  ```yaml
  env:
    - name: GITHUB_PAT
      valueFrom:
        secretKeyRef:
          name: portal-github-pat
          key: GITHUB_PAT
  ```
- Confirm the `ClusterSecretStore` name used in the cluster by checking an existing ESO ExternalSecret (e.g. from terminus-inference-gateway or fourdogs-kaylee-agent)
- After applying, verify with: `kubectl get secret portal-github-pat -n terminus-portal -o jsonpath='{.data.GITHUB_PAT}' | base64 -d | wc -c` (should be non-zero)

---

## Definition of Done

- [ ] Code merged to feature branch (`portal4`)
- [ ] `helm template` renders ExternalSecret and sidecar env var mount without errors
- [ ] k8s Secret `portal-github-pat` synced from Vault in dev cluster (verified via `kubectl get secret`)
- [ ] `GITHUB_PAT` env var accessible inside sidecar container (verified via `kubectl exec` or startup log)
- [ ] PAT value not visible in any sidecar log line
- [ ] No hardcoded PAT value in any committed file
- [ ] ArgoCD dev app syncs Healthy
