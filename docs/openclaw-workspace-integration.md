# OpenClaw Workspace Integration

This document defines the current Lumina Note integration contract for OpenClaw workspaces.

## Status

Current status: Phase 1 in-product integration is available.

Implemented in Lumina:

- Detect whether an external folder matches an OpenClaw workspace contract.
- Allow the user to explicitly attach or detach an external OpenClaw workspace onto the current Lumina workspace.
- Surface key memory files, recent daily memory files, conventional plan files, artifacts, and Lumina bridge notes inside Lumina.
- Treat OpenClaw memory and artifact files as normal Lumina materials instead of copying them into a separate store.
- Ship a bundled official `OpenClaw Workspace` plugin with overview, manual refresh, bridge-note staging, and status affordances.
- Add filtered workspace views for OpenClaw memories, plans, and artifacts through Lumina global search scopes.
- Persist optional Gateway endpoint metadata and expose concurrent-change warnings when Lumina and OpenClaw touch the same dirty file.

Not implemented yet:

- Gateway-backed live transport or auth flows beyond local file-centric attachment.
- Aggressive conflict resolution beyond warning-level visibility and normal local file refresh behavior.
- A separate import/sync layer or remote admin workflow.

## Verified Workspace Contract

This integration is intentionally file-first. Lumina works against OpenClaw's workspace files rather than trying to replace OpenClaw runtime, Gateway, or IM-facing surfaces.

Verified references on March 12, 2026:

- [Personal Assistant Setup](https://docs.openclaw.ai/clawd)
- [Default AGENTS.md](https://docs.openclaw.ai/reference/AGENTS.default)
- [FAQ](https://docs.openclaw.ai/start/faq/)
- [Sandboxing](https://docs.openclaw.ai/gateway/sandboxing)
- [Skills](https://docs.openclaw.ai/skills)

Workspace markers Lumina currently treats as first-class:

- Root files: `AGENTS.md`, `SOUL.md`, `USER.md`
- Additional known root files: `IDENTITY.md`, `TOOLS.md`, `HEARTBEAT.md`, `BOOT.md`, `BOOTSTRAP.md`, `MEMORY.md`
- Known folders: `memory/`, `skills/`, `canvas/`, `output/`
- Daily memory pattern: `memory/YYYY-MM-DD.md`

Local verification used during implementation:

- A live local OpenClaw workspace existed at `~/.openclaw/workspace` on March 12, 2026
- Observed root files included `AGENTS.md`, `SOUL.md`, `USER.md`, `IDENTITY.md`, `TOOLS.md`, `HEARTBEAT.md`, `MEMORY.md`
- Observed folders included `memory/`, `skills/`, `output/`, and `tmp/`
- Observed artifact output included `output/playwright/`
- No conventional `plans/` directory was present in that local workspace at validation time

## Product Contract

Lumina currently supports one explicit integration model:

- Open any normal Lumina workspace as the host workspace.
- Optionally attach one external OpenClaw workspace path onto that host workspace.

Behavioral rules:

- Detection is driven by the configured OpenClaw path when Lumina loads or refreshes that mounted file tree.
- Attachment is explicit and user-controlled.
- Editing in Lumina changes the same files OpenClaw reads and writes.
- Lumina does not create a second storage format for OpenClaw materials.
- Search, markdown editing, retrieval, and project organization can operate across the host workspace and the mounted OpenClaw workspace.
- Conventional `plans/` folders and staged `.lumina/openclaw-bridge-*.md` notes get special affordances, but they remain ordinary workspace files.
- The mounted OpenClaw root is rendered as an additional source inside Sidebar and filtered search scopes; it is not imported into the host vault.

## Security And Support Boundaries

Lumina is not the OpenClaw execution engine.

Out of scope for this integration:

- Replacing OpenClaw Gateway
- Replacing OpenClaw sandboxing
- Rebuilding OpenClaw's chat or IM trigger surfaces inside Lumina
- Managing OpenClaw credentials or Gateway auth flows

Important user-facing boundary:

- If a file is edited in Lumina, that edit is written to the real workspace on disk.
- If OpenClaw edits the same file nearby in time, Lumina follows normal local file refresh behavior; it does not provide CRDT merge or distributed conflict resolution.

## Failure Boundaries

Expected failure modes:

- The configured OpenClaw path is not an OpenClaw workspace.
- The mounted workspace path becomes unavailable or is moved.
- Required root markers disappear after attachment.
- OpenClaw writes new files that Lumina has not refreshed yet.

Current handling:

- Lumina keeps detection state per host workspace path.
- Attachment can be cleared manually.
- Missing markers degrade the integration affordances instead of blocking normal file browsing.
- Workspace refresh updates grouped memory / plan / artifact affordances when the mounted file tree is reloaded.
- Settings and the built-in plugin surface warning-only conflict visibility when attached OpenClaw files overlap with dirty Lumina files.
- A release flag can disable the feature at build time: `VITE_ENABLE_OPENCLAW_WORKSPACE=0`.

## Validation Checklist

Completed validation for the current implementation:

- Unit tests for workspace detection and derived memory / plan / artifact metadata
- Unit tests for persisted OpenClaw workspace store behavior
- Unit tests for built-in plugin enablement defaults
- TypeScript compile check with `npx tsc --noEmit`
- Production web build with `npm run build`
- Local filesystem verification against a real `~/.openclaw/workspace` on March 12, 2026

Recommended manual checks before broader rollout:

- Open any normal Lumina workspace
- In settings, pick a real `~/.openclaw/workspace` as the mounted OpenClaw path
- Confirm the settings panel shows detected markers and attachment status
- Attach the mounted workspace and verify the bundled plugin overview opens
- Open `AGENTS.md` and a recent `memory/YYYY-MM-DD.md` from Sidebar and Overview
- Stage the current note or selection and confirm a `.lumina/openclaw-bridge-*.md` file is created
- Use filtered views for memories, plans, and artifacts from Sidebar or Overview
- Refresh after creating a new file under `output/` or `canvas/`
- Rename or remove the mounted workspace path and confirm Lumina degrades safely
