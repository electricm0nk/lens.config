---
feature: terminus-inference-qwen-adapter
story_id: S4
title: Readiness Probe
epic: E2
sprint: 2
points: 5
status: not-started
assignee: null
depends_on: [S3]
blocks: [S9]
updated_at: '2026-04-26T14:55:00Z'
---

# S4 — Readiness Probe

## As a
Platform engineer

## I want
A K8s readiness probe that checks both the warmup ready marker and the `/health/ready` endpoint

## So that
The pod is never marked Ready and never receives traffic until the Qwen model is fully loaded in VRAM

## Context

K8s readiness probes determine when a pod is eligible to receive traffic. Without a probe tied to warmup completion, k8s might route requests to the pod while it's still loading the model, causing 10-minute response times for unlucky users.

The probe must accommodate the full warmup window (up to 5 minutes) before giving up.

**Note:** S6 (Health Endpoints) must be complete before this probe can work end-to-end, but the probe configuration can be written in parallel with S6.

## Acceptance Criteria

1. K8s deployment manifest includes a `readinessProbe` on the agent container
2. Probe type: `exec` with command: `test -f /tmp/warmup.ready && curl -sf http://localhost:8000/health/ready`
3. Probe timing: `initialDelaySeconds: 60`, `periodSeconds: 10`, `failureThreshold: 30` — allows up to 360 seconds total for warmup to complete
4. `successThreshold: 1` (default — once ready, it's ready)
5. Pod is NOT marked Ready until the probe succeeds
6. Traffic is only routed to the pod after readiness probe succeeds
7. Probe failure does not kill the pod (that's liveness) — only delays readiness

## Technical Notes

```yaml
readinessProbe:
  exec:
    command:
      - /bin/sh
      - -c
      - "test -f /tmp/warmup.ready && curl -sf http://localhost:8000/health/ready"
  initialDelaySeconds: 60
  periodSeconds: 10
  failureThreshold: 30
  successThreshold: 1
```

**Why exec instead of httpGet?** The probe must check both the file marker AND the HTTP endpoint. A compound check requires exec.

**Why 360s total window?** Qwen 3.6 warmup target is <300s; 360s provides 60s buffer. If warmup consistently exceeds this, the timeout should be raised — not lowered.

## Definition of Done

- [ ] K8s deployment manifest updated with readiness probe
- [ ] Probe timing configured as specified (60/10/30)
- [ ] Verified in dev: pod stays NotReady during warmup
- [ ] Verified in dev: pod transitions to Ready after `/tmp/warmup.ready` is written and `/health/ready` returns 200
- [ ] Code reviewed and merged to feature branch
