---
feature: terminus-inference-qwen-adapter
story_id: S7
title: Model Cache Volume
epic: E2
sprint: 2
points: 3
status: not-started
assignee: null
depends_on: [S3]
blocks: [S9]
updated_at: '2026-04-26T14:55:00Z'
---

# S7 — Model Cache Volume

## As a
Platform engineer

## I want
A PersistentVolumeClaim mounted at `/models` in the Qwen adapter pod

## So that
The downloaded model survives pod restarts and warm restarts don't require re-downloading 14GB+ of model weights

## Context

When a pod restarts (crash, rollout, node drain), the container filesystem is wiped. Without a persistent model cache, the warmup script must re-download the full model (~14GB for Qwen 3.6 Q4) on every restart, adding 10+ minutes before warmup can begin.

A PVC at `/models` (matching `OLLAMA_MODELS=/models` from S2) ensures the model is downloaded once and persists across restarts.

**Pre-requisite action:** Confirm the StorageClass name available in the target cluster before starting this story.

## Acceptance Criteria

1. A `PersistentVolumeClaim` named `qwen-adapter-models` is defined in the k8s manifests:
   - `accessModes: [ReadWriteOnce]`
   - `resources.requests.storage: 25Gi`
   - `storageClassName: <confirmed-class>` (confirm with ops team)
2. The Qwen adapter pod/deployment mounts this PVC at `/models` in the container
3. The mount path matches `OLLAMA_MODELS=/models` (set in S2)
4. A warm restart test: delete the pod, wait for it to restart, verify warmup completes faster the second time (model already cached — no re-download, only model load into VRAM)
5. Warm restart warmup duration is <60s (model is local, just needs to be loaded into VRAM by ollama)

## Technical Notes

```yaml
# PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: qwen-adapter-models
  namespace: <target-namespace>
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 25Gi
  storageClassName: <confirm-with-ops>
```

```yaml
# In Deployment spec.containers
volumeMounts:
  - name: models-cache
    mountPath: /models

# In Deployment spec.volumes
volumes:
  - name: models-cache
    persistentVolumeClaim:
      claimName: qwen-adapter-models
```

**Storage sizing:** Qwen 3.6 Q4 is ~8GB; Q8 ~15GB; BF16 ~26GB. 25Gi covers Q4 and Q8 with headroom. If using BF16, increase to 30Gi.

## Definition of Done

- [ ] PVC manifest created in k8s configuration
- [ ] Deployment manifest updated with volumeMount and volume
- [ ] StorageClass confirmed with ops team
- [ ] Warm restart test passes (<60s warmup on second start)
- [ ] Code reviewed and merged to feature branch
