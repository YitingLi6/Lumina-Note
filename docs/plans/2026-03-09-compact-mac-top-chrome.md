# Compact Mac Top Chrome Implementation Plan

> **Execution note:** Use `executing-plans` to implement this plan task-by-task, or another equivalent execution workflow supported by the current agent runtime.

**Goal:** Reduce the macOS custom top chrome to a compact single-line bar that feels closer to Codex.

**Architecture:** Keep the existing mac-only `MacTopChrome` component, but collapse it to one row, remove subtitle rendering, and shrink height/padding. Cover the compact behavior with a targeted component test first, then make the minimal implementation change.

**Tech Stack:** React, Vitest, Testing Library, Tailwind utility classes

---

### Task 1: Compact `MacTopChrome`

**Files:**
- Modify: `/Users/zzzz/Lumina-Note/src/components/layout/MacTopChrome.tsx`
- Test: `/Users/zzzz/Lumina-Note/src/components/layout/MacTopChrome.test.tsx`

**Step 1: Write the failing test**
- Add a test that renders `MacTopChrome` on macOS with a subtitle and asserts the component stays single-line: subtitle is not rendered and the header has compact height classes.

**Step 2: Run test to verify it fails**
- Run: `npm run test:run -- src/components/layout/MacTopChrome.test.tsx`
- Expected: FAIL because the subtitle still renders and the header still uses the old height class.

**Step 3: Write minimal implementation**
- Change `MacTopChrome` to use a shorter height and tighter spacing.
- Stop rendering the subtitle so the bar remains single-line.

**Step 4: Run test to verify it passes**
- Run: `npm run test:run -- src/components/layout/MacTopChrome.test.tsx`
- Expected: PASS

**Step 5: Verify adjacent behavior**
- Run: `npm run test:run -- src/components/layout/MacTopChrome.test.tsx src/components/layout/TitleBar.test.tsx`
- Expected: PASS

**Step 6: Commit**
- `git add /Users/zzzz/Lumina-Note/src/components/layout/MacTopChrome.tsx /Users/zzzz/Lumina-Note/src/components/layout/MacTopChrome.test.tsx /Users/zzzz/Lumina-Note/docs/plans/2026-03-09-compact-mac-top-chrome.md`
- `git commit -m "style(macos): compact top chrome"`
