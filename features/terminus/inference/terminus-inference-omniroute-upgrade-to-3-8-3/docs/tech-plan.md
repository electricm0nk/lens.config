# Tech Plan — OmniRoute Upgrade to 3.8.3

**Feature**: terminus-inference-omniroute-upgrade-to-3-8-3  
**Track**: quickdev  
**Domain/Service**: terminus / inference  
**Date**: 2026-05-26

---

## Current State

| Item | Current | Target |
|---|---|---|
| Image | `diegosouzapw/omniroute:3.2.8` | `diegosouzapw/omniroute:3.8.3` |
| K8s version label (metadata + pod template) | `"3.2.8"` | `"3.8.3"` |
| Service ports exposed | `20128` only | `20128` (+ assess `20129`) |
| Deployed namespace | `inference` | `inference` |
| Data PVC | `omniroute-data-pvc` at `/app/data` | unchanged |

---

## Files to Change

### Primary change — `terminus.infra`

**File**: `platforms/k3s/k8s/omniroute/deployment.yaml`

Changes:
1. `image: diegosouzapw/omniroute:3.2.8` → `image: diegosouzapw/omniroute:3.8.3`
2. `app.kubernetes.io/version: "3.2.8"` in `metadata.labels` → `"3.8.3"`
3. `app.kubernetes.io/version: "3.2.8"` in `spec.template.metadata.labels` → `"3.8.3"`
4. Update header comment: `# Image: diegosouzapw/omniroute:3.8.3`
5. Update initiative annotation comment if referencing the old feature slug

### Required change — `service.yaml`

**File**: `platforms/k3s/k8s/omniroute/service.yaml`

v3.8.3 introduces a WebSocket monitoring sidecar on port `20129`. This port:
- Is not required for API routing (all `/v1/` traffic still on 20128)
- Enables real-time dashboard log streaming in the OmniRoute UI
- Is not currently exposed externally (no ingress needed)

**Decision**: Add port `20129` to the ClusterIP service to allow in-cluster dashboard access. Required — enables WebSocket monitoring dashboard access for the new 3.8.3 observability feature.

---

## Upgrade Path and DB Migration Notes

OmniRoute uses SQLite with an incremental migration system. Upgrading from 3.2.8 to 3.8.3 spans roughly 30+ schema migrations. Key behaviors:

- **Auto-run on startup**: Migrations execute automatically when the process starts
- **PVC preserves data**: The SQLite database at `/app/data/omniroute.db` is on a persistent volume — data survives pod restart and image upgrade
- **No manual migration steps**: OmniRoute's migration system is append-only; no manual intervention required
- **Startup time**: Expect 30–60s startup time on first boot with the migration batch. **Bump `readinessProbe.initialDelaySeconds` from 30 to 60 for the upgrade deploy** — revert after validation confirms startup time is acceptable

Notable migration paths to verify:
- `007` migration uses `IF NOT EXISTS` idempotent logic (fixed in 3.7.4)
- `032` migration tracking reconciled (fixed in 3.7.4) 
- `quota_snapshots` table added (3.7.5)
- Legacy encryption fallback resolved (3.8.2)

**Probe adjustment**: Consider bumping `readinessProbe.initialDelaySeconds` from `30` to `60` for the first deploy, then revert. Alternatively, the `failureThreshold: 3` at `periodSeconds: 10` provides 30s of grace after initial delay.

### `better-sqlite3` Compatibility

In v3.8.1, `better-sqlite3` was moved to `optionalDependencies`. The Docker image cascade is:
```
better-sqlite3 (native) → node:sqlite → sql.js (WASM)
```

The `diegosouzapw/omniroute` Docker image bundles native binaries and should continue to use `better-sqlite3`. No action required. If startup logs show SQLite adapter fallback warnings, assess the Docker image's native binary state.

---

## Validation Plan

### Step 1: Deployment health
```bash
kubectl rollout status deployment/omniroute -n inference
kubectl get pods -n inference -l app.kubernetes.io/name=omniroute
```

### Step 2: Health endpoint
```bash
kubectl exec -n inference deployment/omniroute -- \
  curl -s http://localhost:20128/api/monitoring/health
```
Expect: `{"status":"ok"}` or equivalent 200 response.

### Step 3: Combo validation
Verify each combo responds via the vault-auth-proxy → OmniRoute `/v1/chat/completions` chain:
- combo-chat
- combo-code
- combo-interactive
- combo-triage
- combo-batch (llamacpp-only)

Test via the OmniRoute dashboard or a direct `curl` with the internal API key from the `omniroute-secrets` ExternalSecret.

### Step 4: ArgoCD sync
- Check `omniroute` ArgoCD app shows `Synced` + `Healthy`
- Check `omniroute-ingress` ArgoCD app remains `Healthy`

### Step 5: Version confirmation
```bash
kubectl get deployment omniroute -n inference \
  -o jsonpath='{.metadata.labels.app\.kubernetes\.io/version}'
```
Expect: `3.8.3`

---

## Rollback Plan

1. Revert `deployment.yaml` image tag to `diegosouzapw/omniroute:3.2.8`
2. Revert version labels
3. Commit and push to the infra repo feature branch
4. ArgoCD auto-syncs or manual sync triggers pod rollback
5. SQLite data on PVC is forward-compatible; no DB rollback needed

**RTO**: ~2 minutes (image pull is fast; SQLite PVC is unchanged).

---

## Architecture Notes

No architecture changes are required. The upgrade is a self-contained image bump within the existing topology:

```
Traefik Ingress
    → ForwardAuth middleware → vault-auth-proxy:8080/validate
    → OmniRoute:20128 (ClusterIP)
        → SQLite PVC at /app/data
        → upstream providers via combo routing
```

The new WebSocket sidecar (port 20129) runs within the same container and does not alter this flow.
