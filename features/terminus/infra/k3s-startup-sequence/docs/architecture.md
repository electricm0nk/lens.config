---
feature: k3s-startup-sequence
doc_type: architecture
status: draft
goal: "Establish a deterministic startup sequence for all ArgoCD-managed pods on the k3s cluster using sync-wave annotations"
key_decisions:
  - ArgoCD sync waves are the sole mechanism for controlling Application startup order within the App-of-Apps topology
  - Wave 0 is reserved for cluster primitives with no annotation (metallb, cert-manager, external-secrets, traefik, crossplane)
  - Temporal server and workers must be healthy before CI/CD tooling (Actions runners, Semaphore) deploys
  - All application workloads (fourdogs, inference, monitoring, portals) are deferred to wave 6 — after CI/CD infra is healthy
  - All -infra apps (namespace/RBAC scaffolding) must hold explicit wave 1 annotations; wave 0 infra apps create a race with the external-secrets operator
  - All ingress Applications move to a unified wave 7 to enforce app-before-ingress ordering at the ArgoCD level
open_questions: []
assumptions:
  - ArgoCD is bootstrapped by Ansible before any Application resources are created; it is not managed within the App-of-Apps
  - ArgoCD automated sync with selfHeal is enabled on the root-app; waves advance only when the previous wave's Application objects report Healthy
  - All Application objects in platforms/k3s/argocd/apps/ are managed by the same root-app (terminus-infra-k3s-root)
  - External-secrets operator (ESO) must be Healthy before any -infra app that creates ExternalSecret resources
depends_on:
  - k3s (cluster substrate — MetalLB, cert-manager, Traefik, ArgoCD bootstrap)
  - proxmox-vm-boot-order (VM-level boot ordering — prerequisite at the hypervisor layer)
blocks: []
updated_at: "2026-04-17"
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
---

# Architecture: K3s Pod Startup Sequence

**Feature:** `k3s-startup-sequence`  
**Track:** tech-change  
**Domain/Service:** terminus / infra  

---

## Problem Statement

The k3s cluster's App-of-Apps topology (managed by `terminus-infra-k3s-root`) deploys all ArgoCD `Application` objects simultaneously unless sync-wave annotations are set. The current wave assignments are incomplete and inconsistent:

| Problem | Impact |
|---------|--------|
| `semaphoreui` has **no wave annotation** (defaults to wave 0) | deploys alongside MetalLB, before Temporal is running |
| `fourdogs-*` apps (6 objects) are **wave 1** | deploy alongside namespace scaffolding, before Temporal server or workers are ready |
| `actions-runner` is **wave 2** — same wave as `temporal-server` | deploys before Temporal workers (wave 3) are Healthy |
| `-infra` apps for fourdogs, kaylee, portal have **no wave annotation** | race with the external-secrets operator at wave 0 |
| `temporal-infra` has **no wave annotation** (defaults to wave 0) | ExternalSecrets in the manifest race with the ESO operator |
| Ingress Applications are spread across waves 3 and 4 with no consistent rule | creates implicit ordering ambiguity |

The result: on a fresh cluster sync or `selfHeal` re-sync, pods are not guaranteed to start in the sequence required for operational stability.

---

## Ordering Requirements

The required startup contract — derived from service dependencies:

```
Layer 0: Cluster primitives (MetalLB, cert-manager, external-secrets, Traefik, Crossplane)
            ↓  [ArgoCD healthchecks pass]
Layer 1: Namespace / RBAC / secret scaffolding for ALL services
            ↓  [All namespaces and ExternalSecrets rendered]
Layer 2: Temporal server
            ↓  [temporal-server Application Healthy]
Layer 3: Temporal workers
            ↓  [temporal-worker + temporal-worker-dev Applications Healthy]
Layer 4: GitHub Actions runners
            ↓  [actions-runner Application Healthy]
Layer 5: SemaphoreUI
            ↓  [semaphoreui Application Healthy]
Layer 6: All remaining workloads (fourdogs, inference-gateway, grafana, prometheus, influxdb, ollama, terminus-portal)
            ↓  [All layer 6 Applications Healthy]
Layer 7: Ingress Applications
            ↓
Layer 8: ServiceMonitors
```

---

## Solution Overview

All ordering is enforced exclusively via `argocd.argoproj.io/sync-wave` annotations on `Application` objects in `platforms/k3s/argocd/apps/`. No external wait hooks or health hooks are required — the ArgoCD automated sync with `selfHeal` already gates wave advancement on Application health status.

No new infrastructure components, side-cars, or init containers are introduced. This is a pure annotation-layer change confined to the App-of-Apps manifests.

### Why sync waves (not ArgoCD resource hooks or app dependencies)

| Option | Verdict | Reason |
|--------|---------|--------|
| `argocd.argoproj.io/sync-wave` on Application objects | **Chosen** | Native, zero-dependency, declarative, visible in git diff |
| ArgoCD `health.lua` custom checks | Not needed | Built-in Application health check is sufficient |
| ArgoCD `sync-wave` on individual k8s resources inside apps | Partial overlap | Already used inside some apps (e.g. temporal-infra child resources); orthogonal to Application-level sequencing |
| Ansible wait loops post-deploy | Not needed | ArgoCD self-heals deterministically; operator wait loops add fragility |

### ArgoCD Health Gate Behavior

ArgoCD advances to the next wave only after **all Applications in the current wave report `Healthy`** (when the root-app uses automated sync). If any Application in wave N stays `Progressing` or `Degraded`, wave N+1 never starts.

This is the health gate the startup sequence relies on. No additional configuration is required to activate it.

---

## Complete Wave Map

### Wave 0 — Cluster Primitives (no annotation; implicit default)

These run first and have no ordering dependency on anything within the cluster.

| Application | File | Note |
|-------------|------|------|
| MetalLB | `metallb.yaml` | L2 LB — must be healthy before any Service of type LoadBalancer |
| cert-manager | `cert-manager.yaml` | In-cluster TLS lifecycle — must be healthy before any Certificate/ClusterIssuer |
| external-secrets | `external-secrets.yaml` | ESO operator — must be healthy before any ExternalSecret in wave 1+ |
| Traefik | `traefik.yaml` | Ingress controller — must be healthy before any Ingress resource |
| Crossplane | `crossplane.yaml` | Infrastructure provider — must be healthy before any Crossplane Claim |

**No annotation added** — these Applications must remain at the implicit default (wave 0) because they have no ordering dependency on any other Application. Adding an explicit `"0"` annotation would be equivalent and is acceptable but unnecessary.

---

### Wave 1 — Namespace / RBAC / Secret Scaffolding

All `-infra` Applications create namespaces, RBAC, and ExternalSecret objects. They must run **after** the external-secrets operator (wave 0) is Healthy to avoid ExternalSecrets rendering against an unavailable ESO webhook.

**Applications staying at wave 1 (annotation already correct):**
- `grafana-infra.yaml`
- `grafana-dev-infra.yaml`
- `influxdb-infra.yaml`
- `influxdb-dev-infra.yaml`
- `inference-gateway-infra.yaml`
- `inference-gateway-dev-infra.yaml`
- `prometheus-infra.yaml`

**Applications gaining wave 1 annotation (currently no annotation):**

| File | Current state | Change |
|------|--------------|--------|
| `temporal-infra.yaml` | No annotation (wave 0) | Add `argocd.argoproj.io/sync-wave: "1"` |
| `fourdogs-central-infra.yaml` | No annotation (wave 0) | Add `argocd.argoproj.io/sync-wave: "1"` |
| `fourdogs-central-dev-infra.yaml` | No annotation (wave 0) | Add `argocd.argoproj.io/sync-wave: "1"` |
| `fourdogs-kaylee-agent-infra.yaml` | No annotation (wave 0) | Add `argocd.argoproj.io/sync-wave: "1"` |
| `fourdogs-kaylee-agent-dev-infra.yaml` | No annotation (wave 0) | Add `argocd.argoproj.io/sync-wave: "1"` |
| `terminus-portal-infra.yaml` | No annotation (wave 0) | Add `argocd.argoproj.io/sync-wave: "1"` |

> **Important:** `temporal-infra.yaml` deploys ExternalSecrets for Temporal's database credentials. If it races with the ESO operator at wave 0, those secrets may fail to render. Moving to wave 1 eliminates this race.

---

### Wave 2 — Temporal Server

| Application | File | Current | Change |
|-------------|------|---------|--------|
| temporal-server | `temporal-server.yaml` | `"2"` | None — correct ✓ |

Temporal server runs schema migrations via the `admintools` Job on install. It must be Healthy before workers start.

---

### Wave 3 — Temporal Workers

| Application | File | Current | Change |
|-------------|------|---------|--------|
| temporal-worker | `temporal-worker.yaml` | `"3"` | None — correct ✓ |
| temporal-worker-dev | `temporal-worker-dev.yaml` | `"3"` | None — correct ✓ |

Workers register task queues against the Temporal server. They must be Healthy before CI/CD tools that submit Temporal workflows (Actions runners, Semaphore pipelines) can function.

---

### Wave 4 — GitHub Actions Runners

| Application | File | Current | Change |
|-------------|------|---------|--------|
| actions-runner | `actions-runner.yaml` | `"2"` | **Change to `"4"`** |

The Actions runner is the execution substrate for GitHub-triggered CI/CD pipelines, several of which submit Temporal workflows. Deploying it at wave 2 (alongside `temporal-server`) means it may attempt connections before workers are registered.

---

### Wave 5 — SemaphoreUI

| Application | File | Current | Change |
|-------------|------|---------|--------|
| semaphoreui | `semaphoreui.yaml` | None (wave 0) | **Add `argocd.argoproj.io/sync-wave: "5"`** |

SemaphoreUI is the GitOps CI/CD orchestration platform. It orchestrates OpenTofu runs, Ansible playbooks, and verifies deployments. It depends on Temporal (for release pipelines) and GitHub Actions (for triggered workflows). Deploying it at wave 0 while those services have not started is both functionally incorrect and operationally dangerous — Semaphore tasks will fail immediately on startup.

---

### Wave 6 — All Remaining Application Workloads

All non-infra, non-CI/CD application workloads deploy after the full CI/CD platform is healthy.

**Applications moving to wave 6 (currently wave 1 or wave 2):**

| Application | File | Current | Change |
|-------------|------|---------|--------|
| fourdogs-central | `fourdogs-central.yaml` | `"1"` | **Change to `"6"`** |
| fourdogs-central-dev | `fourdogs-central-dev.yaml` | `"1"` | **Change to `"6"`** |
| fourdogs-emailfetcher | `fourdogs-emailfetcher.yaml` | `"1"` | **Change to `"6"`** |
| fourdogs-emailfetcher-dev | `fourdogs-emailfetcher-dev.yaml` | `"1"` | **Change to `"6"`** |
| fourdogs-kaylee-agent | `fourdogs-kaylee-agent.yaml` | `"1"` | **Change to `"6"`** |
| fourdogs-kaylee-agent-dev | `fourdogs-kaylee-agent-dev.yaml` | `"1"` | **Change to `"6"`** |
| grafana | `grafana.yaml` | `"2"` | **Change to `"6"`** |
| grafana-dev | `grafana-dev.yaml` | `"2"` | **Change to `"6"`** |
| inference-gateway | `inference-gateway.yaml` | `"2"` | **Change to `"6"`** |
| inference-gateway-dev | `inference-gateway-dev.yaml` | `"2"` | **Change to `"6"`** |
| influxdb | `influxdb.yaml` | `"2"` | **Change to `"6"`** |
| influxdb-dev | `influxdb-dev.yaml` | `"2"` | **Change to `"6"`** |
| prometheus | `prometheus.yaml` | `"2"` | **Change to `"6"`** |
| prometheus-dev | `prometheus-dev.yaml` | `"2"` | **Change to `"6"`** |
| ollama | `ollama.yaml` | `"2"` | **Change to `"6"`** |
| terminus-portal | `terminus-portal.yaml` | `"2"` | **Change to `"6"`** |
| terminus-portal-dev | `terminus-portal-dev.yaml` | `"2"` | **Change to `"6"`** |

All wave 6 Applications start concurrently — there are no ordering dependencies between monitoring tools, application services, and dev environments within this layer.

---

### Wave 7 — Ingress Applications

All ingress Applications deploy after their upstream app is Healthy. Consolidating ingresses to a single wave eliminates the current 3-way split (waves 3, 4) and creates a clear, auditable pattern: apps at wave 6, ingresses at wave 7.

| Application | File | Current | Change |
|-------------|------|---------|--------|
| grafana-ingress | `grafana-ingress.yaml` | `"3"` | **Change to `"7"`** |
| grafana-dev-ingress | `grafana-dev-ingress.yaml` | `"3"` | **Change to `"7"`** |
| inference-gateway-ingress | `inference-gateway-ingress.yaml` | `"3"` | **Change to `"7"`** |
| inference-gateway-dev-ingress | `inference-gateway-dev-ingress.yaml` | `"3"` | **Change to `"7"`** |
| influxdb-ingress | `influxdb-ingress.yaml` | `"3"` | **Change to `"7"`** |
| influxdb-dev-ingress | `influxdb-dev-ingress.yaml` | `"3"` | **Change to `"7"`** |
| prometheus-ingress | `prometheus-ingress.yaml` | `"3"` | **Change to `"7"`** |
| terminus-portal-dev-ingress | `terminus-portal-dev-ingress.yaml` | `"3"` | **Change to `"7"`** |
| temporal-worker-ingress | `temporal-worker-ingress.yaml` | `"4"` | **Change to `"7"`** |
| temporal-worker-dev-ingress | `temporal-worker-dev-ingress.yaml` | `"4"` | **Change to `"7"`** |

---

### Wave 8 — ServiceMonitors

| Application | File | Current | Change |
|-------------|------|---------|--------|
| monitoring-servicemonitors | `monitoring-servicemonitors.yaml` | `"3"` | **Change to `"8"`** |

ServiceMonitors require both the Prometheus operator (grafana-infra, prometheus at wave 6) and all monitored applications to be deployed. Wave 8 guarantees this.

---

## Change Summary

| Category | Files | Action |
|----------|-------|--------|
| Add wave annotation (critical — out-of-order deploy) | `semaphoreui.yaml` | Add `"5"` |
| Add wave annotation (bug fix — ESO race) | `temporal-infra.yaml`, `fourdogs-central-infra.yaml`, `fourdogs-central-dev-infra.yaml`, `fourdogs-kaylee-agent-infra.yaml`, `fourdogs-kaylee-agent-dev-infra.yaml`, `terminus-portal-infra.yaml` | Add `"1"` |
| Change wave value (sequencing correction) | `actions-runner.yaml` | `"2"` → `"4"` |
| Change wave value (sequencing correction) | `fourdogs-central.yaml`, `fourdogs-central-dev.yaml`, `fourdogs-emailfetcher.yaml`, `fourdogs-emailfetcher-dev.yaml`, `fourdogs-kaylee-agent.yaml`, `fourdogs-kaylee-agent-dev.yaml` | `"1"` → `"6"` |
| Change wave value (unified "everything else") | `grafana.yaml`, `grafana-dev.yaml`, `inference-gateway.yaml`, `inference-gateway-dev.yaml`, `influxdb.yaml`, `influxdb-dev.yaml`, `prometheus.yaml`, `prometheus-dev.yaml`, `ollama.yaml`, `terminus-portal.yaml`, `terminus-portal-dev.yaml` | `"2"` → `"6"` |
| Change wave value (unified ingress layer) | `grafana-ingress.yaml`, `grafana-dev-ingress.yaml`, `inference-gateway-ingress.yaml`, `inference-gateway-dev-ingress.yaml`, `influxdb-ingress.yaml`, `influxdb-dev-ingress.yaml`, `prometheus-ingress.yaml`, `terminus-portal-dev-ingress.yaml` | `"3"` → `"7"` |
| Change wave value (unified ingress layer) | `temporal-worker-ingress.yaml`, `temporal-worker-dev-ingress.yaml` | `"4"` → `"7"` |
| Change wave value (after all apps) | `monitoring-servicemonitors.yaml` | `"3"` → `"8"` |
| No change | `metallb.yaml`, `cert-manager.yaml`, `external-secrets.yaml`, `traefik.yaml`, `crossplane.yaml`, `temporal-server.yaml`, `temporal-worker.yaml`, `temporal-worker-dev.yaml`, all existing wave 1 `-infra` apps | None |

**Total files modified:** 30  
**Total files unchanged:** 12 (+ all existing wave 1 `-infra` apps already correct)

---

## Project Structure (No New Files)

This feature introduces **no new files** and **no new directories**. All changes are annotation edits within existing YAML files in:

```
platforms/k3s/argocd/apps/
├── actions-runner.yaml              wave: 2 → 4
├── cert-manager.yaml                wave: 0 (no change)
├── crossplane.yaml                  wave: 0 (no change)
├── external-secrets.yaml            wave: 0 (no change)
├── fourdogs-central-dev-infra.yaml  wave: 0 → 1 (add annotation)
├── fourdogs-central-dev.yaml        wave: 1 → 6
├── fourdogs-central-infra.yaml      wave: 0 → 1 (add annotation)
├── fourdogs-central.yaml            wave: 1 → 6
├── fourdogs-emailfetcher-dev.yaml   wave: 1 → 6
├── fourdogs-emailfetcher.yaml       wave: 1 → 6
├── fourdogs-kaylee-agent-dev-infra.yaml  wave: 0 → 1 (add annotation)
├── fourdogs-kaylee-agent-dev.yaml   wave: 1 → 6
├── fourdogs-kaylee-agent-infra.yaml wave: 0 → 1 (add annotation)
├── fourdogs-kaylee-agent.yaml       wave: 1 → 6
├── grafana-dev-infra.yaml           wave: 1 (no change)
├── grafana-dev-ingress.yaml         wave: 3 → 7
├── grafana-dev.yaml                 wave: 2 → 6
├── grafana-infra.yaml               wave: 1 (no change)
├── grafana-ingress.yaml             wave: 3 → 7
├── grafana.yaml                     wave: 2 → 6
├── inference-gateway-dev-infra.yaml wave: 1 (no change)
├── inference-gateway-dev-ingress.yaml  wave: 3 → 7
├── inference-gateway-dev.yaml       wave: 2 → 6
├── inference-gateway-infra.yaml     wave: 1 (no change)
├── inference-gateway-ingress.yaml   wave: 3 → 7
├── inference-gateway.yaml           wave: 2 → 6
├── influxdb-dev-infra.yaml          wave: 1 (no change)
├── influxdb-dev-ingress.yaml        wave: 3 → 7
├── influxdb-dev.yaml                wave: 2 → 6
├── influxdb-infra.yaml              wave: 1 (no change)
├── influxdb-ingress.yaml            wave: 3 → 7
├── influxdb.yaml                    wave: 2 → 6
├── metallb.yaml                     wave: 0 (no change)
├── monitoring-servicemonitors.yaml  wave: 3 → 8
├── ollama.yaml                      wave: 2 → 6
├── prometheus-dev.yaml              wave: 2 → 6
├── prometheus-infra.yaml            wave: 1 (no change)
├── prometheus-ingress.yaml          wave: 3 → 7
├── prometheus.yaml                  wave: 2 → 6
├── semaphoreui.yaml                 wave: 0 → 5 (add annotation)
├── temporal-infra.yaml              wave: 0 → 1 (add annotation)
├── temporal-server.yaml             wave: 2 (no change)
├── temporal-worker-dev-ingress.yaml wave: 4 → 7
├── temporal-worker-dev.yaml         wave: 3 (no change)
├── temporal-worker-ingress.yaml     wave: 4 → 7
├── temporal-worker.yaml             wave: 3 (no change)
├── terminus-portal-dev-ingress.yaml wave: 3 → 7
├── terminus-portal-dev.yaml         wave: 2 → 6
├── terminus-portal-infra.yaml       wave: 0 → 1 (add annotation)
├── terminus-portal.yaml             wave: 2 → 6
└── traefik.yaml                     wave: 0 (no change)
```

---

## ADRs

### ADR-001: Sync Waves as Sole Sequencing Mechanism

**Status:** Accepted  
**Decision:** Use `argocd.argoproj.io/sync-wave` annotations on `Application` objects as the sole mechanism for cluster-level pod startup ordering.

**Alternatives considered:**
- Kubernetes pod priority classes — applies within a node scheduler, not across ArgoCD Application lifecycles
- ArgoCD resource hooks (PreSync/PostSync) — add complexity for something sync waves handle natively
- Ansible wait-for loops — fragile, out-of-band, not declarative

**Rationale:** Sync waves are native to ArgoCD, visible in `git diff`, and require no new dependencies. The code change is confined to annotation values in 30 YAML files.

---

### ADR-002: Wave 6 "Everything Else" — Monitoring and App Workloads Share a Wave

**Status:** Accepted  
**Decision:** Grafana, Prometheus, InfluxDB, inference-gateway, ollama, fourdogs apps, and terminus-portal are all assigned wave 6. They start concurrently.

**Rationale:** None of these services have a real ordering dependency among themselves at the ArgoCD Application level. Their internal startup ordering (e.g. Prometheus scraping a target that doesn't exist yet) is handled by pod-level probes and retry logic, not by wave sequencing.

---

### ADR-003: Unified Ingress Layer at Wave 7

**Status:** Accepted  
**Decision:** All ingress Applications are assigned wave 7. Previous wave 3 and wave 4 assignments had no consistent rule and will be replaced.

**Rationale:** An Ingress Application has exactly one dependency: its upstream app must be Healthy (wave 6). A single consistent rule is easier to maintain and audit than per-ingress wave assignments.

---

### ADR-004: No Wave Annotation for Cluster Primitives

**Status:** Accepted  
**Decision:** `metallb`, `cert-manager`, `external-secrets`, `traefik`, and `crossplane` retain no wave annotation and therefore default to wave 0.

**Rationale:** These are the foundational layer — there is nothing before them in the cluster. Adding an explicit `"0"` would be functionally identical and add noise. The implicit default is correct and intentional.

---

## Implementation Patterns

### Annotation placement rule

Wave annotations go on the `Application` object's `metadata.annotations` block, not on child resources. Placement is consistent across all files:

```yaml
metadata:
  name: <app-name>
  namespace: argocd
  labels:
    ...
  annotations:
    argocd.argoproj.io/sync-wave: "<N>"
  finalizers:
    ...
```

If a file currently has no `annotations:` block in `metadata`, one must be added before the `finalizers:` block.

### Comment convention

Each Application YAML that has a wave annotation should include a comment (in the file header block) explaining its wave assignment. Existing files follow this pattern (e.g. `# Sync wave 2 — after namespace/RBAC/ESO secrets (wave 1), before worker (wave 3)`). Follow the same style when adding or changing annotations.

---

## Validation Checklist

Before implementation is complete, verify:

- [ ] All 30 modified files have been updated
- [ ] `fourdogs-central.yaml` header comment updated to reflect wave 6 (not wave 1)
- [ ] `semaphoreui.yaml` has `annotations:` block added before `finalizers:`
- [ ] `temporal-infra.yaml` header comment updated to reflect wave 1 (not "runs first")
- [ ] `fourdogs-*-infra.yaml` header comments updated to reflect explicit wave 1
- [ ] Empty `kubectl get applications -n argocd -o custom-columns=NAME:.metadata.name,WAVE:.metadata.annotations.argocd\.argoproj\.io/sync-wave --sort-by='.metadata.annotations.argocd\.argoproj\.io/sync-wave'` shows expected wave distribution after a `git push` + ArgoCD refresh
- [ ] No Application missing an annotation when its deployment order matters (i.e., everything except the 5 cluster primitives has an explicit wave)
