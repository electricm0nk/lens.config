---
feature: terminus-infra-gpu-passthrough
doc_type: finalizeplan-review
phase: finalizeplan
source: phase-complete
verdict: pass-with-warnings
reviewed_artifacts:
  - architecture.md
  - techplan-adversarial-review.md
updated_at: "2026-04-25T00:00:00Z"
---

# FinalizePlan Review — GPU Passthrough to k3s

Feature: terminus-infra-gpu-passthrough  
Phase: finalizeplan  
Track: tech-change  
Verdict: pass-with-warnings

## Artifact Review Summary

### Architecture.md

ADR-1 (k3s worker): Internally consistent throughout. Correctly confirmed aligns with live gateway in k3s.  
ADR-2 (static VFIO): Hard gate is properly declared. Evidence requirement exists but evidence storage mechanism is not specified.  
ADR-3 (GPU worker OpenTofu module + MAC pin): Pattern is consistent with terminus.infra OpenTofu conventions.  
ADR-4 (Ollama first, gateway routing owns policy): Correct alignment with existing deployed gateway/provider-routing.  
ADR-5 (taint/label isolation): Standard k8s pattern, correctly applied.

The architecture.md is coherent, well-scoped, and ready for implementation planning. The standalone-VM pivot was completed cleanly. No residual design contradictions remain.

### TechPlan Adversarial Review

TechPlan review verdict was `pass-with-warnings`. Six warnings were issued (T-1 through T-6). None have been resolved yet — resolution is deferred to the implementation phase and must be captured in stories. The blind-spot questions raised during party-mode challenge have been carried forward and are addressed below.

## TechPlan Blind-Spot Questions — Final Status

| # | Question | Status |
|---|----------|--------|
| 1 | Which exact Proxmox node and PCI ID for the 3090? | Open — must be recorded before first `tofu apply` (F-1) |
| 2 | Canonical bootstrap method for NVIDIA runtime on gpu-worker-01? | Open — must be specified in stories (T-2) |
| 3 | Where is the MAC address source of truth? | Open — architecture says tfvars or Vault; one must be chosen (T-5) |
| 4 | Which workload classes become local-first once Ollama is live? | Open — tracked as routing config update in gateway story |
| 5 | Minimum fallback test that must pass before rollout sign-off? | Open — T-3 requires acceptance criteria in routing integration story |

## Party-Mode Challenge Round

### Infra perspective
Architecture correctly targets k3s, but the GPU worker OpenTofu module is new code. The most common first-run failure for PCI passthrough is IOMMU group overlap. The hard gate (ADR-2) is declared, but the implementation plan does not yet specify where IOMMU evidence must be committed (in-repo) before `tofu apply`. This is a checklist artifact requirement, not just a human memory gate. **Captured as F-1.**

### Security perspective
T-6 (NetworkPolicy for provider service) is a medium-severity warning in the TechPlan review. It risks being silently deferred if no story explicitly claims it. NetworkPolicy restricting Ollama service access to gateway pods/namespace is a required security control per ADR-2 intent. **Captured as F-2.**

### Operations perspective
T-8 (GPU telemetry) is low severity but has material operational impact. If GPU metrics (nvidia.com/gpu allocation, memory, utilization) are not available from day one, debugging inference issues will rely on manual SSH and nvidia-smi. Recommend requiring minimum GPU visibility metrics as a launch acceptance criterion for the first deployment story.

### Scheduling perspective
The taint/label pattern (ADR-5) is correct, but worker VM sizing in the implementation phase must account for system daemonset overhead: NVIDIA device plugin, DCGM exporter, and k3s daemonsets together consume approximately 500m CPU and 512Mi RAM on the GPU worker. Worker VM sizing must reserve this headroom above the Ollama model budget. **Captured as F-3.**

## New Findings

### F-1 — IOMMU evidence must be committed as a repo artifact before provisioning
**Severity: HIGH**

The hard gate declared in ADR-2 is correct, but it is only a human process gate unless evidence is committed to the repository. The implementation phase must include a story to capture and commit IOMMU group output for the target Proxmox node and PCI slot before any `tofu apply` against gpu-worker resources.

Recommendation:
- Add story: "Capture and commit IOMMU group output for RTX 3090 host slot"
- Gate `tofu apply` on presence of this committed artifact (checklist item)

### F-2 — NetworkPolicy story must be explicit in sprint plan
**Severity: MEDIUM**

T-6 from TechPlan review flagged that the security boundary relies on convention (Ollama service is internal-only) rather than an enforced policy. Without an explicit story, this is likely to be deferred indefinitely. The NetworkPolicy restricting provider service access to gateway pods must be a first-sprint story, not a backlog item.

Recommendation:
- Add story: "Apply NetworkPolicy to restrict Ollama provider service to gateway namespace"
- Mark as sprint-1 acceptance gate alongside routing integration

### F-3 — GPU worker VM sizing must reserve system daemonset overhead
**Severity: LOW**

NVIDIA device plugin, DCGM exporter, and k3s daemonsets together consume approximately 500m CPU and 512Mi RAM on the GPU worker node. Worker VM sizing in the OpenTofu module inputs must reserve this headroom above the Ollama model memory budget. Failure to do so will cause inference pods to be unschedulable when GPU memory is at the configured limit.

Recommendation:
- Document minimum worker VM sizing in gpu-worker module README
- Add sizing validation to implementation readiness checklist

## Governance Impact Assessment

| Related Feature | Phase | Impact | Action Required |
|-----------------|-------|--------|-----------------|
| terminus-infra-proxmox | preplan | **Dependency** — GPU passthrough requires VFIO host capability. If proxmox feature changes passthrough strategy, this feature must be revised. | Flag sequence dependency; align when proxmox exits preplan. |
| terminus-infra-k3s | preplan | **Dependency** — GPU worker provisioning depends on k3s worker module patterns. If k3s feature changes provisioning conventions, GPU worker module must follow. | Track in epics/stories; review when k3s exits preplan. |
| terminus-inference-gateway | live (deployed) | **Integration** — Gateway receives new local provider. No planning-phase risk; routing config update is bounded change. | Implementation story: update provider routing config. |
| terminus-inference-provider-routing | live (deployed) | **Integration** — Provider routing library handles local provider selection and fallback. Stable API. | Implementation story: add Ollama local provider entry. |

## Open Questions Carried Forward

These must be resolved before implementation begins (not before finalizeplan):

1. **OQ-1**: Confirm exact Proxmox node and PCI address for the RTX 3090 Turbo.  
2. **OQ-2**: Confirm final static IP and FQDN for gpu-worker-01.  
3. **OQ-3**: Confirm default concurrency targets and model memory budget for 3090-backed Ollama deployment.  
4. **OQ-MAC**: Confirm whether MAC address source of truth is tfvars (in-repo) or Vault (external record).  
5. **OQ-BOOTSTRAP**: Confirm canonical bootstrap method for NVIDIA driver + container toolkit + k3s runtime compatibility on gpu-worker-01.

## Verdict

**pass-with-warnings**

The architecture is sound, the pivot to k3s GPU worker is coherent, and the planning set is ready for the implementation bundle. Three new findings (F-1, F-2, F-3) supplement the existing TechPlan warnings. None invalidate the architecture. All must be represented as stories or checklist items in the finalizeplan bundle.

Readiness to proceed: ✅ Proceed with epics, stories, implementation-readiness, and sprint planning.
