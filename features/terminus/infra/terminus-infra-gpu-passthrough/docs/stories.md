---
feature: terminus-infra-gpu-passthrough
doc_type: stories
status: draft
goal: "User stories for GPU passthrough implementation"
depends_on:
  - epics.md
  - architecture.md
updated_at: "2026-04-25T00:00:00Z"
---

# Story List — terminus-infra-gpu-passthrough

Feature: terminus-infra-gpu-passthrough  
Track: tech-change  
Domain: terminus/infra

---

## Epic 1 — Proxmox Host VFIO Preparation

### Story 1.1 — Identify GPU Host and Capture IOMMU Group Output

**As a** platform operator  
**I want to** identify the Proxmox node hosting the RTX 3090 Turbo, capture its PCI address and IOMMU group membership, and commit that output as a repository artifact  
**So that** the IOMMU isolation gate (ADR-2 / F-1) is satisfied before any provisioning begins

**Acceptance Criteria:**
- [ ] Target Proxmox node identified and recorded in `docs/terminus/infra/terminus-infra-gpu-passthrough/iommu-evidence.txt`
- [ ] PCI address for RTX 3090 Turbo confirmed and recorded
- [ ] Full IOMMU group listing for the GPU PCI slot captured and committed
- [ ] IOMMU group does NOT include critical host devices (NIC, storage controller); if it does, this story is blocked and must be escalated before Epic 2 begins
- [ ] Committed file is non-empty and passes review

**Story Points:** 2

---

### Story 1.2 — Configure Proxmox Host VFIO at Boot

**As a** platform operator  
**I want to** configure IOMMU kernel parameters and VFIO module loading on the target Proxmox host  
**So that** the RTX 3090 Turbo is reliably bound to vfio-pci at every host boot

**Acceptance Criteria:**
- [ ] BIOS VT-d / AMD-Vi confirmed enabled on target host
- [ ] Kernel cmdline includes `intel_iommu=on iommu=pt` (Intel) or `amd_iommu=on iommu=pt` (AMD) — confirmed in `/proc/cmdline`
- [ ] VFIO modules loaded: `vfio`, `vfio_pci`, `vfio_iommu_type1` — confirmed via `lsmod`
- [ ] RTX 3090 vendor:device bound to `vfio-pci` — confirmed via `lspci -k`
- [ ] Host survives one full reboot with GPU still bound to `vfio-pci` (not to NVIDIA driver)
- [ ] Story 1.1 must be complete before this story runs

**Story Points:** 3

---

## Epic 2 — GPU Worker k3s Node Provisioning

### Story 2.1 — Resolve Open Questions Gating Provisioning

**As a** platform engineer  
**I want to** resolve all OQ blocks for the GPU worker before any OpenTofu work begins  
**So that** provisioning variables are concrete and the MAC source of truth is declared

**Acceptance Criteria:**
- [ ] OQ-1: Proxmox node name + PCI ID recorded in tfvars or equivalent config file
- [ ] OQ-2: gpu-worker-01 static IP address and FQDN assigned and DNS record created
- [ ] OQ-MAC: Source of truth for MAC address declared (tfvars in-repo OR Vault secret path); recorded in module README
- [ ] OQ-BOOTSTRAP: Single bootstrap method selected (cloud-init vs Ansible vs image bake); version pins recorded for NVIDIA driver, container toolkit, k3s

**Story Points:** 1

---

### Story 2.2 — Author GPU Worker OpenTofu Module

**As a** infrastructure engineer  
**I want to** create a `gpu-worker` OpenTofu module (or extend the k3s extension module) with all required inputs and GPU passthrough configuration  
**So that** the GPU worker VM can be provisioned reproducibly with MAC pinning and GPU passthrough

**Acceptance Criteria:**
- [ ] Module exists at `tofu/modules/gpu-worker/` in terminus.infra repo
- [ ] Required inputs declared: `target_node`, `vm_id`, `gpu_pci_id`, `mac_address`, `static_ipv4`, `kube_node_name`
- [ ] `hostpci` block includes GPU PCI ID with `pcie=true` and `rombar=true`
- [ ] `network_device` block includes required `mac_address` (mac pinning enforced)
- [ ] Module README documents minimum worker VM sizing: base + NVIDIA device plugin + DCGM + k3s daemonset overhead (~500m CPU, 512Mi RAM reserved above Ollama budget) per F-3
- [ ] Story 2.1 must be complete before this story runs

**Story Points:** 5

---

### Story 2.3 — Provision gpu-worker-01 and Bootstrap NVIDIA Runtime

**As a** infrastructure engineer  
**I want to** apply the GPU worker OpenTofu module to provision gpu-worker-01 and bootstrap the NVIDIA software stack using the canonical method decided in Story 2.1  
**So that** the k3s GPU worker is live and NVIDIA device plugin advertises nvidia.com/gpu in the cluster

**Acceptance Criteria:**
- [ ] `tofu apply` completes without error for gpu-worker-01 environment
- [ ] VM boots with correct static IP and FQDN
- [ ] MAC address matches the pinned value in tfvars / Vault
- [ ] NVIDIA driver, container toolkit, and k3s runtime are installed and running
- [ ] NVIDIA k8s device plugin deployed cluster-wide
- [ ] `kubectl describe node gpu-worker-01` shows `nvidia.com/gpu: 1` under Allocatable
- [ ] Test pod requesting `nvidia.com/gpu: 1` runs successfully on gpu-worker-01
- [ ] `nvidia-smi` returns RTX 3090 Turbo details inside the test pod
- [ ] Stories 2.1 and 2.2 must be complete before this story runs

**Story Points:** 8

---

### Story 2.4 — Apply GPU Node Policy and Deploy Ollama

**As a** platform engineer  
**I want to** apply the correct k8s taint/label to gpu-worker-01 and deploy Ollama with GPU resource claims  
**So that** only inference workloads schedule on the GPU node and Ollama is accessible via ClusterIP

**Acceptance Criteria:**
- [ ] Node labeled: `workload.accelerator/gpu=true`
- [ ] Node tainted: `workload.accelerator/gpu=true:NoSchedule`
- [ ] Ollama deployment has `nodeSelector` matching GPU worker label
- [ ] Ollama deployment has toleration for GPU taint
- [ ] Ollama deployment has `resources.limits nvidia.com/gpu: 1`
- [ ] Ollama ClusterIP service created (no NodePort or LoadBalancer)
- [ ] Ollama pod runs successfully on gpu-worker-01 with GPU claim satisfied
- [ ] General workloads without GPU toleration do NOT schedule on gpu-worker-01 (verified by test)
- [ ] Story 2.3 must be complete before this story runs

**Story Points:** 5

---

## Epic 3 — Inference Gateway Integration and Security Hardening

### Story 3.1 — Register Ollama as Local Provider in Gateway Routing Config

**As a** platform engineer  
**I want to** add the Ollama ClusterIP service as a local provider entry in the inference gateway's provider routing configuration  
**So that** the gateway can route eligible workloads to local GPU capacity

**Acceptance Criteria:**
- [ ] Ollama provider entry added to provider routing YAML config with correct ClusterIP service DNS
- [ ] Workload classes designated as local-first are defined (from OQ-3 resolution)
- [ ] Gateway configuration deployed and running without errors
- [ ] Gateway health check shows Ollama local provider as healthy
- [ ] Story 2.4 must be complete before this story runs

**Story Points:** 3

---

### Story 3.2 — Validate Healthy-Path and Fallback-Path Routing

**As a** platform engineer  
**I want to** validate that the gateway correctly routes to Ollama on the healthy path and falls back correctly when Ollama is unavailable  
**So that** the routing integration is proven end-to-end before production traffic (F-T-3 acceptance gate)

**Acceptance Criteria:**
- [ ] Healthy-path test: gateway routes local-first workload class requests to Ollama and returns a valid response
- [ ] Forced-failure test: Ollama provider marked unavailable (service scaled to 0); gateway falls back to remote provider; no error returned to client
- [ ] Recovery test: Ollama restored (service scaled back); gateway resumes routing to local provider
- [ ] All three test outcomes documented in a test-results file committed to the repo
- [ ] Story 3.1 must be complete before this story runs

**Story Points:** 5

---

### Story 3.3 — Apply NetworkPolicy to Restrict Ollama Provider Access

**As a** security-conscious platform engineer  
**I want to** apply a Kubernetes NetworkPolicy that restricts access to the Ollama service to only the gateway namespace/pods  
**So that** the local provider service is not reachable from arbitrary cluster workloads (F-2 from finalizeplan review)

**Acceptance Criteria:**
- [ ] NetworkPolicy applied in the namespace where Ollama service runs
- [ ] Policy allows ingress only from gateway namespace (or specific pod selector matching gateway pods)
- [ ] Policy denies ingress from all other namespaces/pods
- [ ] Verification: gateway can reach Ollama service; a test pod outside gateway namespace CANNOT reach Ollama service
- [ ] Story 3.1 must be complete before this story runs

**Story Points:** 3

---

### Story 3.4 — Establish Minimum GPU Telemetry

**As a** platform operator  
**I want to** confirm that minimum GPU telemetry metrics are visible in the observability stack before declaring the feature ready  
**So that** GPU allocation, utilization, and provider degradation can be diagnosed without relying on manual SSH access

**Acceptance Criteria:**
- [ ] `nvidia.com/gpu` allocation visible per pod in cluster metrics
- [ ] GPU memory utilization metric visible (via DCGM exporter or nvidia-smi scrape)
- [ ] Gateway degraded-state counter for local provider visible in Prometheus / observability stack
- [ ] At minimum one dashboard panel or alert definition covers GPU-provider degraded state
- [ ] Stories 3.2 and 3.3 must be complete before this story qualifies as launch gate

**Story Points:** 3

---

## Story Summary

| Story | Epic | Points | Key Gate |
|-------|------|--------|----------|
| 1.1 — Identify GPU host + capture IOMMU evidence | 1 | 2 | F-1 artifact gate |
| 1.2 — Configure VFIO at boot | 1 | 3 | Host reboot validation |
| 2.1 — Resolve OQ blocks | 2 | 1 | Unblocks all Epic 2 |
| 2.2 — GPU worker OpenTofu module | 2 | 5 | Module review |
| 2.3 — Provision worker + bootstrap NVIDIA runtime | 2 | 8 | nvidia.com/gpu allocatable |
| 2.4 — Node policy + Ollama deploy | 2 | 5 | GPU pod scheduled |
| 3.1 — Register Ollama provider in gateway | 3 | 3 | Gateway health check |
| 3.2 — Healthy + fallback routing validation | 3 | 5 | F-T-3 acceptance gate |
| 3.3 — NetworkPolicy for Ollama service | 3 | 3 | F-2 gate |
| 3.4 — Minimum GPU telemetry | 3 | 3 | Launch gate |
| **Total** | | **38** | |
