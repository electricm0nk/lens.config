---
feature: portal3
doc_type: finalizeplan-review
status: draft
goal: "Confirm portal3 is ready to move from planning into execution with an implementation bundle centered on reliability, layout consolidation, and CI/CD visibility."
key_decisions:
  - "Treat the latest user-requested portal fixes as authoritative scope refinements for the first execution slice."
  - "Keep GitHub Actions visibility browser-safe for now by using public workflow-run data only."
  - "Defer deeper CI/CD telemetry and any authenticated GitHub runner status until a proxy or backend feed exists."
open_questions:
  - "Are the monitored GitHub repos all public, or do we need a proxy-backed Actions feed next?"
  - "Should Fourdogs remain enabled in the catalog if its health endpoint is not consistently available?"
  - "Do we want follow-on work to expose deployment state from ArgoCD alongside GitHub Actions?"
depends_on: []
blocks: []
updated_at: 2026-04-19T02:08:00Z
---

# Adversarial Review: portal3 / finalizeplan

**Reviewed:** 2026-04-19T02:08:00Z
**Source:** manual-rerun
**Overall Rating:** pass-with-warnings

## Summary

`portal3` is ready to proceed through FinalizePlan as a stabilization-focused portal upgrade. The scope is now much more concrete than the original brief: fix the live service catalog, reorganize the dashboard around domain/service sections, reduce wasted screen space, and add an initial CI/CD visibility panel centered on GitHub Actions. The plan is viable and execution-ready, but a few important constraints remain: GitHub Actions visibility is only safely implementable from the browser for public workflow data, browser health checks remain coarse because of `no-cors`, and the Proxmox card intentionally accepts a certificate mismatch tradeoff by targeting the IP address directly.

## Findings

### Critical

None.

### High

| # | Dimension | Finding | Recommendation |
|---|-----------|---------|----------------|
| H1 | Complexity and Risk | The original GitHub Actions HUD strawman embeds a PAT in client-side code and calls authenticated GitHub endpoints directly from the browser. That would leak credentials immediately in a static frontend. | Treat the current browser-safe public-actions view as the only acceptable first increment. Any private-repo or runner-level visibility must go through a proxy, backend service, or signed feed. |

### Medium / Low

| # | Severity | Dimension | Finding | Recommendation |
|---|----------|-----------|---------|----------------|
| M1 | Medium | Coverage Gaps | The latest user-specified fixes materially refine scope beyond the original ExpressPlan docs, especially around service inventory cleanup and layout consolidation. | Fold those refinements into the execution bundle and treat them as the source of truth for the first implementation slice. |
| M2 | Medium | Assumptions and Blind Spots | Proxmox is now intentionally routed to `https://10.0.0.48:8006`, which bypasses the FQDN problem but may still present certificate trust friction in the browser. | Accept the UX tradeoff explicitly and document it as an operational constraint rather than a portal bug. |
| M3 | Medium | Cross-Feature Dependencies | GitHub Actions visibility is only one layer of CI/CD state. Operators may quickly expect deployment, ArgoCD sync, and release status in the same surface. | Keep portal3’s first CI/CD slice narrow: GitHub Actions only. Capture ArgoCD/deployment telemetry as a follow-on initiative or a later portal3 story if still desired. |
| L1 | Low | Logic Flaws | The current dashboard grouping by broad category alone was wasting screen real estate and making navigation less intentional. | Proceed with the domain/service grouping model and denser card layout as the new baseline information architecture. |

## Accepted Risks

- Browser-side health polling still cannot provide deep status semantics for many cross-origin endpoints. `ONLINE` / `UNREACHABLE` remains an acceptable first-order signal.
- The initial GitHub Actions panel may show incomplete data for private repositories without additional server-side support.
- The portal may still need one more iteration after rollout to unify CI/CD visibility with deployment visibility.

## Party-Mode Challenge

Winston (Architect): You improved the information architecture, but are you sure the portal won’t become a second operations console with expanding telemetry requirements? Draw a hard line around what portal3 will and will not visualize.

John (PM): The user asked for “at least add GitHub Actions.” That implies a first step, not the whole destination. Make sure the release value is obvious even if the CI/CD panel is intentionally shallow.

Bob (Delivery Lead): The highest execution risk is not code complexity, it is scope creep. If GitHub Actions, layout overhaul, service cleanup, and follow-on deployment telemetry all land in one batch, review and rollout confidence will drop.

## Gaps You May Not Have Considered

1. Whether the GitHub repos you want surfaced are public enough for unauthenticated browser access.
2. Whether certificate warnings on the Proxmox IP link will confuse less frequent operators.
3. Whether Fourdogs and other marginal services should be shown as enabled, pending, or hidden when their availability is inconsistent.
4. Whether the domain/service layout should become the long-term taxonomy for all future portal cards and metrics panels.

## Open Questions Surfaced

- Should the next CI/CD visibility increment include deployment state from ArgoCD or remain GitHub-only for now?
- Do we want an explicit “unsafe cert / direct IP” badge or hint on the Proxmox card?
- Should Actions monitoring expand beyond `terminus-portal`, `terminus.infra`, and `lens.config` in the first release?
