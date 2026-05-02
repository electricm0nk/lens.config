# Story portal4-4-03: Implement /api/metrics/query sidecar route (Prometheus proxy + CA trust)

**Epic:** 4 — Deep Health Signals and Prometheus Metrics
**Status:** backlog
**Initiative:** portal4
**Date:** 2026-05-02
**Estimate:** M

---

## User Story

As a **developer wiring metrics into the portal**,
I want a sidecar route that proxies Prometheus query requests using internal CA TLS trust,
So that the portal SPA can display live Prometheus metrics without direct browser access to `prometheus.trantor.internal`.

---

## Context

**Repo:** `terminus-portal` (sidecar Go code)
**Branch:** `portal4`
**Depends on:** portal4-2-03 merged (Helm chart with sidecar must exist for ConfigMap mount)

**Architecture (ADR-4 in tech-plan):**
- Route: `GET /api/metrics/query?query=<promql>&start=<rfc3339>&end=<rfc3339>&step=<duration>`
- Sidecar proxies to `https://prometheus.trantor.internal` using a custom HTTP client that trusts the internal CA
- Internal CA cert: mounted as a Kubernetes ConfigMap volume into the sidecar container; NOT `InsecureSkipVerify: true`
- Prometheus URL: `PROMETHEUS_URL` env var (default: `https://prometheus.trantor.internal`)

**Prometheus confirmed deployed** (feature `prometheus-wiring` at `finalizeplan-complete`).

---

## Acceptance Criteria

**Given** the sidecar is running with the internal CA cert mounted
**When** `GET /api/metrics/query?query=up` is called
**Then** the sidecar connects to Prometheus over TLS without certificate errors and returns the Prometheus JSON response

**Given** the sidecar uses `InsecureSkipVerify: false` (default, strict TLS)
**When** the CA cert ConfigMap is correctly mounted
**Then** the TLS handshake succeeds (confirms CA trust is working, not bypassed)

**Given** the CA cert ConfigMap is missing or incorrectly mounted
**When** the sidecar starts
**Then** it logs a startup warning that Prometheus CA cert is not found (not a fatal crash — it should still start)

**Given** the PromQL query is invalid
**When** Prometheus returns a 400
**Then** the sidecar forwards the 400 and error body to the SPA

**Given** `go vet ./...` and `go build ./...` are run
**Then** both pass

---

## Dev Notes

- Add `sidecar/handlers/metrics.go`
- Custom TLS client setup:
  ```go
  caCert, err := os.ReadFile("/etc/ssl/certs/internal-ca.crt")
  if err != nil {
      slog.Warn("Prometheus CA cert not found; TLS will fail for internal endpoints", "path", "/etc/ssl/certs/internal-ca.crt")
  }
  caCertPool := x509.NewCertPool()
  caCertPool.AppendCertsFromPEM(caCert)
  tlsConfig := &tls.Config{RootCAs: caCertPool}
  prometheusClient := &http.Client{Transport: &http.Transport{TLSClientConfig: tlsConfig}, Timeout: 15 * time.Second}
  ```
- Add to Helm chart: a ConfigMap containing the internal CA cert + a volumeMount for the sidecar at `/etc/ssl/certs/internal-ca.crt`
  - The internal CA cert content should be sourced from the cluster's existing CA (check ESO ClusterSecretStore or existing cert-manager CA for the trantor.internal domain)
  - If the CA cert is already in a k8s Secret or ConfigMap, reference it; otherwise add it as a new Helm values entry
- Route: `GET /api/metrics/query` — forward `query`, `start`, `end`, `step` query params to Prometheus `/api/v1/query_range`
- `PROMETHEUS_URL` env var: add to Helm chart deployment with value `https://prometheus.trantor.internal`
- Never log Prometheus response body (may contain sensitive metric labels)

---

## Definition of Done

- [ ] Code merged to feature branch (`portal4`)
- [ ] `go vet ./...` and `go build ./...` passing
- [ ] `GET /api/metrics/query?query=up` returns valid Prometheus JSON from dev cluster
- [ ] TLS uses internal CA cert (not InsecureSkipVerify) — confirmed by checking `tlsConfig.InsecureSkipVerify` is `false`
- [ ] CA cert mounted as ConfigMap, not hardcoded in image
- [ ] Missing CA cert logs a warning (not fatal)
- [ ] No Prometheus response body logged
- [ ] ArgoCD dev app syncs Healthy after deploy
- [ ] No secrets committed to source code
