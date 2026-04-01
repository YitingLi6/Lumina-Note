# Remove Blue App Background Glow Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Remove the blue `--primary` glow from the shared app background while keeping the neutral app surface intact.

**Architecture:** Keep the existing `ui-app-bg` hook-up points in React unchanged and narrow the change to the shared CSS definition. Add a targeted regression test that inspects `globals.css` so future tweaks do not reintroduce blue glow layers by accident.

**Tech Stack:** React, Tailwind CSS, Vitest, Node `fs`

---

### Task 1: Lock the background contract

**Files:**
- Test: `src/styles/globals.test.ts`

**Step 1: Write the failing test**
- Read `src/styles/globals.css`
- Assert `.ui-app-bg` and `.dark .ui-app-bg` do not reference `--primary`

**Step 2: Run test to verify it fails**
- Run: `npm run test:run -- src/styles/globals.test.ts`

### Task 2: Remove blue glow layers

**Files:**
- Modify: `src/styles/globals.css`

**Step 1: Remove `--primary` radial gradients from both app background blocks**
- Keep the neutral base background intact

**Step 2: Re-run the test**
- Run: `npm run test:run -- src/styles/globals.test.ts`

### Task 3: Verify and commit

**Files:**
- Modify: `src/styles/globals.css`
- Test: `src/styles/globals.test.ts`

**Step 1: Run focused verification**
- Run: `npm run test:run -- src/styles/globals.test.ts && npm run build`

**Step 2: Commit**
- Run: `git add src/styles/globals.css src/styles/globals.test.ts docs/plans/2026-03-09-remove-blue-app-background-design.md && git commit -m "style: remove blue app background glow"`
