# Retrospective: UI Fix 2

**Feature:** uifix2
**Completed:** 2026-04-20

## What Went Well

- **Express track worked well**: Four tightly scoped defect fixes with no planning ceremony needed. Feature moved from init → dev-ready → implemented in a single session.
- **Test-first discipline**: Every story had tests written alongside the fix — resolve-sku-tier (8 tests), floor-walk priority badge (1 integration test), catalog-tabs (7 unit tests), applyKayleeRecommendations (1 integration test). All 162 pass.
- **Clean extractions**: `resolveSkuTier` was extracted from a private component function to an exported util in `orderGrid.ts`, making it properly testable without any downstream breakage.
- **Pre-verification pattern**: Confirmed `riskScore` was already mapped in `normalizeItem` before writing story uifix2-1-4, avoiding unnecessary churn.
- **Regression coverage**: Two order-lifecycle tests that verified the old buggy behavior were correctly updated to assert the new correct behavior rather than deleted.

## What Didn't Go Well

- **Phase not advanced during dev**: Feature.yaml remained at `dev-ready` throughout implementation. The `/dev` orchestrator was not used, so no automated phase transition occurred. Required manual cleanup at close.
- **Multiple "Bark Biscuits" elements**: A subtle test issue where the item appeared in both the cart summary panel and the catalog table required `getAllByText` + `find(el => el.closest('tr'))` pattern — not obvious from the component code alone.
- **Branch hygiene**: Two stale `temp/promote-*` branches were left diverged and needed investigation + cleanup at end of session.

## Key Learnings

- When using express track without the `/dev` orchestrator, manually update `feature.yaml` phase to `dev-complete` before running `/complete`.
- `sku_id` (snake_case) comes from the API; `skuId` (camelCase) is local state — always verify field names against the actual API response shape before writing integration tests.
- Items that appear in multiple DOM sections (cart summary + catalog table) need `getAllByText` with a structural filter, not `getByText`.
- `getQtyConfidenceTier` and `resolveSkuTier` are now separate functions — `getQtyConfidenceTier` is a raw signal, `resolveSkuTier` gates on Kaylee presence first.

## Metrics

- Planned duration: 1 session
- Actual duration: 1 session
- Stories completed: 4/4 (uifix2-1-1, uifix2-1-2, uifix2-1-3, uifix2-1-4)
- Tests added: 17 new tests; 3 existing tests updated; final count 162 pass
- Bugs found post-merge: 0

## Action Items

- None — all stories implemented and verified. No deferred scope.
