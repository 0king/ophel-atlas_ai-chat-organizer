# Implementation Plan: Harden Userscript for Chat Outline & Export Only

## Goal Description
Modify [ophel-atlas_ai-chat-organizer.js](file:///home/dj/Desktop/projects/userscripts.js/ophel-atlas_ai-chat-organizer.js) so that:
1. It is strictly scoped to the 10 specified AI chat websites (removing universal `@match http://*/*` and `https://*/*`).
2. All unneeded and sensitive permissions are removed (`GM_cookie`, `GM_xmlhttpRequest`, `GM_notification`, `window.focus`, `unsafeWindow`, and `@connect *`).
3. Outbound network capabilities across **all** protocols and side-channels (`fetch`, `XMLHttpRequest`, `WebSocket`, `WebRTC`, `navigator.sendBeacon`, `EventSource`, `WebTransport`, `Worker`, DNS prefetching) are audited and proactively blocked at runtime within the userscript scope, while preserving local in-memory `blob:`/`data:` operations necessary for exporting files.
4. Auto-update URLs (`@updateURL`, `@downloadURL`) are removed to prevent Tampermonkey from restoring original permissions.
5. The **Chat Outline** (DOM-based tree parsing, heading navigation, search, bookmarking) and **Conversation Export** (DOM message collection, Markdown/JSON formatting, browser file download via `createObjectURL`) remain completely functional.

---

## Protocol & Exfiltration Threat Analysis

### Can a Userscript Use Protocols Besides HTTP?
**Yes.** Modern browsers expose several communication channels and transport layers beyond standard HTTP/HTTPS `fetch`:

| Vector / Protocol | Transport Layer | How It Works / Exfiltration Risk | Status in This Script |
| :--- | :--- | :--- | :--- |
| **Dynamic `<script>` Injection** | DOM / HTTP(S) | `document.createElement("script")` appended to `document.head` to load external scripts directly into the page context. | **Found in Mermaid runner** (index 1583278) |
| **Dynamic `<img>` / `<link>` Exfil** | DOM / HTTP(S) | Creates `<img>` or `<link>` with external URLs carrying stolen data in query parameters. | **Found in Vite preload** (index 16461) |
| **WebRTC (`RTCPeerConnection`)** | UDP/TCP (ICE/STUN/TURN) | Bypasses HTTP proxies and CSPs; connects directly to arbitrary IP/ports; can leak local/public IPs. | **Clean** (0 occurrences) |
| **DNS Query Prefetching** | UDP 53 (DNS) | Injects `<link rel="dns-prefetch" href="//data.attacker.com">`. Resolves domain via OS DNS even if HTTP ports are blocked. | **Clean** (0 occurrences) |
| **`navigator.sendBeacon`** | HTTP POST | Asynchronous telemetry POST; survives tab closure; doesn't wait for response. | **Clean** (0 occurrences) |
| **`WebSocket`** | WS / WSS (TCP) | Full-duplex bidirectional socket communication. | **Clean** (0 occurrences) |
| **`WebTransport`** | HTTP/3 (QUIC / UDP) | Low-latency bidirectional transport protocol. | **Clean** (0 occurrences) |
| **`EventSource` (SSE)** | HTTP stream | Persistent server-sent events stream. | **Clean** (0 occurrences) |
| **`Worker` / `SharedWorker`** | Background Thread | Spawns separate execution threads with independent network capabilities. | **Clean** (0 occurrences) |
| **`XMLHttpRequest`** | HTTP/HTTPS | Standard AJAX request mechanism. | Present as `GM_xmlhttpRequest` & passive observer |
| **OS Scheme Handlers** | IPC | Custom URIs (`slack://`, `mailto:`, etc.). | **Clean** (0 occurrences) |

> [!WARNING]
> **Critical Finding: Dynamic Script Injection Found in Script**:
> Searching for `document.createElement("script")` revealed an active dynamic script injection routine at index 1583278 (`injectScript(e, t)`):
> ```javascript
> injectScript(e, t) {
>   const n = `script[data-ophel-runtime="${t}"]`, r = document.querySelector(n);
>   const a = r || document.createElement("script");
>   // ...
>   (document.head || document.documentElement).appendChild(a);
> }
> ```
> This function was created to dynamically inject the Assistant Mermaid diagram runner from an unpinned remote GitHub URL (`https://cdn.jsdelivr.net/gh/urzeye/ophel@main/assets/assistant-mermaid-runner.js`).
>
> **Countermeasures:**
> 1. Remove the unpinned Mermaid resource from the metadata header.
> 2. Neuter the `injectScript` method to a no-op so it never attempts to load external scripts.
> 3. Enforce a hard block on `document.createElement("script")` and `Node.prototype.appendChild(script)` in our Zero-Trust Sandbox.

---

## User Review Required

> [!IMPORTANT]
> **Behavioral Changes After Modification**:
> 1. **WebDAV Sync & Cloud Features Disabled**: Syncing chat history to WebDAV or external servers will fail cleanly (intercepted by the sandbox).
> 2. **Auto-Updates Disabled**: Tampermonkey will not auto-update this script. This is required so GreasyFork doesn't overwrite our security restrictions with the upstream script.
> 3. **Image Asset Inclusion in ZIP Exports**: If a conversation contains remote images hosted on external servers, the exporter normally tries to fetch them to include them in the `.zip`. With outbound network disabled, text/Markdown/JSON exports will export markdown references without phoning home to download image blobs. Local `blob:` images (generated in-browser) will still export normally.

---

## Proposed Changes

### 1. Userscript Header & Metadata Configuration

#### [MODIFY] [ophel-atlas_ai-chat-organizer.js](file:///home/dj/Desktop/projects/userscripts.js/ophel-atlas_ai-chat-organizer.js#L35-L103)

1. **Replace Universal Match with 10 Explicit AI Domains**:
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

### 2. Multi-Protocol Zero-Trust Runtime Sandbox

#### [MODIFY] [ophel-atlas_ai-chat-organizer.js](file:///home/dj/Desktop/projects/userscripts.js/ophel-atlas_ai-chat-organizer.js#L104-L120)

Immediately following the metadata block, insert a comprehensive Multi-Protocol Network Guard that blocks HTTP, WebRTC, Sockets, Beacons, Workers, and DNS side-channels within the script context:

```javascript
/* =========================================================================
 * MULTI-PROTOCOL ZERO-TRUST NETWORK GUARD (OUTLINE & EXPORT PRIVACY SANDBOX)
 * Proactively blocks all outbound communication vectors:
 * - HTTP/HTTPS (fetch, XMLHttpRequest, GM_xmlhttpRequest)
 * - WebSockets (WebSocket)
 * - WebRTC (RTCPeerConnection, RTCDataChannel)
 * - Telemetry & Side-Channels (sendBeacon, EventSource, WebTransport)
 * - Background Execution (Worker, SharedWorker)
 * - DNS Prefetching / Link Exfiltration (dns-prefetch, preconnect, ping)
 * Permits ONLY in-memory local blob: and data: URLs required for file export.
 * ========================================================================= */
(function() {
  const _blocked = (api, target) => {
    console.warn(`[Ophel Security Sandbox] Blocked outbound ${api}${target ? `: ${target}` : ""}`);
    throw new Error(`[Ophel Security Sandbox] Network disabled for ${api}`);
  };

  // 1. HTTP Fetch Guard
  const _nativeFetch = window.fetch.bind(window);
  window.fetch = async function(input, init) {
    const url = typeof input === "string" ? input : (input?.url || "");
    if (url.startsWith("blob:") || url.startsWith("data:")) {
      return _nativeFetch(input, init);
    }
    _blocked("fetch", url);
  };

  // 2. XMLHttpRequest Guard
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

  // 3. WebSocket Guard
  window.WebSocket = class {
    constructor(url) {
      _blocked("WebSocket", url);
    }
  };

  // 4. WebRTC Guard (Blocks peer connections & STUN/TURN traffic)
  window.RTCPeerConnection = class {
    constructor() { _blocked("RTCPeerConnection"); }
  };
  window.webkitRTCPeerConnection = window.RTCPeerConnection;
  if ("RTCDataChannel" in window) {
    window.RTCDataChannel = class {
      constructor() { _blocked("RTCDataChannel"); }
    };
  }

  // 5. Telemetry & Streaming Guards
  if (navigator.sendBeacon) {
    navigator.sendBeacon = function(url) {
      _blocked("navigator.sendBeacon", url);
      return false;
    };
  }
  if ("EventSource" in window) {
    window.EventSource = class {
      constructor(url) { _blocked("EventSource", url); }
    };
  }
  if ("WebTransport" in window) {
    window.WebTransport = class {
      constructor(url) { _blocked("WebTransport", url); }
    };
  }

  // 6. Background Worker Guard
  window.Worker = class {
    constructor() { _blocked("Worker"); }
  };
  window.SharedWorker = class {
    constructor() { _blocked("SharedWorker"); }
  };

  // 7. Dynamic Script & DOM Exfiltration Guard
  const _origCreateElement = document.createElement.bind(document);
  document.createElement = function(tagName, options) {
    const tag = String(tagName).toLowerCase();

    // HARD BLOCK: Dynamic script creation (prevents remote script loading)
    if (tag === "script") {
      _blocked("document.createElement('script')");
    }

    // HARD BLOCK: Frames and object embeds
    if (tag === "iframe" || tag === "object" || tag === "embed") {
      _blocked(`document.createElement('${tag}')`);
    }

    const el = _origCreateElement(tagName, options);

    // Block image/audio/video phone-home telemetry
    if (tag === "img" || tag === "audio" || tag === "video") {
      const origSetAttr = el.setAttribute.bind(el);
      el.setAttribute = function(name, val) {
        if (name === "src") {
          const str = String(val);
          if (!str.startsWith("blob:") && !str.startsWith("data:") && !str.startsWith("/")) {
            _blocked(`<${tag} src="${str}">`);
          }
        }
        return origSetAttr(name, val);
      };
    }

    // Block link prefetch & external resource requests
    if (tag === "link") {
      const origSetAttr = el.setAttribute.bind(el);
      el.setAttribute = function(name, val) {
        if (name === "rel" && /^(dns-prefetch|preconnect|prefetch|prerender)$/i.test(val)) {
          _blocked("DNS/Resource Prefetch", val);
        }
        return origSetAttr(name, val);
      };
    }

    if (tag === "a") {
      el.removeAttribute("ping");
    }

    return el;
  };

  // 8. Prevent appending any script elements to the DOM
  const _origAppendChild = Node.prototype.appendChild;
  Node.prototype.appendChild = function(child) {
    if (child instanceof HTMLScriptElement) {
      _blocked("Node.appendChild(<script>)");
    }
    return _origAppendChild.call(this, child);
  };

  // 9. Stub GM_xmlhttpRequest
  globalThis.GM_xmlhttpRequest = function(opts) {
    _blocked("GM_xmlhttpRequest", opts?.url || "");
    opts?.onerror?.(new Error("GM_xmlhttpRequest is disabled"));
    return { abort() {} };
  };
})();
```

---

## Verification Plan

### Automated Verification
1. **Metadata Header Validation**:
   Check that `@match` lists only the 10 sites and sensitive grants/URLs are completely eliminated:
   ```bash
   grep -E "(@connect|GM_cookie|GM_xmlhttpRequest|unsafeWindow|updateURL|downloadURL)" ophel-atlas_ai-chat-organizer.js
   ```
2. **Syntax and Bundle Integrity Check**:
   Verify syntax validity:
   ```bash
   node -c ophel-atlas_ai-chat-organizer.js
   ```
3. **Multi-Protocol Sandbox Smoke Test**:
   Execute the sandbox in a test harness verifying that every channel triggers a block:
   - `fetch("https://...")` -> Throws
   - `fetch("blob:...")` -> Allowed
   - `new WebSocket(...)` -> Throws
   - `new RTCPeerConnection(...)` -> Throws
   - `navigator.sendBeacon(...)` -> Throws / returns false
   - `new EventSource(...)` -> Throws
   - `new WebTransport(...)` -> Throws
   - `new Worker(...)` -> Throws
   - `<link rel="dns-prefetch">` -> Throws / blocked

### Manual Verification Instructions
1. Install the modified script in Tampermonkey.
2. Confirm the script is only active on the 10 specified AI domains.
3. Verify on `chatgpt.com` or `gemini.google.com`:
   - Outline loads and tracks headings.
   - Conversation exports to Markdown and JSON without network errors.

