# Image Manager Compact Redesign

## Problem

ImageManagerView header consumes ~400px before any content is visible. Detail panel permanently occupies 360px width. List view 7-column grid overflows on non-fullscreen. Non-fullscreen usage is essentially broken.

## Design Decisions

- **Summary Cards** → Compact inline stats bar: `128 images · 24 orphans · 3.2 GB`
- **Detail Panel** → Always present but collapsible. New `detailPanelOpen` persisted state.
- **Layout approach** → Two-row compact header (~90px total)

## Header (2 rows, ~90px)

Row 1: Title (text-sm) + inline stats + refresh/view-mode buttons
Row 2: Search (flex-1) + status select + folder select + sort select + sort-order button
Row 3 (conditional): Group mode toggle, only when viewMode === "group"

Removed: Badge label, h1, description paragraph, 6 SummaryCards, gradient background.
Success messages → fixed bottom-right toast.
Orphan warning → inline banner inside content area.

## Detail Panel (collapsible)

- New store field: `detailPanelOpen: boolean` (default true, persisted)
- Expanded: w-[320px] with collapse button at top
- Collapsed: width 0, show ~32px expand button strip
- Remove auto-select of filteredImages[0] as primaryAsset
- `< xl`: collapsed = hidden; expanded = bottom block (existing behavior)
- Transition: `transition-[width,opacity] duration-200 ease-out`

## List View (responsive columns)

- Wide (≥1024px): all 7 columns
- Medium (≥640px): hide "Changed" column
- Narrow (<640px): hide "Changed" + "Location" columns
- Replace hardcoded px widths with `auto` + `minmax`

## Grid Card (hover actions)

- Action buttons: `opacity-0 group-hover:opacity-100 transition-opacity`
- Simplify card info: name + refs badge + size only
- Remove folder line from card (available via folder filter + detail panel)

## Files Changed

1. `src/components/images/ImageManagerView.tsx` — main rework
2. `src/stores/useImageManagerStore.ts` — add `detailPanelOpen`
3. `src/i18n/locales/{en,zh-CN,zh-TW,ja}.ts` — new keys for collapse/expand
4. `src/components/images/ImageManagerView.test.tsx` — adapt tests
