---
story_id: uifix2-1-3
feature: uifix2
title: Gate tier labels on Kaylee recommendation presence
status: review
points: 2
repo: fourdogs-central-ui
target_branch: develop
files_to_change:
  - src/pages/OrderDetail.tsx
depends_on: []
updated_at: "2026-04-28"
---

# Story uifix2-1-3 — Gate tier labels on Kaylee recommendation presence

## Context

On the ordering worksheet (`OrderDetail.tsx`), every row in the order grid shows a
coloured tier badge: **INCREASED**, **KAYLEE**, **DECREASED**, or **GUESS**. These
labels are intended to compare the operator's entered qty against Kaylee's recommendation.

The bug: the tier label fires even when no Kaylee recommendations have been loaded.
When the operator opens the worksheet and types quantities before pressing
"Load Recommendations", every row with a non-zero qty shows INCREASED, KAYLEE, or
DECREASED based purely on the magnitude of the quantity — not on any comparison with
Kaylee.

This is caused by `resolveSkuTier` falling through to `getQtyConfidenceTier(qty)` when
`kayleeQty === undefined`.

## Root Cause

```typescript
// OrderDetail.tsx — resolveSkuTier (approximate)
export function resolveSkuTier(
  qty: number,
  kayleeQty: number | undefined,
  isImported: boolean
): number {
  if (kayleeQty !== undefined) {
    if (qty > kayleeQty) return 1;    // INCREASED
    if (qty === kayleeQty) return 2;  // KAYLEE
    if (qty > 0) return 3;            // DECREASED
    return 4;                         // GUESS
  }
  // BUG: these fire when no Kaylee rec is loaded
  if (!isImported && qty > 0) return 1;    // INCREASED for any non-floor-walk item
  return getQtyConfidenceTier(qty);        // INCREASED/KAYLEE/DECREASED by qty magnitude
}
```

## Acceptance Criteria

1. When the ordering worksheet is opened and no Kaylee recommendations have been loaded
   (i.e., "Load Recommendations" has not been pressed in this session), all rows show
   **no tier label** (tier 4 / GUESS / empty badge) regardless of their qty.
2. After pressing "Load Recommendations" successfully, rows with a qty above Kaylee's
   recommendation show **INCREASED**, at-recommendation show **KAYLEE**, and below show
   **DECREASED**.
3. The `resolveSkuTier` function returns `4` for any input where `kayleeQty === undefined`,
   regardless of `qty` or `isImported` values.
4. Unit tests for `resolveSkuTier` pass for all cases including the `kayleeQty === undefined`
   case.

## Implementation Guide

**File:** `src/pages/OrderDetail.tsx`

Find `resolveSkuTier` (it may be a local function or imported from `orderGrid.ts`):

```typescript
// BEFORE
function resolveSkuTier(qty: number, kayleeQty: number | undefined, isImported: boolean): number {
  if (kayleeQty !== undefined) {
    if (qty > kayleeQty) return 1;
    if (qty === kayleeQty) return 2;
    if (qty > 0) return 3;
    return 4;
  }
  if (!isImported && qty > 0) return 1;
  return getQtyConfidenceTier(qty);
}

// AFTER
function resolveSkuTier(qty: number, kayleeQty: number | undefined, isImported: boolean): number {
  if (kayleeQty !== undefined) {
    if (qty > kayleeQty) return 1;
    if (qty === kayleeQty) return 2;
    if (qty > 0) return 3;
    return 4;
  }
  return 4;  // No Kaylee recommendation loaded → no tier label
}
```

Note: `getQtyConfidenceTier` is NOT removed — it may be used elsewhere. Only the
fallback call from `resolveSkuTier` is removed.

## Tests to Write

```typescript
// resolveSkuTier
it('returns tier 4 when kayleeQty is undefined, any qty', () => {
  expect(resolveSkuTier(0, undefined, false)).toBe(4);
  expect(resolveSkuTier(4, undefined, false)).toBe(4);
  expect(resolveSkuTier(100, undefined, false)).toBe(4);
  expect(resolveSkuTier(4, undefined, true)).toBe(4);
});

it('returns INCREASED (1) when qty > kayleeQty', () => {
  expect(resolveSkuTier(5, 3, false)).toBe(1);
});

it('returns KAYLEE (2) when qty === kayleeQty', () => {
  expect(resolveSkuTier(3, 3, false)).toBe(2);
});

it('returns DECREASED (3) when qty < kayleeQty and qty > 0', () => {
  expect(resolveSkuTier(1, 4, false)).toBe(3);
});
```

## Definition of Done

- [ ] `resolveSkuTier` returns `4` for all inputs when `kayleeQty === undefined`
- [ ] No tier labels visible on fresh worksheet open (before Load Recommendations)
- [ ] Tier labels appear correctly after Load Recommendations is pressed
- [ ] Unit tests written and passing
- [ ] No regression in Kaylee recommendation display for items with a `kayleeQty` set
