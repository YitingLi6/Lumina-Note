# Preview Tab (预览标签页) Design

## 1. Expected Behavior — What Users Expect

### 1.1 Core Concept

Preview tabs solve the "tab explosion" problem: when users browse files in the sidebar, each click currently creates a permanent tab, quickly cluttering the tab bar with files the user only glanced at.

The industry-standard solution (VS Code, Obsidian, JetBrains IDEs) is a **preview tab** — a temporary, single-instance tab that gets reused when the user clicks another file, and only becomes permanent ("pinned" in VS Code terminology, though distinct from our existing pin feature) when the user signals intent to keep it.

### 1.2 Interaction Rules

The design principle is **interaction-driven, not type-driven**: whether a tab is preview or permanent depends on *how* the user opened it, not *what* type of file it is.

#### What creates a preview tab

| Trigger | Rationale |
|---------|-----------|
| Single-click a file in the sidebar file tree | Browsing / glancing |
| Click a backlink in the right panel | Exploratory navigation |
| Click a related file in the right panel | Exploratory navigation |
| Click a node in the knowledge graph / local graph | Exploratory navigation |
| Click a file in the image manager "Referenced by" list | Exploratory navigation |

#### What creates a permanent tab

| Trigger | Rationale |
|---------|-----------|
| Double-click a file in the sidebar file tree | Explicit "I want this open" |
| Open a file from the command palette (Ctrl+O) | User actively searched for it |
| Open a file from global search results | User actively searched for it |
| Create a new file (quick note, new file dialog) | Just created it, will edit |
| AI agent opens/creates a file | Programmatic, needs to stay |
| Ctrl+Click / Cmd+Click an internal link in the editor | Deliberate action |
| Open from favorites | Favorites = intentional |
| Singleton tabs (AI chat, graph, flashcard, etc.) | Not file-browsing |
| Ribbon button tabs (card flow, image manager) | Not file-browsing |
| Database / webpage / video-note tabs | Opened via specific actions, not browsing |

#### What promotes a preview tab to permanent

| Action | Rationale |
|--------|-----------|
| User edits the content (first keystroke) | Started working |
| User double-clicks the preview tab header | Explicit "keep this" |
| User pins the tab | Stronger form of "keep this" |
| File is saved manually (Cmd+S) | Intentional save = working on it |

#### Preview tab behavior

- **Only one preview tab exists at a time.** Opening a new preview replaces the existing one.
- **Preview tab appears at the end of the tab bar** (after all permanent tabs but before any future preview), matching VS Code behavior.
- **Visual distinction:** the tab name is rendered in *italic* to distinguish it from permanent tabs. This is the established convention (VS Code, JetBrains).
- **Closing a preview tab** works the same as closing a permanent tab — no special behavior needed.
- **Preview tab with unsaved changes:** if the user somehow edits content that doesn't trigger auto-promotion (unlikely but defensive), the tab should auto-promote before being replaced.

### 1.3 Reference: VS Code Behavior

VS Code calls this "Preview Mode" (`workbench.editor.enablePreview`):

- Single-click in explorer → opens in preview (italic title)
- Double-click in explorer → opens as permanent
- Edit the file → preview becomes permanent
- Double-click the tab → preview becomes permanent
- Only one preview editor at a time; new preview replaces old
- Preview tab position: replaces the existing preview tab in-place (does not move to end)

Our design follows this closely with one deviation: we keep the preview tab at the tab bar end rather than replacing in-place, because our tab bar is simpler and positional replacement would feel jarring with fewer tabs.

---

## 2. Current Implementation — The Gap

### 2.1 Current behavior

Every file-open action creates a permanent tab. There is no concept of preview or temporary tabs. The `Tab` interface has `isPinned` for pinning but nothing for preview state.

Key code locations:

- **Tab type definition:** `src/stores/useFileStore.ts:49-68` — `Tab` interface, no `isPreview` field
- **`openFile()`:** `src/stores/useFileStore.ts:392-537` — always creates a permanent tab and appends to end
- **Sidebar click handler:** `src/components/layout/Sidebar.tsx:792-820` — `handleSelect()` calls `openFile()` on single-click, no double-click handler exists
- **Sidebar file item render:** `src/components/layout/Sidebar.tsx:1670-1672` — only `onClick`, no `onDoubleClick`
- **Tab bar render:** `src/components/layout/TabBar.tsx:29-113` — `TabItem` component, no italic styling for preview
- **`switchTab()`:** `src/stores/useFileStore.ts:539-579` — no preview-awareness
- **`closeTab()`:** `src/stores/useFileStore.ts:582-676` — no preview-awareness

### 2.2 What's missing

1. **`Tab.isPreview` field** — no way to mark a tab as preview
2. **Preview-aware open logic** — `openFile()` always appends; needs to replace existing preview tab
3. **Double-click handler on sidebar** — `Sidebar.tsx` file items only have `onClick`
4. **Promotion triggers** — no mechanism to convert preview → permanent on edit/double-click
5. **Visual styling** — `TabItem` has no italic state
6. **Store actions** — no `promotePreviewTab()` or `openFileAsPreview()` actions

---

## 3. Implementation Scope

### 3.1 Files to modify

#### Core store — `src/stores/useFileStore.ts`

This is the primary change. Modifications:

1. **Add `isPreview?: boolean` to `Tab` interface** (line 49-68)
   - Optional field, defaults to `false`/`undefined` for backward compatibility

2. **Add `openFileAsPreview(path)` action** or modify `openFile()` signature
   - Recommended: add an `options` parameter to `openFile()`:
     ```
     openFile(path: string, options?: { addToHistory?: boolean; forceReload?: boolean; preview?: boolean })
     ```
   - When `preview: true`:
     - Find existing preview tab (`tabs.find(t => t.isPreview)`)
     - If found: replace its content/path/name in-place (reuse the slot)
     - If not found: create new tab with `isPreview: true`, append to end
     - If the target file is already open as a permanent tab: just switch to it (don't create preview)

3. **Add `promotePreviewTab(tabId?: string)` action**
   - Sets `isPreview = false` on the target tab (or current preview tab)
   - Called by: edit trigger, double-click on tab, pin action, manual save

4. **Modify `closeTab()`** — no structural change needed, preview tabs close the same way

5. **Modify `togglePinTab()`** — if pinning a preview tab, also promote it (set `isPreview = false`)

6. **Modify `openDiagramTab()`, `openPDFTab()`** — add preview support
   - These methods also need a `preview?: boolean` option
   - Follow the same pattern: find existing preview → replace or create

#### Sidebar — `src/components/layout/Sidebar.tsx`

1. **Split `handleSelect()` into single-click (preview) and double-click (permanent)**
   - Single-click: call `openFile(path, { preview: true })`
   - Double-click: call `openFile(path, { preview: false })`
   - Need to handle click/double-click disambiguation (use a small timer ~200ms, or handle via `onDoubleClick` which naturally suppresses the second `onClick`)

2. **Add `onDoubleClick` to file tree item** (line 1670-1672)
   - Currently only has `onClick={() => onSelect(entry)}`
   - Add `onDoubleClick={() => onPermanentOpen(entry)}`

3. **Propagate `onPermanentOpen` callback through `FileTreeItem` props** (line 1374)

#### Tab bar — `src/components/layout/TabBar.tsx`

1. **Italic styling for preview tabs** (line 87)
   - Add `italic` class to `<span>` when `tab.isPreview`
   - Example: `className={cn("truncate max-w-[120px]", tab.isPreview && "italic")}`

2. **Double-click on tab header promotes it** (line 42-112)
   - Add `onDoubleClick` to the tab `<div>` that calls `promotePreviewTab(tab.id)`

3. **Update `useShallow` selector** to include `promotePreviewTab`

#### Editor integration — `src/editor/CodeMirrorEditor.tsx`

1. **Promote preview tab on first edit**
   - In the `updateContent()` or `onChange` handler, check if current tab is preview and call `promotePreviewTab()`
   - This should be a one-time check (once promoted, stop checking)
   - Could also be done in `useFileStore.updateContent()` directly in the store

   Recommended location: inside `updateContent()` in `useFileStore.ts`, since it's the central place for content changes:
   ```
   updateContent: (content, source, description) => {
     const { tabs, activeTabIndex } = get();
     const activeTab = tabs[activeTabIndex];
     if (activeTab?.isPreview) {
       // Auto-promote on edit
       get().promotePreviewTab(activeTab.id);
     }
     // ... existing logic
   }
   ```

#### Right panel — `src/components/layout/RightPanel.tsx`

1. **Backlink clicks → preview** (line 136)
   - Change `onClick={() => openFile(backlink.path)}` to `onClick={() => openFile(backlink.path, { preview: true })}`

2. **Related file clicks → preview** (line 231)
   - Same change

#### Knowledge graph — `src/components/effects/KnowledgeGraph.tsx`

1. **Node clicks → preview** (lines 852, 924, 1190, 1227)
   - Change `openFile()` calls to pass `{ preview: true }`

#### Local graph — `src/components/effects/LocalGraph.tsx`

1. **Node clicks → preview** (line 477)
   - Change `openFile()` call to pass `{ preview: true }`

#### Image manager — `src/components/images/ImageManagerView.tsx`

1. **"Referenced by" file clicks → preview** (line 421)
   - Change to `openFile(path, { preview: true })`

### 3.2 Files that should NOT change

These entry points should keep creating permanent tabs:

- `src/components/search/CommandPalette.tsx` — command palette opens are intentional
- `src/components/search/GlobalSearch.tsx` — search result opens are intentional
- `src/components/layout/Ribbon.tsx` — ribbon buttons open singleton tabs
- `src/components/chat/AgentMessageRenderer.tsx` — AI agent file opens
- `src/components/layout/MainAIChatShell.tsx` — AI chat file opens
- `src/hooks/useVoiceNote.ts` — voice note creation
- `src/components/database/` — database view interactions
- `src/components/cardflow/CardFlowView.tsx` — card flow interactions
- `src/services/plugins/runtime.ts` — plugin API
- `src/editor/CodeMirrorEditor.tsx` — Ctrl+Click internal links (deliberate action)
- `src/editor/ReadingView.tsx` — reading view link clicks (deliberate action)
- Sidebar favorites section (line 1056) — favorites are intentional

### 3.3 Click disambiguation strategy

The sidebar needs to distinguish single-click from double-click. Two approaches:

**Option A: Timer-based disambiguation**
- On click, set a 200ms timeout
- If double-click fires within 200ms, cancel the timeout and open permanent
- If timeout expires, open as preview
- Downside: 200ms delay before preview opens, feels sluggish

**Option B: Immediate open + promote on double-click (recommended)**
- Single-click immediately opens as preview (no delay)
- Double-click: the first click opens preview, the second click promotes it to permanent
- This is how VS Code does it
- No perceptible delay, feels instant
- Implementation: `onClick` → preview open, `onDoubleClick` → promote

Option B is strictly better because there's zero latency on single-click.

### 3.4 Edge cases

1. **Preview tab has unsaved changes when being replaced**
   - Should not happen because editing promotes the tab
   - Defensive: check `isDirty` before replacing; if dirty, promote instead of replacing

2. **Same file opened as preview, then double-clicked**
   - Just promote the existing preview tab, don't create a new tab

3. **File already open as permanent tab, single-clicked in sidebar**
   - Switch to the existing permanent tab; do NOT create a preview

4. **Split editor**
   - Preview behavior only applies to the primary pane
   - Secondary pane opens are always permanent (split view is an intentional layout action)

5. **Drag-and-drop reorder of preview tab**
   - Allowed; preview tab can be reordered without promoting it

---

## 4. Summary of Changes by Priority

| Priority | Scope | Description |
|----------|-------|-------------|
| P0 | `useFileStore.ts` | Add `isPreview` to `Tab`, add preview-aware open logic, add `promotePreviewTab()` |
| P0 | `Sidebar.tsx` | Add double-click handler, single-click opens preview |
| P0 | `TabBar.tsx` | Italic styling for preview, double-click to promote |
| P0 | `useFileStore.ts` | Auto-promote on edit (in `updateContent()`) |
| P1 | `RightPanel.tsx` | Backlink/related file clicks → preview |
| P1 | `KnowledgeGraph.tsx`, `LocalGraph.tsx` | Graph node clicks → preview |
| P1 | `ImageManagerView.tsx` | Referenced-by clicks → preview |
| P2 | i18n | Add tooltip/context-menu text if needed (e.g., "Keep Open") |

P0 items deliver the core feature. P1 items extend preview to other browsing surfaces. P2 is polish.
