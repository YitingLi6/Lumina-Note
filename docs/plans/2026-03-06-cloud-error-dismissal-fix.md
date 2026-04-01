# Cloud Error Dismissal Fix Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Allow the cloud sync settings banner to dismiss cloud auth errors instead of leaving a sticky error state.

**Architecture:** Add an explicit cloud-store error reset action and have the shared banner clear both cloud and WebDAV errors. Keep the UI structure unchanged and verify behavior through a focused settings regression test.

**Tech Stack:** React, Zustand, Vitest, Testing Library

---

### Task 1: Add regression coverage for dismissing cloud errors

**Files:**
- Modify: `src/components/settings/WebDAVSettings.test.tsx`

**Step 1: Write the failing test**
- Render `WebDAVSettings` with both `useCloudSyncStore.error` and `useWebDAVStore.connectionError` populated.
- Click the existing alert close button.
- Assert the cloud error is cleared and the banner disappears.

**Step 2: Run test to verify it fails**
Run: `npm run test:run -- src/components/settings/WebDAVSettings.test.tsx`
Expected: FAIL because the close button only clears the WebDAV error today.

### Task 2: Implement the minimal fix

**Files:**
- Modify: `src/stores/useCloudSyncStore.ts`
- Modify: `src/components/settings/WebDAVSettings.tsx`

**Step 1: Add cloud error reset action**
- Extend `CloudSyncState` with `clearError: () => void`.
- Implement it as `set({ error: null })`.

**Step 2: Wire the banner dismiss handler**
- Read `clearError` from `useCloudSyncStore` under a distinct local name.
- Replace the inline close handler with a small function that clears both cloud and WebDAV errors.

**Step 3: Keep scope minimal**
- Do not restructure the banner.
- Do not change other auth or sync flows.

### Task 3: Verify the fix and append it to the PR branch

**Files:**
- Modify: `docs/plans/2026-03-06-cloud-error-dismissal-fix.md`

**Step 1: Run focused verification**
Run: `npm run test:run -- src/components/settings/WebDAVSettings.test.tsx`
Expected: PASS.

**Step 2: Run cloud-focused verification**
Run: `npm run test:cloud`
Expected: PASS if the environment supports the existing suite.

**Step 3: Commit and push**
- Commit only the plan and code/test changes.
- Push `pr-157` so the open PR picks up the fix.
