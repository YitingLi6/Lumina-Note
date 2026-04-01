# Proxy Configuration Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a global proxy configuration so all HTTP requests route through a user-specified proxy server.

**Architecture:** A Rust-side `ProxyState` manages a shared `reqwest::Client` behind `Arc<RwLock<>>`. All ~11 call sites that create their own client switch to using this shared client. The frontend stores `proxyUrl`/`proxyEnabled` in `useUIStore` and syncs to Rust via Tauri commands. A `ProxySection` component in SettingsModal provides the UI.

**Tech Stack:** Rust (reqwest with socks feature), Tauri 2 commands, React + TypeScript, Zustand, Tailwind CSS

---

### Task 1: Enable reqwest socks feature in Cargo.toml

**Files:**
- Modify: `src-tauri/Cargo.toml:43`

**Step 1: Add socks feature**

Change line 43 from:
```toml
reqwest = { version = "0.13", features = ["json", "stream"] }
```
to:
```toml
reqwest = { version = "0.13", features = ["json", "stream", "socks"] }
```

**Step 2: Verify it compiles**

Run: `cd src-tauri && cargo check 2>&1 | tail -5`
Expected: `Finished` (no errors)

**Step 3: Commit**

```bash
git add src-tauri/Cargo.toml
git commit -m "feat(proxy): enable reqwest socks feature for SOCKS5 proxy support"
```

---

### Task 2: Create Rust proxy module (`proxy.rs`)

**Files:**
- Create: `src-tauri/src/proxy.rs`
- Modify: `src-tauri/src/main.rs:7-24` (add `mod proxy;`)
- Modify: `src-tauri/src/lib.rs` (add `pub mod proxy;`)

**Step 1: Write `proxy.rs`**

```rust
use reqwest::Client;
use std::sync::Arc;
use std::time::Duration;
use tokio::sync::RwLock;

/// Global proxy configuration and shared HTTP client.
pub struct ProxyState {
    client: Arc<RwLock<Client>>,
    config: Arc<RwLock<ProxyConfig>>,
}

#[derive(Clone, serde::Serialize, serde::Deserialize)]
pub struct ProxyConfig {
    pub proxy_url: String,
    pub enabled: bool,
}

impl Default for ProxyConfig {
    fn default() -> Self {
        Self {
            proxy_url: String::new(),
            enabled: false,
        }
    }
}

impl ProxyState {
    pub fn new() -> Self {
        Self {
            client: Arc::new(RwLock::new(build_client(None).expect("default client"))),
            config: Arc::new(RwLock::new(ProxyConfig::default())),
        }
    }

    /// Get a clone of the current shared client.
    pub async fn client(&self) -> Client {
        self.client.read().await.clone()
    }

    /// Get a client with a custom timeout (still respects proxy).
    pub async fn client_with_timeout(&self, timeout: Duration) -> Result<Client, String> {
        let config = self.config.read().await;
        build_client_with_timeout(
            if config.enabled { Some(&config.proxy_url) } else { None },
            timeout,
        )
    }

    /// Update proxy config and rebuild the shared client.
    pub async fn set_config(&self, proxy_url: String, enabled: bool) -> Result<(), String> {
        let proxy = if enabled { Some(proxy_url.as_str()) } else { None };
        let new_client = build_client(proxy)?;
        {
            let mut cfg = self.config.write().await;
            cfg.proxy_url = proxy_url;
            cfg.enabled = enabled;
        }
        {
            let mut c = self.client.write().await;
            *c = new_client;
        }
        Ok(())
    }

    pub async fn get_config(&self) -> ProxyConfig {
        self.config.read().await.clone()
    }
}

fn build_client(proxy_url: Option<&str>) -> Result<Client, String> {
    build_client_with_timeout(proxy_url, Duration::from_secs(120))
}

fn build_client_with_timeout(proxy_url: Option<&str>, timeout: Duration) -> Result<Client, String> {
    let mut builder = Client::builder().timeout(timeout);
    if let Some(url) = proxy_url {
        if !url.is_empty() {
            let proxy = reqwest::Proxy::all(url)
                .map_err(|e| format!("Invalid proxy URL: {}", e))?;
            builder = builder.proxy(proxy);
        }
    }
    builder.build().map_err(|e| format!("Failed to build HTTP client: {}", e))
}

// ── Tauri commands ──

#[tauri::command]
pub async fn set_proxy_config(
    state: tauri::State<'_, ProxyState>,
    proxy_url: String,
    enabled: bool,
) -> Result<(), String> {
    state.set_config(proxy_url, enabled).await
}

#[tauri::command]
pub async fn get_proxy_config(
    state: tauri::State<'_, ProxyState>,
) -> Result<ProxyConfig, String> {
    Ok(state.get_config().await)
}

#[tauri::command]
pub async fn test_proxy_connection(
    proxy_url: String,
) -> Result<(), String> {
    let proxy = reqwest::Proxy::all(&proxy_url)
        .map_err(|e| format!("Invalid proxy URL: {}", e))?;
    let client = Client::builder()
        .proxy(proxy)
        .timeout(Duration::from_secs(10))
        .build()
        .map_err(|e| format!("Failed to build client: {}", e))?;
    client
        .get("https://httpbin.org/ip")
        .send()
        .await
        .map_err(|e| format!("Connection failed: {}", e))?;
    Ok(())
}
```

**Step 2: Register module in `main.rs`**

Add after line 24 (`mod webdav;`):
```rust
mod proxy;
```

**Step 3: Register module in `lib.rs`**

Add:
```rust
pub mod proxy;
```

**Step 4: Verify it compiles**

Run: `cd src-tauri && cargo check 2>&1 | tail -5`

**Step 5: Commit**

```bash
git add src-tauri/src/proxy.rs src-tauri/src/main.rs src-tauri/src/lib.rs
git commit -m "feat(proxy): add ProxyState module with shared HTTP client and Tauri commands"
```

---

### Task 3: Register ProxyState and commands in main.rs

**Files:**
- Modify: `src-tauri/src/main.rs`

**Step 1: Add commands to invoke_handler**

In `main.rs`, inside `.invoke_handler(tauri::generate_handler![...])`, add after the Cloud Relay commands block (after line 224):
```rust
            // Proxy commands
            proxy::set_proxy_config,
            proxy::get_proxy_config,
            proxy::test_proxy_connection,
```

**Step 2: Add state management**

After line 238 (`.manage(commands::ChildWebviewBoundsState::default())`), add:
```rust
        .manage(proxy::ProxyState::new())
```

**Step 3: Verify it compiles**

Run: `cd src-tauri && cargo check 2>&1 | tail -5`

**Step 4: Commit**

```bash
git add src-tauri/src/main.rs
git commit -m "feat(proxy): register ProxyState and proxy commands in Tauri app"
```

---

### Task 4: Migrate HTTP call sites to use ProxyState

This is the largest task. Each file that creates `reqwest::Client` must be updated to accept or extract `ProxyState`.

**Files to modify (9 files, ~11 call sites):**

1. `src-tauri/src/llm.rs` (2 sites: `llm_fetch`, `llm_fetch_stream`)
2. `src-tauri/src/cloud_relay.rs` (1 site: `login_for_token`)
3. `src-tauri/src/agent/deep_research/tavily.rs` (1 site: `TavilyClient::new`)
4. `src-tauri/src/agent/deep_research/crawler.rs` (1 site: `JinaClient::new`)
5. `src-tauri/src/agent/llm_client.rs` (1 site: `LlmClient::new`)
6. `src-tauri/src/codex_extension.rs` (2 sites: marketplace query + VSIX download)
7. `src-tauri/src/forge_runtime/tools/fetch.rs` (1 site: fetch tool)
8. `src-tauri/src/update_manager.rs` (1 site: `download_once`)
9. `src-tauri/src/webdav/client.rs` (1 site: `WebDAVClient::new`)
10. `src-tauri/src/commands/mod.rs` (2 sites: bilibili CID + danmaku)

**Strategy per file:**

For **Tauri command** functions (have `#[tauri::command]`), add `proxy_state: tauri::State<'_, crate::proxy::ProxyState>` parameter and use `proxy_state.client_with_timeout(duration).await?` instead of `Client::builder()...build()`.

For **non-command** functions (called internally like `TavilyClient::new`, `LlmClient::new`, `WebDAVClient::new`, `JinaClient::new`), change the constructor to accept `Client` and have the caller pass in the proxy-aware client. Trace the call chain upward to the Tauri command that creates these structs.

**Step 1: Migrate `llm.rs`**

In `llm_fetch` (line ~30), replace:
```rust
let client = reqwest::Client::builder()
    .timeout(std::time::Duration::from_secs(request.timeout_secs.unwrap_or(120)))
    .build()
    .map_err(|e| format!("Failed to create HTTP client: {}", e))?;
```
with:
```rust
let client = proxy_state
    .client_with_timeout(std::time::Duration::from_secs(request.timeout_secs.unwrap_or(120)))
    .await?;
```
Add `proxy_state: tauri::State<'_, crate::proxy::ProxyState>` to the function signature.

Do the same for `llm_fetch_stream` (line ~115), adding the `proxy_state` parameter.

**Step 2: Migrate `cloud_relay.rs`**

`login_for_token` is a private async function, not a Tauri command. Trace its callers — it's likely called from a Tauri command that can pass through the client. Change `login_for_token` to accept a `&Client` parameter. Build the client from ProxyState at the Tauri command call site.

**Step 3: Migrate `agent/deep_research/tavily.rs`**

Change `TavilyClient::new` to accept `client: reqwest::Client` instead of building its own. Trace callers to pass the proxy-aware client.

**Step 4: Migrate `agent/deep_research/crawler.rs`**

Change `JinaClient::new` to accept `client: reqwest::Client`.

**Step 5: Migrate `agent/llm_client.rs`**

Change `LlmClient::new` to accept `client: reqwest::Client`.

**Step 6: Migrate `codex_extension.rs`**

Both functions (`marketplace_latest_openai_chatgpt` and `download_openai_chatgpt_extension`) are likely called from Tauri commands. Add `ProxyState` parameter to the Tauri command and pass a client down.

**Step 7: Migrate `forge_runtime/tools/fetch.rs`**

The fetch tool handler needs access to ProxyState. Check how `ToolContext`/`ToolEnvironment` is structured — the proxy client may need to be added to the environment or context.

**Step 8: Migrate `update_manager.rs`**

`download_once` is internal. Pass a `Client` from the Tauri command that starts the update.

**Step 9: Migrate `webdav/client.rs`**

Change `WebDAVClient::new` to accept an optional `Client`.

**Step 10: Migrate `commands/mod.rs`**

For `get_bilibili_cid` and `get_bilibili_danmaku`, add `proxy_state` parameter (they are Tauri commands).

**Step 11: Verify compilation**

Run: `cd src-tauri && cargo check 2>&1 | tail -20`
Fix any remaining compilation errors.

**Step 12: Commit**

```bash
git add src-tauri/src/
git commit -m "feat(proxy): migrate all HTTP call sites to use shared ProxyState client"
```

---

### Task 5: Add i18n keys for proxy settings

**Files:**
- Modify: `src/i18n/locales/en.ts` (insert before `about:` line)
- Modify: `src/i18n/locales/zh-CN.ts` (insert before `about:` line)
- Modify: `src/i18n/locales/zh-TW.ts` (insert before `about:` line)
- Modify: `src/i18n/locales/ja.ts` (insert before `about:` line)

**Step 1: Add keys to all 4 locales**

Insert before the `about:` key in each file's `settingsModal` section:

**en.ts** (before line 1208):
```typescript
    proxyTitle: 'Proxy',
    proxyDesc: 'Route all network requests through a proxy server',
    proxyUrl: 'Proxy URL',
    proxyUrlPlaceholder: 'http://127.0.0.1:7890',
    proxyEnable: 'Enable proxy',
    proxyTestConnection: 'Test connection',
    proxyTestSuccess: 'Connection successful',
    proxyTestFailed: 'Connection failed: {error}',
    proxyInvalidUrl: 'Invalid proxy URL format',
    proxyHint: 'Supports HTTP, HTTPS, and SOCKS5 protocols',
```

**zh-CN.ts** (before line 1209):
```typescript
    proxyTitle: '代理',
    proxyDesc: '通过代理服务器转发所有网络请求',
    proxyUrl: '代理地址',
    proxyUrlPlaceholder: 'http://127.0.0.1:7890',
    proxyEnable: '启用代理',
    proxyTestConnection: '测试连接',
    proxyTestSuccess: '连接成功',
    proxyTestFailed: '连接失败：{error}',
    proxyInvalidUrl: '代理地址格式无效',
    proxyHint: '支持 HTTP、HTTPS 和 SOCKS5 协议',
```

**zh-TW.ts** (before line 1103):
```typescript
    proxyTitle: '代理',
    proxyDesc: '透過代理伺服器轉發所有網路請求',
    proxyUrl: '代理位址',
    proxyUrlPlaceholder: 'http://127.0.0.1:7890',
    proxyEnable: '啟用代理',
    proxyTestConnection: '測試連線',
    proxyTestSuccess: '連線成功',
    proxyTestFailed: '連線失敗：{error}',
    proxyInvalidUrl: '代理位址格式無效',
    proxyHint: '支援 HTTP、HTTPS 和 SOCKS5 協定',
```

**ja.ts** (before line 1103):
```typescript
    proxyTitle: 'プロキシ',
    proxyDesc: 'すべてのネットワークリクエストをプロキシサーバー経由で送信します',
    proxyUrl: 'プロキシURL',
    proxyUrlPlaceholder: 'http://127.0.0.1:7890',
    proxyEnable: 'プロキシを有効にする',
    proxyTestConnection: '接続テスト',
    proxyTestSuccess: '接続成功',
    proxyTestFailed: '接続失敗：{error}',
    proxyInvalidUrl: 'プロキシURLの形式が無効です',
    proxyHint: 'HTTP、HTTPS、SOCKS5プロトコルに対応',
```

**Step 2: Verify TypeScript compilation**

Run: `npx tsc --noEmit 2>&1 | head -20`
Expected: no errors (all locales have matching structure)

**Step 3: Commit**

```bash
git add src/i18n/locales/
git commit -m "feat(proxy): add i18n keys for proxy settings in all 4 locales"
```

---

### Task 6: Add proxy fields to useUIStore

**Files:**
- Modify: `src/stores/useUIStore.ts`

**Step 1: Add proxy state to UIState interface** (after line 91 `setEditorInteractionTraceEnabled`)

```typescript
  // Proxy
  proxyUrl: string;
  proxyEnabled: boolean;
  setProxyUrl: (url: string) => void;
  setProxyEnabled: (enabled: boolean) => void;
```

**Step 2: Add to store implementation** (after `setEditorFontSize` around line 229)

```typescript
      // Proxy
      proxyUrl: "",
      proxyEnabled: false,
      setProxyUrl: (url) => set({ proxyUrl: url }),
      setProxyEnabled: (enabled) => set({ proxyEnabled: enabled }),
```

**Step 3: Add to `partializeUIState`** (around line 117)

```typescript
  proxyUrl: state.proxyUrl,
  proxyEnabled: state.proxyEnabled,
```

**Step 4: Verify TypeScript compilation**

Run: `npx tsc --noEmit 2>&1 | head -10`

**Step 5: Commit**

```bash
git add src/stores/useUIStore.ts
git commit -m "feat(proxy): add proxyUrl and proxyEnabled to useUIStore"
```

---

### Task 7: Create ProxySection settings component

**Files:**
- Create: `src/components/settings/ProxySection.tsx`
- Modify: `src/components/layout/SettingsModal.tsx`

**Step 1: Create ProxySection.tsx**

```tsx
import { useState, useEffect } from "react";
import { invoke } from "@tauri-apps/api/core";
import { Globe, Loader2 } from "lucide-react";
import { useLocaleStore } from "@/stores/useLocaleStore";
import { useUIStore } from "@/stores/useUIStore";
import { reportOperationError } from "@/lib/reportError";

export function ProxySection() {
  const { t } = useLocaleStore();
  const { proxyUrl, proxyEnabled, setProxyUrl, setProxyEnabled } = useUIStore();
  const [testing, setTesting] = useState(false);
  const [testResult, setTestResult] = useState<{ ok: boolean; msg: string } | null>(null);

  // Sync proxy config to Rust backend whenever it changes
  useEffect(() => {
    invoke("set_proxy_config", {
      proxyUrl: proxyUrl,
      enabled: proxyEnabled,
    }).catch((err) => {
      reportOperationError({
        source: "ProxySection",
        action: "Sync proxy config",
        error: err,
        level: "warning",
      });
    });
  }, [proxyUrl, proxyEnabled]);

  // Load saved config from Rust on mount
  useEffect(() => {
    invoke<{ proxy_url: string; enabled: boolean }>("get_proxy_config")
      .then((config) => {
        if (config.proxy_url && config.proxy_url !== proxyUrl) {
          setProxyUrl(config.proxy_url);
        }
        if (config.enabled !== proxyEnabled) {
          setProxyEnabled(config.enabled);
        }
      })
      .catch(() => {
        // First launch, no config yet — ignore
      });
    // Only on mount
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  const handleTest = async () => {
    if (!proxyUrl.trim()) return;
    setTesting(true);
    setTestResult(null);
    try {
      await invoke("test_proxy_connection", { proxyUrl: proxyUrl.trim() });
      setTestResult({ ok: true, msg: t.settingsModal.proxyTestSuccess });
    } catch (err) {
      setTestResult({
        ok: false,
        msg: t.settingsModal.proxyTestFailed.replace("{error}", String(err)),
      });
    } finally {
      setTesting(false);
    }
  };

  return (
    <section className="space-y-4 rounded-xl border border-border bg-background/60 p-4">
      <div>
        <h3 className="text-sm font-medium text-foreground/90 flex items-center gap-2">
          <Globe size={14} />
          {t.settingsModal.proxyTitle}
        </h3>
        <p className="text-xs text-muted-foreground mt-1">
          {t.settingsModal.proxyDesc}
        </p>
      </div>

      <div className="space-y-3">
        <div className="space-y-1">
          <label className="text-xs text-muted-foreground">
            {t.settingsModal.proxyUrl}
          </label>
          <input
            type="text"
            value={proxyUrl}
            onChange={(e) => setProxyUrl(e.target.value)}
            placeholder={t.settingsModal.proxyUrlPlaceholder}
            className="w-full rounded-md border border-border bg-background px-3 py-2 text-xs text-foreground"
          />
          <p className="text-[10px] text-muted-foreground">
            {t.settingsModal.proxyHint}
          </p>
        </div>

        <div className="flex items-center justify-between">
          <span className="text-xs text-foreground/80">
            {t.settingsModal.proxyEnable}
          </span>
          <button
            type="button"
            role="switch"
            aria-checked={proxyEnabled}
            onClick={() => setProxyEnabled(!proxyEnabled)}
            className={`relative inline-flex h-5 w-9 items-center rounded-full transition-colors ${
              proxyEnabled ? "bg-primary" : "bg-muted"
            }`}
          >
            <span
              className={`inline-block h-3.5 w-3.5 rounded-full bg-white transition-transform ${
                proxyEnabled ? "translate-x-4.5" : "translate-x-0.5"
              }`}
            />
          </button>
        </div>

        <button
          type="button"
          onClick={handleTest}
          disabled={testing || !proxyUrl.trim()}
          className="inline-flex items-center gap-1.5 rounded-lg border border-border px-3 py-1.5 text-xs font-medium text-foreground hover:bg-muted disabled:opacity-60"
        >
          {testing && <Loader2 size={12} className="animate-spin" />}
          {t.settingsModal.proxyTestConnection}
        </button>

        {testResult && (
          <p className={`text-xs ${testResult.ok ? "text-green-500" : "text-red-500"}`}>
            {testResult.msg}
          </p>
        )}
      </div>
    </section>
  );
}
```

**Step 2: Add ProxySection to SettingsModal.tsx**

Add import at top (after line 28):
```typescript
import { ProxySection } from "../settings/ProxySection";
```

Add the component in JSX, after `<CloudRelaySection />` (line 351) and before the WebDAV section:
```tsx
          <ProxySection />
```

**Step 3: Verify TypeScript compilation**

Run: `npx tsc --noEmit 2>&1 | head -10`

**Step 4: Commit**

```bash
git add src/components/settings/ProxySection.tsx src/components/layout/SettingsModal.tsx
git commit -m "feat(proxy): add ProxySection settings UI component"
```

---

### Task 8: Inject proxy env vars for Node.js / Codex extension

**Files:**
- Modify: `src-tauri/src/node_runtime.rs` or the codex extension spawn site

**Step 1: Find where Codex Node.js process is spawned**

Search for where `Command::new` or process spawn happens with the Node.js binary for codex. The proxy URL needs to be injected as `HTTP_PROXY` and `HTTPS_PROXY` env vars.

**Step 2: Inject proxy environment variables**

At the spawn site, read proxy config from `ProxyState` and inject:
```rust
if config.enabled && !config.proxy_url.is_empty() {
    cmd.env("HTTP_PROXY", &config.proxy_url);
    cmd.env("HTTPS_PROXY", &config.proxy_url);
}
```

**Step 3: Verify compilation**

Run: `cd src-tauri && cargo check 2>&1 | tail -5`

**Step 4: Commit**

```bash
git add src-tauri/src/
git commit -m "feat(proxy): inject HTTP_PROXY/HTTPS_PROXY env vars for Codex Node.js runtime"
```

---

### Task 9: Frontend tests for ProxySection

**Files:**
- Create: `src/components/settings/__tests__/ProxySection.test.tsx`

**Step 1: Write component tests**

```tsx
import { describe, it, expect, vi, beforeEach } from "vitest";
import { render, screen, fireEvent, waitFor } from "@testing-library/react";
import { ProxySection } from "../ProxySection";
import en from "@/i18n/locales/en";

vi.mock("@/stores/useLocaleStore", () => ({
  useLocaleStore: () => ({ t: en }),
  getCurrentTranslations: () => en,
}));

const mockSetProxyUrl = vi.fn();
const mockSetProxyEnabled = vi.fn();

vi.mock("@/stores/useUIStore", () => ({
  useUIStore: () => ({
    proxyUrl: "",
    proxyEnabled: false,
    setProxyUrl: mockSetProxyUrl,
    setProxyEnabled: mockSetProxyEnabled,
  }),
}));

const mockInvoke = vi.fn();
vi.mock("@tauri-apps/api/core", () => ({
  invoke: (...args: unknown[]) => mockInvoke(...args),
}));

describe("ProxySection", () => {
  beforeEach(() => {
    vi.clearAllMocks();
    mockInvoke.mockResolvedValue({ proxy_url: "", enabled: false });
  });

  it("renders title and input", () => {
    render(<ProxySection />);
    expect(screen.getByText(en.settingsModal.proxyTitle)).toBeInTheDocument();
    expect(screen.getByPlaceholderText(en.settingsModal.proxyUrlPlaceholder)).toBeInTheDocument();
  });

  it("calls setProxyUrl on input change", () => {
    render(<ProxySection />);
    const input = screen.getByPlaceholderText(en.settingsModal.proxyUrlPlaceholder);
    fireEvent.change(input, { target: { value: "http://127.0.0.1:7890" } });
    expect(mockSetProxyUrl).toHaveBeenCalledWith("http://127.0.0.1:7890");
  });

  it("calls test_proxy_connection on test button click", async () => {
    mockInvoke.mockImplementation((cmd: string) => {
      if (cmd === "get_proxy_config") return Promise.resolve({ proxy_url: "", enabled: false });
      if (cmd === "set_proxy_config") return Promise.resolve();
      if (cmd === "test_proxy_connection") return Promise.resolve();
      return Promise.resolve();
    });

    // Re-mock with a proxy URL so button is enabled
    vi.mocked(mockSetProxyUrl).mockClear();
    const { unmount } = render(<ProxySection />);
    unmount();

    // Need to re-render with proxyUrl set — adjust mock
    const useUIStoreMod = await import("@/stores/useUIStore");
    vi.mocked(useUIStoreMod.useUIStore).mockReturnValue({
      proxyUrl: "http://127.0.0.1:7890",
      proxyEnabled: false,
      setProxyUrl: mockSetProxyUrl,
      setProxyEnabled: mockSetProxyEnabled,
    } as any);

    render(<ProxySection />);
    const btn = screen.getByText(en.settingsModal.proxyTestConnection);
    fireEvent.click(btn);
    await waitFor(() => {
      expect(mockInvoke).toHaveBeenCalledWith("test_proxy_connection", {
        proxyUrl: "http://127.0.0.1:7890",
      });
    });
  });
});
```

**Step 2: Run tests**

Run: `npx vitest run src/components/settings/__tests__/ProxySection.test.tsx`
Expected: all pass

**Step 3: Commit**

```bash
git add src/components/settings/__tests__/ProxySection.test.tsx
git commit -m "test(proxy): add ProxySection component tests"
```

---

### Task 10: Rust unit tests for proxy module

**Files:**
- Modify: `src-tauri/src/proxy.rs` (add `#[cfg(test)]` module at bottom)

**Step 1: Add tests**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn build_client_no_proxy() {
        let client = build_client(None);
        assert!(client.is_ok());
    }

    #[test]
    fn build_client_with_http_proxy() {
        let client = build_client(Some("http://127.0.0.1:7890"));
        assert!(client.is_ok());
    }

    #[test]
    fn build_client_with_socks5_proxy() {
        let client = build_client(Some("socks5://127.0.0.1:1080"));
        assert!(client.is_ok());
    }

    #[test]
    fn build_client_with_empty_proxy() {
        let client = build_client(Some(""));
        assert!(client.is_ok());
    }

    #[test]
    fn build_client_with_invalid_proxy() {
        let client = build_client(Some("not-a-url"));
        assert!(client.is_err());
    }

    #[tokio::test]
    async fn proxy_state_default_config() {
        let state = ProxyState::new();
        let config = state.get_config().await;
        assert!(!config.enabled);
        assert!(config.proxy_url.is_empty());
    }

    #[tokio::test]
    async fn proxy_state_set_config() {
        let state = ProxyState::new();
        state
            .set_config("http://127.0.0.1:7890".into(), true)
            .await
            .unwrap();
        let config = state.get_config().await;
        assert!(config.enabled);
        assert_eq!(config.proxy_url, "http://127.0.0.1:7890");
    }
}
```

**Step 2: Run tests**

Run: `cd src-tauri && cargo test proxy 2>&1 | tail -15`
Expected: all pass

**Step 3: Commit**

```bash
git add src-tauri/src/proxy.rs
git commit -m "test(proxy): add unit tests for proxy module"
```

---

### Task 11: Final verification

**Step 1: Run full frontend type check**

Run: `npx tsc --noEmit`

**Step 2: Run full frontend test suite**

Run: `npx vitest run`

**Step 3: Run full Rust check**

Run: `cd src-tauri && cargo check && cargo test`

**Step 4: Fix any issues found**

**Step 5: Final commit if any fixes were needed**
