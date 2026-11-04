# Implementation Plan: Harden Userscript for Chat Outline & Export Only

## Goal Description
Modify [ophel-atlas_ai-chat-organizer.js](file:///home/dj/Desktop/projects/userscripts.js/ophel-atlas_ai-chat-organizer.js) so that:
1. It is strictly scoped to the 10 specified AI chat websites (removing universal `@match http://*/*` and `https://*/*`).
2. All unneeded and sensitive permissions are removed (`GM_cookie`, `GM_xmlhttpRequest`, `GM_notification`, `window.focus`, `unsafeWindow`, and `@connect *`).
3. Outbound network capabilities (`window.fetch`, `WebSocket`, `XMLHttpRequest`, `GM_xmlhttpRequest`) are blocked at runtime within the userscript scope, while preserving local in-memory `blob:`/`data:` operations necessary for exporting files.
4. Auto-update URLs (`@updateURL`, `@downloadURL`) are removed to prevent Tampermonkey from restoring original permissions.
5. The **Chat Outline** (DOM-based tree parsing, heading navigation, search, bookmarking) and **Conversation Export** (DOM message collection, Markdown/JSON formatting, browser file download via `createObjectURL`) remain completely functional.

---

## User Review Required

> [!IMPORTANT]
> **Behavioral Changes After Modification**:
> 1. **WebDAV Sync & Cloud Features Disabled**: Syncing chat history to WebDAV or external servers will fail cleanly (intercepted by the sandbox).
> 2. **Auto-Updates Disabled**: Tampermonkey will not auto-update this script. This is required so GreasyFork doesn't overwrite our security restrictions with the upstream script.
> 3. **Image Asset Inclusion in ZIP Exports**: If a conversation contains remote images hosted on external servers, the exporter normally tries to fetch them to include them in the `.zip`. With outbound network disabled, text/Markdown/JSON exports will export markdown references without phoning home to download image blobs. Local `blob:` images (generated in-browser) will still export normally.

---

## Proposed Changes

### Userscript Header & Metadata Configuration

#### [MODIFY] [ophel-atlas_ai-chat-organizer.js](file:///home/dj/Desktop/projects/userscripts.js/ophel-atlas_ai-chat-organizer.js#L35-L103)

1. **Replace Universal Match with 10 Explicit AI Domains**:
   Replace:
   ```javascript
   // @match              http://*/*
   // @match              https://*/*
   ```
   With:
   ```javascript
   // @match              https://gemini.google.com/*
   // @match              https://aistudio.google.com/*
   // @match              https://grok.com/*
   // @match              https://chat.openai.com/*
   // @match              https://chatgpt.com/*
   // @match              https://claude.ai/*
   // @match              https://chat.deepseek.com/*
   // @match              https://www.kimi.com/*
   // @match              https://chat.qwen.ai/*
   // @match              https://chat.z.ai/*
   ```

2. **Strip Sensitive Grants & Directives**:
   - **Remove**: `// @connect *`
   - **Remove**: `// @grant GM_cookie` (Prevents reading/writing cookies)
   - **Remove**: `// @grant GM_xmlhttpRequest` (Prevents background network requests)
   - **Remove**: `// @grant GM_notification` (Prevents desktop notifications)
   - **Remove**: `// @grant window.focus` (Prevents tab refocusing)
   - **Remove**: `// @grant unsafeWindow` (Runs the userscript in Tampermonkey's isolated world; host page JS scope is protected)
   - **Remove**: `// @downloadURL ...` and `// @updateURL ...` (Prevents Tampermonkey from pulling upstream updates and undoing security changes)
   - **Remove**: Unpinned mutable `@resource ophelAssistantMermaidRunner https://cdn.jsdelivr.net/gh/urzeye/ophel@main/assets/assistant-mermaid-runner.js`

3. **Retain Required Local Grants**:
   - `// @grant GM_addStyle` (Renders outline sidebar and export modal UI)
   - `// @grant GM_getResourceText` (Loads bundled markdown preview CSS and styles)
   - `// @grant GM_getResourceURL` (Loads local icons and assets)
   - `// @grant GM_getValue`, `// @grant GM_setValue`, `// @grant GM_deleteValue` (Persists outline level, UI state, and export format preferences locally)
   - `// @grant GM_addValueChangeListener`, `// @grant GM_removeValueChangeListener` (Multi-tab settings sync)
   - `// @grant GM_info` (Version inspection)

---

### In-Script Runtime Network Blocker

#### [MODIFY] [ophel-atlas_ai-chat-organizer.js](file:///home/dj/Desktop/projects/userscripts.js/ophel-atlas_ai-chat-organizer.js#L104-L120)

Immediately following the metadata block, before any application logic or vendor modules execute, install a scoped Network Guard:

```javascript
/* =========================================================================
 * ZERO-TRUST NETWORK GUARD (OUTLINE & EXPORT PRIVACY SANDBOX)
 * Blocks all outbound network requests (fetch, XHR, WebSocket, GM_xmlhttpRequest)
 * Permits only in-memory local blob: and data: URLs required for file export.
 * ========================================================================= */
(function() {
  const _blocked = (api, url) => {
    console.warn(`[Ophel Security Sandbox] Blocked outbound ${api} to: ${url}`);
    throw new Error(`[Ophel Security Sandbox] Network disabled for ${api}`);
  };

  // 1. Scoped fetch guard
  const _nativeFetch = window.fetch.bind(window);
  window.fetch = async function(input, init) {
    const url = typeof input === "string" ? input : (input?.url || "");
    if (url.startsWith("blob:") || url.startsWith("data:")) {
      return _nativeFetch(input, init);
    }
    _blocked("fetch", url);
  };

  // 2. Scoped WebSocket guard
  window.WebSocket = class {
    constructor(url) {
      _blocked("WebSocket", url);
    }
  };

  // 3. Scoped XMLHttpRequest guard
  const _NativeXHR = window.XMLHttpRequest;
  window.XMLHttpRequest = class extends _NativeXHR {
    open(method, url, ...args) {
      const urlStr = String(url);
      if (urlStr.startsWith("blob:") || urlStr.startsWith("data:")) {
        return super.open(method, url, ...args);
      }
      _blocked("XMLHttpRequest", urlStr);
    }
  };

  // 4. Stub GM_xmlhttpRequest in case any deferred reference exists
  if (typeof GM_xmlhttpRequest === "undefined" || true) {
    globalThis.GM_xmlhttpRequest = function(opts) {
      _blocked("GM_xmlhttpRequest", opts?.url || "");
      opts?.onerror?.(new Error("GM_xmlhttpRequest is disabled"));
      return { abort() {} };
    };
  }
})();
```

---

## Verification Plan

### Automated Verification
1. **Metadata Header Validation**:
   Run a syntax inspection script verifying:
   - Zero occurrences of `http://*/*`, `https://*/*`, `@connect *`, `GM_cookie`, `GM_xmlhttpRequest`, `unsafeWindow`, `@updateURL`, `@downloadURL`.
   - Exactly the 10 requested `@match` URLs.
2. **Syntax and Bundle Integrity Check**:
   Run Node.js to verify that the modified file has no JavaScript syntax errors:
   ```bash
   node -c ophel-atlas_ai-chat-organizer.js
   ```
3. **Sandbox Execution Smoke Test**:
   Execute the userscript preamble in a headless DOM context (JSDOM) simulating:
   - Attempting `fetch("https://example.com")` -> Must throw security error.
   - Attempting `fetch("blob:abc-123")` -> Must be permitted.
   - Attempting `new WebSocket("wss://echo.websocket.org")` -> Must throw security error.
   - Calling `GM_xmlhttpRequest({ url: "https://evil.com" })` -> Must throw security error.

### Manual Verification Instructions
1. Install or replace the script in Tampermonkey/Violentmonkey.
2. Verify that Tampermonkey lists the script as active **only** on the 10 designated sites, and shows no `@connect` or cookie permissions.
3. Open a supported site (e.g. `https://chatgpt.com` or `https://gemini.google.com`):
   - Confirm the Outline panel appears and indexes conversation headers and user queries.
   - Confirm clicking outline items jumps smoothly to that section of the chat.
   - Confirm clicking "Export" downloads the conversation as Markdown/JSON properly without any network errors.
