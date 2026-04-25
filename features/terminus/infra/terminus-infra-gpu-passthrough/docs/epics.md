---
feature: terminus-infra-gpu-passthrough
doc_type: epics
status: draft
goal: "Break GPU passthrough k3s architecture into delivery epics"
key_decisions:
  - 3 epics: host preparation, k3s worker provisioning, gateway integration
  - Epic 1 is a hard prerequisite for Epic 2
  - Epic 2 includes F-1 IOMMU evidence artifact requirement
  - Epic 3 includes F-2 NetworkPolicy story
depends_on:
  - architecture.md
  - finalizeplan-review.md
updated_at: "2026-04-25T00:00:00Z"
---

# Epic List — terminus-infra-gpu-passthrough

Feature: terminus-infra-gpu-passthrough  
Track: tech-change  
Domain: terminus/infra

## Epic List

### Epic 1 — Proxmox Host VFIO Preparation

**Goal:** Verify and configure the Proxmox host to expose the RTX 3090 Turbo via VFIO passthrough, and commit the captured evidence as a pre-provisioning gate artifact.

**Scope:**
- Identify target Proxmox node and confirm PCI address for RTX 3090 Turbo
- Enable IOMMU in BIOS (VT-d) and set kernel parameters (intel_iommu=on iommu=pt)
- Load VFIO modules at boot (vfio, vfio_pci, vfio_iommu_type1)
- Bind RTX 3090 vendor:device to vfio-pci driver
- Verify IOMMU group isolation (group must not include critical host devices)
- Commit captured IOMMU group output as a repository artifact (F-1 gate)

**Dependencies:** terminus-infra-proxmox (host access), no k8s dependencies

**Done when:** IOMMU group output is committed; RTX 3090 is bound to vfio-pci on the target host; a verification test confirms passthrough readiness.

---

### Epic 2 — GPU Worker k3s Node Provisioning

**Goal:** Provision a dedicated k3s GPU worker VM in Proxmox with GPU passthrough, NVIDIA runtime, and correct node policy, using the GPU worker OpenTofu module.

**Scope:**
- Author `gpu-worker` OpenTofu module (or extend k3s extension module) with required inputs: target_node, vm_id, gpu_pci_id, mac_address, static_ipv4, kube_node_name
- Provision gpu-worker-01 VM with pinned MAC address
- Bootstrap NVIDIA driver + container toolkit + k3s runtime (pick one canonical method — F-OQ-BOOTSTRAP)
- Enable NVIDIA k8s device plugin cluster-wide
- Verify nvidia.com/gpu allocatable on node
- Apply taint (workload.accelerator/gpu=true:NoSchedule) and label
- Deploy Ollama with GPU resource claim and node selector/toleration
- Validate GPU pod scheduling (test pod requesting nvidia.com/gpu=1 runs successfully)
- Define MAC address source of truth (tfvars in-repo, F-T-5)
- Document worker VM sizing with daemonset overhead reserve (F-3: min 500m CPU / 512Mi RAM above Ollama budget)

**Dependencies:** Epic 1 must be complete (IOMMU gate passed, vfio-pci bound)

**Done when:** GPU worker node shows allocatable nvidia.com/gpu; Ollama pod runs and schedules on the GPU worker; nvidia-smi returns correct card info inside pod.

---

### Epic 3 — Inference Gateway Integration and Security Hardening

**Goal:** Configure the inference gateway to route eligible workloads to the local Ollama provider, validate fallback behavior, and enforce network isolation.

**Scope:**
- Add local Ollama provider entry to gateway provider routing config (Ollama service DNS, workload class assignment)
- Assign local-first routing for target workload classes
- Verify gateway can reach Ollama ClusterIP service
- Validate healthy-path routing: gateway correctly sends local-first workloads to Ollama
- Validate fallback-path routing: force Ollama provider down and confirm fallback to remote provider (F-T-3 acceptance gate)
- Apply NetworkPolicy restricting Ollama service access to gateway namespace/pods only (F-2)
- Define GPU telemetry minimum: nvidia.com/gpu allocation, pod GPU utilization, gateway degraded-state counters

**Dependencies:** Epic 2 must be complete (Ollama running on GPU worker)

**Done when:** Gateway routes local workloads to Ollama; forced-failure fallback test passes; NetworkPolicy applied and verified; minimum GPU metrics visible in observability stack.

---

## Dependency Map

```
Epic 1 (Proxmox VFIO prep)
  └── Epic 2 (k3s GPU worker provisioning)
        └── Epic 3 (Gateway integration + security hardening)
```

## Open Questions Gating Epics

| OQ | Epics Gated | Question |
|----|-------------|----------|
| OQ-1 | Epic 1, 2 | Exact Proxmox node + PCI address for RTX 3090 Turbo |
| OQ-2 | Epic 2 | Final static IP + FQDN for gpu-worker-01 |
| OQ-3 | Epic 3 | Default concurrency targets + model memory budget for 3090 Ollama |
| OQ-MAC | Epic 2 | MAC address source of truth: tfvars in-repo vs Vault |
| OQ-BOOTSTRAP | Epic 2 | Canonical bootstrap method for NVIDIA driver + runtime on gpu-worker-01 |

These must be resolved before Epic 1 and 2 implementation begins, not before sprint planning.
