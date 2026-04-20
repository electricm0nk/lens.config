---
story_id: uifix2-1-1
feature: uifix2
title: Fix brand tab filtering in catalogTabs
status: review
points: 2
repo: fourdogs-central-ui
target_branch: develop
files_to_change:
  - src/lib/catalogTabs.ts
depends_on: []
updated_at: "2026-04-28"
---

# Story uifix2-1-1 — Fix brand tab filtering in catalogTabs

## Context

`buildCatalogTabs(skus)` in `src/lib/catalogTabs.ts` always includes these tabs
regardless of whether any SKU from the vendor's catalog classifies into them:
- `ruffwear`
- `tractor-supply`
- `reedy-fork`

These are defined in `STATIC_REST_TABS` (and/or `ALL_TAB_IDS`). When ordering from a
vendor like SE Pet that doesn't carry Ruffwear, Tractor Supply, or Reedy Fork items,
those tabs appear with 0 items, confusing operators.

## Root Cause

`buildCatalogTabs` uses a hardcoded `STATIC_REST_TABS` inclusion list rather than
deriving the tab list from the actual SKUs in the vendor's catalog.
`classifyCatalogTab(sku)` already knows how to classify each SKU — the output is just
not used to gate inclusion.

## Acceptance Criteria

1. When `buildCatalogTabs(skus)` is called with a set of SKUs that contains no items
   classified as `ruffwear`, the returned tab list does NOT include a `ruffwear` tab.
2. When `buildCatalogTabs(skus)` is called with at least one Ruffwear SKU, the returned
   tab list DOES include the `ruffwear` tab.
3. Tab ordering is preserved (tabs appear in the same relative order as `STATIC_REST_TABS`
   defines for those that are present; novel tabs appended after).
4. When `skus` is empty (catalog loading), the function returns either an empty list OR
   a minimal default (e.g., `['everything-else']`) — not the full 8-tab list. The
   consuming component must handle an empty tab list without crashing.
5. Existing unit tests for `buildCatalogTabs` pass, and new tests cover criteria 1 and 2.

## Implementation Guide

**File:** `src/lib/catalogTabs.ts`

Current logic (approximate):
```typescript
export function buildCatalogTabs(skus: ChairSku[]): CatalogTab[] {
  const tabSet = new Set<string>([...STATIC_REST_TABS]);  // always includes all
  skus.forEach(s => tabSet.add(classifyCatalogTab(s)));
  return [...tabSet].map(tabId => ({ id: tabId, label: TAB_LABELS[tabId] ?? tabId }));
}
```

Target logic:
```typescript
export function buildCatalogTabs(skus: ChairSku[]): CatalogTab[] {
  if (skus.length === 0) return [];   // caller renders loading state
  const occupied = new Set(skus.map(s => classifyCatalogTab(s)));
  // Preserve STATIC_REST_TABS order for tabs that are present
  const ordered: string[] = [];
  for (const t of STATIC_REST_TABS) {
    if (occupied.has(t)) ordered.push(t);
  }
  // Append any tab classified by skus but not in STATIC_REST_TABS
  for (const t of occupied) {
    if (!ordered.includes(t)) ordered.push(t);
  }
  return ordered.map(tabId => ({ id: tabId, label: TAB_LABELS[tabId] ?? tabId }));
}
```

Verify callers handle `[]` return gracefully (show loading spinner or 'no items').

## Tests to Write

```typescript
// catalogTabs.test.ts
it('excludes ruffwear tab when no ruffwear SKUs present', () => {
  const skus = [/* SE Pet food/treat skus with no brand='ruffwear' */];
  const tabs = buildCatalogTabs(skus);
  expect(tabs.map(t => t.id)).not.toContain('ruffwear');
});

it('includes ruffwear tab when ruffwear SKUs are present', () => {
  const skus = [{ brand: 'ruffwear', ... }];
  const tabs = buildCatalogTabs(skus);
  expect(tabs.map(t => t.id)).toContain('ruffwear');
});

it('returns empty array for empty sku list', () => {
  expect(buildCatalogTabs([])).toEqual([]);
});
```

## Definition of Done

- [ ] `buildCatalogTabs` only returns tabs with ≥1 classified SKU
- [ ] Returns `[]` (or a minimal default) for empty `skus`
- [ ] Unit tests written and passing
- [ ] No console errors in dev environment with SE Pet vendor selected
- [ ] No ruffwear/tractor-supply/reedy-fork tabs visible when ordering from SE Pet
