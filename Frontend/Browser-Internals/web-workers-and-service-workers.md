# Web Workers and Service Workers

JavaScript is single-threaded, but **Workers** allow running code in background threads. This note covers Web Workers
(parallel computation), Service Workers (network proxy / offline), and Shared Workers. A common interview topic that
tests understanding of the browser threading model.

## Worker Types Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      Browser                                 │
│                                                              │
│  Main Thread                                                 │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  DOM, UI rendering, event handlers, React, etc.       │  │
│  └────────┬──────────────────┬────────────────────────────┘  │
│           │ postMessage      │ postMessage                    │
│     ┌─────▼──────┐    ┌─────▼──────────┐                    │
│     │ Web Worker │    │ Service Worker  │                    │
│     │ (per page) │    │ (per origin)   │                    │
│     │            │    │                │                    │
│     │ Heavy      │    │ Network proxy  │                    │
│     │ computation│    │ Offline cache  │                    │
│     │ Data proc. │    │ Push notif.    │                    │
│     └────────────┘    └────────────────┘                    │
└─────────────────────────────────────────────────────────────┘
```

| Feature               | Web Worker              | Service Worker            | Shared Worker           |
|------------------------|-------------------------|---------------------------|-------------------------|
| **Purpose**            | Parallel computation    | Network proxy, offline    | Shared state across tabs|
| **Lifecycle**          | Lives while page is open| Independent of any page   | Lives while any tab uses it |
| **DOM access**         | ❌ No                   | ❌ No                     | ❌ No                   |
| **Scope**              | One page                | Entire origin (all tabs)  | Multiple tabs/iframes   |
| **Communication**      | `postMessage`           | `postMessage` + `fetch` events | `port.postMessage`  |
| **HTTPS required**     | No (localhost OK)       | ✅ Yes (except localhost) | No                      |
| **Can intercept fetch**| ❌ No                   | ✅ Yes                    | ❌ No                   |
| **Persistent**         | ❌ No (dies with page)  | ✅ Yes (survives tab close)| ❌ No                  |

## Web Workers

### Dedicated Worker (Most Common)

A background thread tied to a **single page**. Use for CPU-intensive tasks that would block the main thread.

#### Basic Usage

```javascript
// main.js — main thread
const worker = new Worker("worker.js");

// Send data to worker
worker.postMessage({ type: "SORT", data: hugeArray });

// Receive result from worker
worker.onmessage = (event) => {
    console.log("Sorted:", event.data);
};

// Handle errors
worker.onerror = (error) => {
    console.error("Worker error:", error.message);
};

// Terminate when done
worker.terminate();
```

```javascript
// worker.js — runs in background thread
self.onmessage = (event) => {
    const { type, data } = event.data;

    if (type === "SORT") {
        const sorted = data.sort((a, b) => a - b); // heavy work — doesn't block UI
        self.postMessage(sorted);
    }
};
```

#### Inline Worker (Blob URL)

Useful for bundlers (Webpack, Vite) — no separate file needed:

```javascript
const workerCode = `
    self.onmessage = (e) => {
        const result = heavyComputation(e.data);
        self.postMessage(result);
    };
`;

const blob = new Blob([workerCode], { type: "application/javascript" });
const worker = new Worker(URL.createObjectURL(blob));
```

#### Module Workers (ES Modules)

```javascript
const worker = new Worker("worker.js", { type: "module" });
```

```javascript
// worker.js — can use import/export
import { processData } from "./utils.js";

self.onmessage = (event) => {
    self.postMessage(processData(event.data));
};
```

### What Workers CAN and CANNOT Access

| ✅ Available                        | ❌ Not Available              |
|--------------------------------------|-------------------------------|
| `fetch`, `XMLHttpRequest`            | `document`, `window`          |
| `setTimeout`, `setInterval`          | DOM (`getElementById`, etc.)  |
| `IndexedDB`                          | `localStorage`, `sessionStorage` |
| `WebSocket`                          | `alert`, `confirm`, `prompt`  |
| `crypto`, `TextEncoder/Decoder`      | `parent`, `opener`            |
| `navigator` (partial)               | Any UI / rendering API        |
| `structuredClone`, `postMessage`     |                               |

### Transferable Objects

`postMessage` normally **copies** data (structured clone). For large data (ArrayBuffers, ImageBitmap), use
**Transferable** — ownership is transferred (zero-copy), but the source loses access:

```javascript
// Main thread
const buffer = new ArrayBuffer(1024 * 1024 * 100); // 100 MB
worker.postMessage(buffer, [buffer]); // second arg = transferable list

console.log(buffer.byteLength); // 0 — ownership transferred, buffer is now empty

// Worker
self.onmessage = (event) => {
    const buffer = event.data; // received without copying
    // process buffer...
    self.postMessage(buffer, [buffer]); // transfer back
};
```

**Transferable types:** `ArrayBuffer`, `MessagePort`, `ImageBitmap`, `OffscreenCanvas`, `ReadableStream`,
`WritableStream`, `TransformStream`.

### Use Cases for Web Workers

- **Sorting / filtering large datasets** — e.g., 100K rows in a table.
- **Image / video processing** — pixel manipulation, resizing, filters.
- **Parsing** — large JSON, CSV, XML files.
- **Cryptography** — hashing, encryption.
- **WebAssembly** — run WASM in a worker to avoid blocking UI.
- **Syntax highlighting** — large code blocks (Monaco editor does this).

### Web Workers in React

```tsx
// useWorker.ts — custom hook
import { useEffect, useRef, useCallback } from "react";

function useWorker<TInput, TOutput>(workerFactory: () => Worker) {
    const workerRef = useRef<Worker | null>(null);

    useEffect(() => {
        workerRef.current = workerFactory();
        return () => workerRef.current?.terminate();
    }, []);

    const postMessage = useCallback((data: TInput): Promise<TOutput> => {
        return new Promise((resolve, reject) => {
            if (!workerRef.current) return reject("Worker not initialized");
            workerRef.current.onmessage = (e) => resolve(e.data);
            workerRef.current.onerror = (e) => reject(e.message);
            workerRef.current.postMessage(data);
        });
    }, []);

    return { postMessage };
}

// Usage
function DataTable({ rawData }: { rawData: Item[] }) {
    const { postMessage } = useWorker<Item[], Item[]>(
        () => new Worker(new URL("./sort-worker.ts", import.meta.url))
    );

    const handleSort = async () => {
        const sorted = await postMessage(rawData); // doesn't block UI
        setSortedData(sorted);
    };
}
```

### Comlink — Simplified Worker API

[Comlink](https://github.com/GoogleChromeLabs/comlink) wraps `postMessage` with a Proxy-based RPC:

```javascript
// worker.js
import * as Comlink from "comlink";

const api = {
    heavySort(data) { return data.sort((a, b) => a - b); },
    fibonacci(n) { /* ... */ },
};

Comlink.expose(api);

// main.js
import * as Comlink from "comlink";

const worker = new Worker("worker.js");
const api = Comlink.wrap(worker);

const sorted = await api.heavySort(hugeArray); // looks like a regular async call
```

## Service Workers

A **programmable network proxy** that sits between the browser and the network. Enables offline support, caching
strategies, push notifications, and background sync.

### Lifecycle

```
     install ──▶ waiting ──▶ activate ──▶ running (idle / handling events)
       │                        │              │
  Cache assets           Clean old caches    Intercept fetch, push, sync
       │                        │
  installEvent.           activateEvent.
  waitUntil(...)          waitUntil(...)
```

1. **Register** — page tells the browser about the SW file.
2. **Install** — SW downloads and caches static assets. Fires `install` event.
3. **Wait** — if an old SW is active, the new one waits until all its tabs are closed (unless `skipWaiting()`).
4. **Activate** — old SW is gone, new one takes over. Clean up old caches here.
5. **Fetch** — SW intercepts network requests and can serve from cache, network, or both.

### Registration

```javascript
// main.js
if ("serviceWorker" in navigator) {
    window.addEventListener("load", () => {
        navigator.serviceWorker.register("/sw.js", { scope: "/" })
            .then(reg => console.log("SW registered:", reg.scope))
            .catch(err => console.error("SW registration failed:", err));
    });
}
```

### Install — Pre-Caching

```javascript
// sw.js
const CACHE_NAME = "app-v1";
const STATIC_ASSETS = [
    "/",
    "/index.html",
    "/styles.css",
    "/app.js",
    "/offline.html",
];

self.addEventListener("install", (event) => {
    event.waitUntil(
        caches.open(CACHE_NAME)
            .then(cache => cache.addAll(STATIC_ASSETS))
            .then(() => self.skipWaiting()) // activate immediately, don't wait
    );
});
```

### Activate — Clean Old Caches

```javascript
self.addEventListener("activate", (event) => {
    event.waitUntil(
        caches.keys().then(keys =>
            Promise.all(
                keys
                    .filter(key => key !== CACHE_NAME)
                    .map(key => caches.delete(key))
            )
        ).then(() => self.clients.claim()) // take control of all open tabs immediately
    );
});
```

### Fetch — Caching Strategies

#### 1. Cache First (Offline First)

Best for static assets (images, fonts, CSS, JS bundles):

```javascript
self.addEventListener("fetch", (event) => {
    event.respondWith(
        caches.match(event.request)
            .then(cached => cached || fetch(event.request))
    );
});
```

```
Request ──▶ Cache hit? ──yes──▶ Return cached
                │
               no
                │
                ▼
           Fetch from network ──▶ Return response
```

#### 2. Network First

Best for API calls, dynamic content:

```javascript
self.addEventListener("fetch", (event) => {
    if (event.request.url.includes("/api/")) {
        event.respondWith(
            fetch(event.request)
                .then(response => {
                    const clone = response.clone();
                    caches.open("api-cache").then(cache => cache.put(event.request, clone));
                    return response;
                })
                .catch(() => caches.match(event.request)) // offline fallback
        );
    }
});
```

```
Request ──▶ Fetch from network ──success──▶ Cache + Return
                │
              failure
                │
                ▼
           Return cached (stale) or offline page
```

#### 3. Stale While Revalidate

Return cached immediately, update cache in background. Best for content that can be slightly stale (avatars, feeds):

```javascript
self.addEventListener("fetch", (event) => {
    event.respondWith(
        caches.open("swr-cache").then(cache =>
            cache.match(event.request).then(cached => {
                const networkFetch = fetch(event.request).then(response => {
                    cache.put(event.request, response.clone());
                    return response;
                });
                return cached || networkFetch;
            })
        )
    );
});
```

```
Request ──▶ Return cached (fast!) + Fetch in background ──▶ Update cache for next time
```

#### Strategy Comparison

| Strategy              | Speed    | Freshness | Offline | Best for                    |
|-----------------------|----------|-----------|---------|------------------------------|
| Cache First           | Fastest  | Stale     | ✅      | Static assets, fonts         |
| Network First         | Slower   | Fresh     | ✅ (fallback) | API calls, dynamic data  |
| Stale While Revalidate| Fast     | Eventually| ✅      | Feeds, avatars, semi-static  |
| Network Only          | Depends  | Always    | ❌      | Analytics, non-cacheable     |
| Cache Only            | Fastest  | Stale     | ✅      | Pre-cached app shell         |

### Push Notifications

```javascript
// sw.js
self.addEventListener("push", (event) => {
    const data = event.data?.json() ?? { title: "Notification", body: "" };

    event.waitUntil(
        self.registration.showNotification(data.title, {
            body: data.body,
            icon: "/icon-192.png",
            badge: "/badge.png",
            data: { url: data.url },
        })
    );
});

self.addEventListener("notificationclick", (event) => {
    event.notification.close();
    event.waitUntil(
        clients.openWindow(event.notification.data.url)
    );
});
```

### Background Sync

Defer actions until the user has connectivity:

```javascript
// main.js — queue a sync
navigator.serviceWorker.ready.then(reg => {
    reg.sync.register("send-messages");
});

// sw.js — execute when back online
self.addEventListener("sync", (event) => {
    if (event.tag === "send-messages") {
        event.waitUntil(sendQueuedMessages());
    }
});
```

## Shared Workers

A single worker instance shared by **multiple tabs/iframes** of the same origin. Useful for shared WebSocket
connections or cross-tab state.

```javascript
// shared-worker.js
const connections = [];

self.onconnect = (event) => {
    const port = event.ports[0];
    connections.push(port);

    port.onmessage = (e) => {
        // Broadcast to all connected tabs
        connections.forEach(p => p.postMessage(e.data));
    };
};

// main.js (any tab)
const worker = new SharedWorker("shared-worker.js");
worker.port.start();

worker.port.postMessage("Hello from tab");
worker.port.onmessage = (event) => {
    console.log("Received:", event.data);
};
```

**Use cases:**
- Shared WebSocket connection (one WS for all tabs instead of N).
- Cross-tab state synchronization.
- Shared computation cache.

**Note:** Limited browser support compared to dedicated workers. For cross-tab communication, `BroadcastChannel` or
`localStorage` `storage` event are simpler alternatives.

## Worklets (Brief)

Lightweight, special-purpose workers for rendering pipelines:

| Worklet         | Purpose                                  | API                  |
|-----------------|------------------------------------------|----------------------|
| **Paint**       | Custom CSS painting                      | CSS Houdini          |
| **Animation**   | Off-main-thread animations               | `AnimationWorklet`   |
| **Audio**       | Custom audio processing                  | Web Audio API        |
| **Layout**      | Custom CSS layout algorithms             | CSS Houdini          |

Not commonly asked in interviews, but good to know they exist.

## `OffscreenCanvas`

Allows rendering canvas in a Web Worker — useful for heavy graphics/charts without blocking UI:

```javascript
// main.js
const canvas = document.getElementById("chart");
const offscreen = canvas.transferControlToOffscreen();

const worker = new Worker("render-worker.js");
worker.postMessage({ canvas: offscreen }, [offscreen]); // transfer ownership

// render-worker.js
self.onmessage = (event) => {
    const canvas = event.data.canvas;
    const ctx = canvas.getContext("2d");
    // Draw complex chart — doesn't block main thread
    ctx.fillRect(0, 0, 100, 100);
};
```

## Performance: When to Use Workers

| Symptom                                      | Solution                                 |
|----------------------------------------------|------------------------------------------|
| Long task blocking UI (> 50ms)               | Move to Web Worker                       |
| App needs offline support                    | Service Worker + caching strategies      |
| Heavy JSON parsing / data transformation     | Web Worker                               |
| Network request caching / proxy              | Service Worker                           |
| Push notifications                           | Service Worker                           |
| WebSocket shared across tabs                 | Shared Worker                            |
| Canvas rendering blocking UI                 | Web Worker + OffscreenCanvas             |

**Rule of thumb:** If a task takes > 50ms and doesn't need DOM access, consider a Web Worker.

## Common Interview Questions

1. **What is a Web Worker?** — A background thread that runs JavaScript in parallel with the main thread. Cannot
   access DOM. Communicates with the main thread via `postMessage`. Use for CPU-intensive tasks to avoid blocking UI.
2. **What is a Service Worker?** — A programmable network proxy that intercepts fetch requests, enables offline
   caching, push notifications, and background sync. Runs independently of any page, requires HTTPS.
3. **Web Worker vs Service Worker?** — Web Worker: parallel computation, tied to one page, no network interception.
   Service Worker: network proxy, independent of pages, enables offline/PWA features.
4. **What caching strategies does a Service Worker support?** — Cache First (fastest, may be stale), Network First
   (fresh, offline fallback), Stale While Revalidate (fast + background refresh), Cache/Network Only.
5. **How does `postMessage` work? What about large data?** — `postMessage` copies data via structured clone. For
   large `ArrayBuffer`s, use Transferable objects (zero-copy transfer, source loses access).
6. **What is the Service Worker lifecycle?** — Register → Install (pre-cache) → Wait (if old SW active) → Activate
   (clean old caches) → Running (intercept fetch). `skipWaiting()` skips the wait phase.
7. **Can a Web Worker access the DOM?** — No. Workers have no access to `document`, `window`, or any DOM API. They
   can use `fetch`, `IndexedDB`, `WebSocket`, timers, and `postMessage`.

## Related

- [Event Loop](../JavaScript/event-loop.md) — main thread execution model that workers offload
- [Web Storage and Cookies](./web-storage-and-cookies.md) — workers can access `IndexedDB` but not `localStorage`
- [Performance Optimization](../React/performance/performance-optimization.md) — React-specific optimizations

## Resources

- [MDN — Web Workers API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API)
- [MDN — Service Worker API](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API)
- [web.dev — Service Workers](https://web.dev/learn/pwa/service-workers)
- [Google — Workbox](https://developer.chrome.com/docs/workbox/) — production-ready SW toolkit
- [Comlink](https://github.com/GoogleChromeLabs/comlink) — simplified Worker communication
