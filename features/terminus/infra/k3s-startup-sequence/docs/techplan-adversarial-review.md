---
feature: k3s-startup-sequence
phase: techplan
source: phase-complete
verdict: pass-with-warnings
reviewed_artifacts:
  - architecture.md
reviewed_by: bmad-lens-adversarial-review
reviewed_at: "2026-04-17"
---

# TechPlan Adversarial Review — k3s-startup-sequence

**Phase:** techplan  
**Source:** phase-complete  
**Verdict:** pass-with-warnings  

---

## Findings

### Critical (0)

None.

---

### High (2)

**H1 — 30-file PR regression surface with no rollback playbook**

Changing sync-wave annotations across 30 Application manifests in a single commit triggers ArgoCD to re-evaluate all Application ordering simultaneously on the next sync. A single annotation typo (wrong value, misplaced `annotations:` block, incorrect indentation) will silently misorder deployments with no automatic recovery. The architecture defines no rollback procedure.

*Accepted as known risk.* Recommended mitigation: document "revert commit + `argocd app sync terminus-infra-k3s-root --force`" in the implementation story acceptance criteria.

**H2 — No post-merge smoke test procedure specified**

The validation checklist is operator-manual (`kubectl get applications` snapshot). There is no post-merge verification procedure or Semaphore verification task specified to confirm all wave assignments are correct after merge.

*Accepted as known risk.* Tracked as implementation story acceptance criteria.

---

### Medium (4)

**M1 — Wave 0 "no annotation = cluster primitive" rule has no enforcement mechanism**

The architecture uses the absence of an annotation to identify cluster primitives. Any new Application that ships without a wave annotation will silently run at wave 0. No CI check or linter enforces the invariant.

**M2 — ESO webhook timing window at wave 1 not mitigated**

The external-secrets operator Application is healthy when its Pod is running, but the ESO webhook may not be ready to serve ExternalSecret requests within the first seconds of wave 1. The six newly-annotated `-infra` Applications do not explicitly specify `syncPolicy.retry`, meaning a failed ExternalSecret render at wave 1 startup may not auto-recover.

**M3 — ArgoCD root-app health is a prerequisite not documented in the runbook**

If `terminus-infra-k3s-root` fails to sync, the entire wave model has no effect. The architecture does not reference a bootstrap gate or health pre-check for the root-app before relying on wave ordering.

**M4 — ArgoCD wave ordering does not apply to out-of-band manual syncs**

Wave sequencing is enforced on full root-app syncs. If any Application is manually synced out of band (e.g. via `argocd app sync <name>`), wave ordering is bypassed for that Application. This is an inherent ArgoCD behavior that the architecture does not acknowledge.

---

### Low (5)

**L1 — Ollama Temporal dependency clarified: none**

Confirmed by implementer: `ollama` has no Temporal dependency. Wave 6 placement is correct as a catch-all and involves no ordering risk.

**L2 — Fourdogs kaylee-agent Temporal dependency not documented**

The architecture moves fourdogs apps to wave 6 (after Temporal workers at wave 3). The reason for this ordering — that kaylee-agent submits Temporal workflows — is not stated in the architecture's dependency section.

**L3 — SemaphoreUI ExternalSecrets are inline in wave 5 Application**

SemaphoreUI deploys `externalsecret.yaml` inline (no separate `-infra` Application). Since `semaphoreui.yaml` is now wave 5 (well after the ESO operator at wave 0), the ExternalSecret will render against a live ESO webhook. The timing risk is acknowledged as low; observed behavior on cluster sync will confirm. *Implementer accepted: "we'll find out."*

**L4 — `prometheus-wiring` feature may deploy additional Application objects**

The `prometheus-wiring` feature is in the same service. If it deploys Application objects not inventoried in this architecture, their wave assignments are unaudited.

**L5 — Wave compliance enforcement is manual/deferred**

No CI automation (e.g. `yq` check that every non-primitive Application has an explicit wave annotation) is specified. Tracked as future operational debt.

---

## Blind-Spot Challenge Responses

| Question | Response |
|---|---|
| Rollback procedure | Not documented — accepted as implementation story acceptance criterion |
| Retry policy on `-infra` apps | Not addressed — tracked as M2 open item |
| Wave compliance CI enforcement | Not addressed — tracked as L5 deferred debt |
| SemaphoreUI `infra` app / ExternalSecret race | ExternalSecrets are inline; wave 5 placement eliminates ESO race. Accepted. |
| Ollama Temporal dependency | **Confirmed none.** Wave 6 placement is correct catch-all. |

---

## Verdict

**pass-with-warnings**

The architecture is sound, coherent, and sufficient to proceed to FinalizePlan. The two high findings (rollback procedure, post-merge smoke test) are accepted implementation risks that should flow into story acceptance criteria rather than blocking the TechPlan phase. No critical findings exist.
