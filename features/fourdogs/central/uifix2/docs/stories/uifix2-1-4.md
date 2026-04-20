---
story_id: uifix2-1-4
feature: uifix2
title: Fix Load Recommendations button qualification
status: review
points: 2
repo: fourdogs-central-ui
target_branch: develop
files_to_change:
  - src/pages/OrderDetail.tsx
  - src/hooks/use_vendor_catalog.ts
  - src/lib/types.ts (or wherever ChairSku is defined — to verify riskScore field)
depends_on: []
updated_at: "2026-04-28"
---

# Story uifix2-1-4 — Fix Load Recommendations button qualification

## Context

The "Load Recommendations" button on the ordering worksheet calls
`applyKayleeRecommendations()` which builds a list of items to apply quantities to.
The filter is:
```typescript
.filter((sku) => sku.suggestedQty && sku.suggestedQty > 0)
```

`suggestedQty` maps from `items.o` in the backend, which is computed by `cmd/ingest`
based on sales velocity. In the dev environment, ingest runs without sales data →
velocity = 0 → `items.o = 0` → `suggestedQty = undefined` for all items.

After velocitysignalwiring2, the system has `risk_score` (0-100) and `dos_days` as
richer signals. The `risk_score` is populated by a separate risk-refresh process
even when sales history is absent. The button should use `risk_score` as a fallback
qualification signal.

## Root Cause

```typescript
// OrderDetail.tsx — applyKayleeRecommendations (approximate)
const applyKayleeRecommendations = () => {
  const sorted = [...sourceSkus]
    .filter((sku) => sku.suggestedQty && sku.suggestedQty > 0)  // ← too narrow
    .sort(...)
    ...
  sorted.forEach(sku => {
    const qty = sku.suggestedQty!;  // ← undefined if risk_score path used
    ...
  });
};
```

## Pre-Implementation Verification (REQUIRED)

Before writing code, developer must verify:

1. **`riskScore` in `ChairSku`:** Check `src/hooks/use_vendor_catalog.ts` `normalizeItem`
   function. Confirm whether `riskScore` is already mapped from `row.risk_score`.
   - If YES: proceed with fix as designed.
   - If NO: add the mapping first:
     ```typescript
     riskScore: typeof row.risk_score === 'number' ? row.risk_score : undefined,
     ```
     Also add `riskScore?: number` to the `ChairSku` type definition.

2. **Dev DB has risk_score data:** Run a quick check that at least some items in the
   dev database have `risk_score > 0`. If all are NULL, the button still won't work
   after this fix, and a separate data task is needed (run risk-refresh endpoint).
   - Check: `GET /internal/velocity-risk/refresh` (POST) or check the DB directly.

## Acceptance Criteria

1. The "Load Recommendations" button applies quantities to items where `suggestedQty > 0`
   OR `riskScore > 0` (either signal qualifies the item).
2. When `suggestedQty` is available, it is used as the applied quantity.
3. When `suggestedQty` is NOT available (undefined or 0) but `riskScore > 0`, the
   applied quantity is `sku.caseSize || 1` (one case, or 1 unit if case size unknown).
4. `riskScore` is present in `ChairSku` type and mapped in `normalizeItem`.
5. The button works in dev environment — at least one item receives a quantity after pressing.
6. Existing behaviour for items with `suggestedQty > 0` is unchanged.

## Implementation Guide

### Step 1 — Verify/add riskScore to ChairSku

File: `src/hooks/use_vendor_catalog.ts` (and ChairSku type location)

```typescript
// In normalizeItem, verify or add:
riskScore: typeof row.risk_score === 'number' ? row.risk_score : undefined,
dosDays:   typeof row.dos_days === 'number'   ? row.dos_days   : undefined,
```

### Step 2 — Update applyKayleeRecommendations

File: `src/pages/OrderDetail.tsx`

```typescript
// BEFORE
const sorted = [...sourceSkus]
  .filter((sku) => sku.suggestedQty && sku.suggestedQty > 0)
  ...

// AFTER
const sorted = [...sourceSkus]
  .filter((sku) =>
    (sku.suggestedQty !== undefined && sku.suggestedQty > 0) ||
    (sku.riskScore !== undefined && sku.riskScore > 0)
  )
  ...

// When applying qty:
// BEFORE: const qty = sku.suggestedQty!;
// AFTER:
const qty = (sku.suggestedQty && sku.suggestedQty > 0)
  ? sku.suggestedQty
  : (sku.caseSize || 1);
```

## Notes for Developer

- If `risk_score` is missing for all items in dev (NULL in DB), trigger the risk-refresh
  endpoint: `curl -X POST http://localhost:8080/internal/velocity-risk/refresh` after
  running `cmd/ingest`. This is a data issue, not a code issue, but must be resolved
  for AC-5 to pass.
- `vc` (velocity class) is missing from `GetItemsByVendor` SQL SELECT. This affects
  sort order in `applyKayleeRecommendations` (items sorted by velocity class). This is
  a known gap deferred to a follow-on story — do not fix it in this story.

## Definition of Done

- [ ] `ChairSku` type has `riskScore?: number` field
- [ ] `normalizeItem` maps `row.risk_score` → `riskScore`
- [ ] `applyKayleeRecommendations` qualifies items by `suggestedQty > 0 || riskScore > 0`
- [ ] Applied qty = `suggestedQty ?? (caseSize || 1)`
- [ ] At least one item receives a qty when "Load Recommendations" is pressed in dev
- [ ] Existing `suggestedQty`-based behaviour is unchanged when `suggestedQty > 0`
- [ ] No TypeScript compilation errors
