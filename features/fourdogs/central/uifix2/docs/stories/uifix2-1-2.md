---
story_id: uifix2-1-2
feature: uifix2
title: Fix floor walk priority badge source
status: review
points: 2
repo: fourdogs-central-ui
target_branch: develop
files_to_change:
  - src/pages/FloorWalk.tsx
depends_on: []
updated_at: "2026-04-28"
---

# Story uifix2-1-2 — Fix floor walk priority badge source

## Context

On the floor walk screen (`FloorWalk.tsx`), certain catalog rows are highlighted with
an amber "priority" badge to indicate items that came from a floor walk scan. This badge
is driven by `prioritySkuIds`.

Currently `prioritySkuIds` is derived from `lineItems` (local React state), which means
**any item the operator manually types a qty for immediately gets the priority badge**,
even if that item never appeared in a floor walk scan. This conflates "items I'm
evaluating" with "items flagged by the floor walk."

The parallel pattern in `OrderDetail.tsx` — `importedSkuIds` — is correctly scoped to
`floorWalkLinesQuery.data` (server-saved floor walk lines).

## Root Cause

```typescript
// FloorWalk.tsx — CURRENT (bug)
const prioritySkuIds = useMemo(
  () => new Set(lineItems.filter(line => line.quantity > 0).map(line => line.skuId)),
  [lineItems]
);
```

`lineItems` is local state updated on every qty change. Should instead use server data.

## Acceptance Criteria

1. The priority badge (amber highlight) on the floor walk screen only appears for items
   that are present in `floorWalkLinesQuery.data` with `quantity > 0` (items saved to
   the server from a previous or current session's floor walk).
2. Typing a qty directly into a catalog row on the floor walk screen does NOT immediately
   light the priority badge for that row.
3. After a floor walk is saved (auto-save or manual save), previously scanned items DO
   show the priority badge on next page load or query refetch.
4. Existing floor walk functionality (scanning, qty input, save) is unaffected.

## Implementation Guide

**File:** `src/pages/FloorWalk.tsx`

Find the `prioritySkuIds` useMemo and replace:

```typescript
// BEFORE
const prioritySkuIds = useMemo(
  () => new Set(lineItems.filter(line => line.quantity > 0).map(line => line.skuId)),
  [lineItems]
);

// AFTER
const prioritySkuIds = useMemo(
  () => new Set(
    floorWalkLinesQuery.data?.filter(l => l.quantity > 0).map(l => l.sku_id) ?? []
  ),
  [floorWalkLinesQuery.data]
);
```

Note: `floorWalkLinesQuery.data` items use `sku_id` (snake_case from API), while
`lineItems` use `skuId` (camelCase). Use the correct field name for the query data type.

Verify `floorWalkLinesQuery` is the React Query for `GET /v1/orders/${id}/floor-walk-lines`
(it should be — confirmed in context analysis).

## Notes for Developer

- The priority badge flickering during auto-save (scanned item loses badge for ~1s until
  query refetches) is an accepted trade-off. If this is jarring, consider returning the
  union of `floorWalkLinesQuery.data` sku_ids and a local `justScannedIds` set.
- Do not change the `lineItems` state logic — only the source for `prioritySkuIds`.

## Definition of Done

- [ ] `prioritySkuIds` derives from `floorWalkLinesQuery.data`, not `lineItems`
- [ ] Manually typing a qty does not light the priority badge
- [ ] After floor walk is saved, reloading the page shows the priority badge on scanned items
- [ ] No regression in scan → save → badge flow
