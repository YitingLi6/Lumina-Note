# Proxy Configuration Design

**Issue:** [#167](https://github.com/blueberrycongee/Lumina-Note/issues/167)
**Date:** 2026-03-12

## Goal

Add a global proxy configuration so all HTTP requests (LLM API, Codex/Marketplace, Tavily search, cloud sync, updates) can be routed through a user-specified proxy.

## Architecture

### Data Flow

```
Settings UI (ProxySection)
  → useSettingsStore (proxyUrl, proxyEnabled) — persisted to localStorage
  → Tauri command: set_proxy_config(url, enabled)
  → Rust ProxyState: rebuild shared reqwest::Client with proxy
  → All HTTP requests use the shared client
```

### Rust: Global HTTP Client (`src-tauri/src/proxy.rs`)

- `ProxyState` holds `Arc<RwLock<reqwest::Client>>`
- Managed as Tauri state (`app.manage(ProxyState::new())`)
- On config change, rebuild the client with `reqwest::Proxy::all(url)` or no proxy
- All ~11 call sites replace `Client::new()` / `Client::builder()` with `proxy_state.client()`

### Supported Proxy Formats

- `http://host:port`
- `https://host:port`
- `socks5://host:port` (reqwest native support)
- `http://user:pass@host:port` (authenticated)

### Frontend Store

Add to `useSettingsStore`:
- `proxyUrl: string` (default `""`)
- `proxyEnabled: boolean` (default `false`)
- `setProxyConfig(url, enabled)` — calls Tauri command, updates store

### Settings UI

New `ProxySection` component in SettingsModal:
- Text input for proxy URL
- Toggle switch for enable/disable
- "Test Connection" button (calls `test_proxy_connection` command)

### Node.js / Codex Integration

When launching Codex extensions, inject `HTTP_PROXY` and `HTTPS_PROXY` environment variables from the stored proxy config.

### i18n

All 4 locales (en, zh-CN, zh-TW, ja) updated with proxy-related keys under `settingsModal.proxy.*`.

### Tauri Commands

- `set_proxy_config(proxy_url: String, enabled: bool)` — rebuild client
- `get_proxy_config() -> { proxy_url, enabled }` — read current config
- `test_proxy_connection(proxy_url: String) -> Result<(), String>` — verify connectivity

## Changes Summary

| Area | Files |
|------|-------|
| Rust proxy module | `src-tauri/src/proxy.rs` (new) |
| Rust main/lib | Register ProxyState, add commands |
| Rust HTTP call sites | ~11 files: llm.rs, agent/llm_client.rs, codex_extension.rs, etc. |
| Frontend store | `useSettingsStore.ts` |
| Frontend UI | `ProxySection.tsx` (new) |
| Frontend settings | `SettingsModal.tsx` (add section) |
| i18n | 4 locale files |
| Node.js runtime | `node_runtime.rs` (inject env vars) |
