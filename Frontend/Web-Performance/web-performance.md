# Web Performance

Web performance directly impacts user experience, SEO rankings, and conversion rates. Interview questions cover Core
Web Vitals, loading strategies, rendering pipeline, and optimization techniques.

---

## Core Web Vitals

Google's key metrics for real-world user experience:

| Metric | Full Name                  | What it measures                       | Good     | Poor    |
|--------|----------------------------|----------------------------------------|----------|---------|
| LCP    | Largest Contentful Paint   | Loading — when largest element renders  | ≤ 2.5s   | > 4.0s  |
| INP    | Interaction to Next Paint  | Responsiveness — input-to-visual delay  | ≤ 200ms  | > 500ms |
| CLS    | Cumulative Layout Shift    | Visual stability — unexpected layout shifts | ≤ 0.1 | > 0.25  |

### LCP (Largest Contentful Paint)

Measures when the **largest visible element** (image, video poster, large text block, background image) finishes
rendering in the viewport.

**Common LCP elements:** hero images, banner videos, large heading text blocks.

**How to improve:**
- Eliminate render-blocking resources (CSS, sync JS).
- Preload the LCP image: `<link rel="preload" as="image" href="hero.webp">`.
- Use `fetchpriority="high"` on the LCP `<img>`.
- Inline critical CSS, defer non-critical CSS.
- Use a CDN for static assets.
- Optimize server response time (TTFB).
- Avoid lazy-loading the LCP image.

### INP (Interaction to Next Paint)

Replaced FID in March 2024. Measures the delay from user input (click, tap, keypress) to the next paint — across
**all interactions** during the page lifecycle (worst case, with outliers excluded).

**How to improve:**
- Break long tasks (> 50ms) into smaller chunks with `requestIdleCallback`, `scheduler.yield()`, or `setTimeout(0)`.
- Reduce main-thread JavaScript — defer, lazy-load, or move to Web Workers.
- Minimize DOM size (< 1400 nodes recommended).
- Avoid forced synchronous layouts (read layout → write DOM → read layout).
- Use `content-visibility: auto` for off-screen content.

### CLS (Cumulative Layout Shift)

Sum of all unexpected layout shifts that occur during the page's lifetime (windowed to the worst 5-second session).

**How to improve:**
- Always set `width` and `height` on `<img>` and `<video>` (or use `aspect-ratio`).
- Reserve space for ads, embeds, and dynamic content.
- Avoid inserting content above existing content.
- Use `font-display: swap` with preloaded fonts to reduce FOIT (Flash of Invisible Text).
- Prefer `transform` animations over layout-triggering properties (`top`, `left`, `width`, `height`).

---

## Other Important Metrics

| Metric | Full Name                   | What it measures                              |
|--------|-----------------------------|-----------------------------------------------|
| TTFB   | Time to First Byte          | Server responsiveness                         |
| FCP    | First Contentful Paint      | When the first content appears on screen      |
| TTI    | Time to Interactive         | When the page is fully interactive (deprecated)|
| TBT    | Total Blocking Time         | Sum of long-task blocking time between FCP and TTI |
| SI     | Speed Index                 | How quickly content is visually populated     |

---

## Critical Rendering Path

The sequence the browser follows to convert HTML, CSS, and JS into pixels:

```
HTML → DOM Tree
                 → Render Tree → Layout → Paint → Composite
CSS  → CSSOM
```

1. **Parse HTML** → build DOM tree.
2. **Parse CSS** → build CSSOM (CSS Object Model).
3. **Combine** DOM + CSSOM → **Render Tree** (only visible nodes).
4. **Layout** (Reflow) — calculate position and size of each node.
5. **Paint** — fill in pixels (text, colors, borders, shadows, images).
6. **Composite** — combine painted layers into the final image (GPU-accelerated).

### Render-Blocking Resources

- **CSS** is render-blocking by default — the browser won't paint until CSSOM is complete.
- **Synchronous JS** (`<script>` without `async`/`defer`) blocks HTML parsing.

```html
<!-- render-blocking -->
<link rel="stylesheet" href="styles.css">
<script src="app.js"></script>

<!-- non-blocking -->
<link rel="stylesheet" href="print.css" media="print">
<script src="app.js" defer></script>
<script src="analytics.js" async></script>
```

### `async` vs `defer`

| Attribute | Download      | Execute                                | Order preserved? |
|-----------|---------------|----------------------------------------|------------------|
| (none)    | Blocks parsing | Immediately, blocks parsing           | Yes              |
| `async`   | Parallel      | As soon as downloaded, blocks parsing  | No               |
| `defer`   | Parallel      | After HTML parsing, before DOMContentLoaded | Yes          |

**Rule of thumb:** use `defer` for scripts that need the DOM. Use `async` for independent scripts (analytics, ads).

---

## Resource Loading Optimization

### Resource Hints

```html
<!-- DNS lookup ahead of time -->
<link rel="dns-prefetch" href="https://api.example.com">

<!-- DNS + TCP + TLS ahead of time -->
<link rel="preconnect" href="https://fonts.googleapis.com">

<!-- Fetch resource needed soon (current page) -->
<link rel="preload" as="font" href="/fonts/Inter.woff2" type="font/woff2" crossorigin>

<!-- Fetch resources for likely next navigation -->
<link rel="prefetch" href="/next-page.js">

<!-- Prerender entire page (Speculation Rules API is the modern replacement) -->
<link rel="modulepreload" href="/module.js">
```

| Hint           | When to use                                        | Priority    |
|----------------|----------------------------------------------------|-------------|
| `dns-prefetch` | Third-party domains you'll connect to later        | Low         |
| `preconnect`   | Critical third-party origins (fonts, API)          | Medium      |
| `preload`      | Resources needed in the current page (LCP image, critical font) | High |
| `prefetch`     | Resources for the next likely navigation           | Low / Idle  |
| `modulepreload`| ES modules needed soon                             | High        |

### `fetchpriority`

Hint the browser about relative priority of a fetch:

```html
<img src="hero.webp" fetchpriority="high" />       <!-- LCP image -->
<img src="carousel-3.webp" fetchpriority="low" />  <!-- below the fold -->
<link rel="preload" as="font" href="body.woff2" fetchpriority="high" />
```

---

## Image Optimization

### Format Selection

| Format | Best for                           | Transparency | Animation | Compression |
|--------|------------------------------------|--------------|-----------|-------------|
| WebP   | Photos and graphics (universal)    | Yes          | Yes       | 25-35% smaller than JPEG |
| AVIF   | Photos (best compression)          | Yes          | Yes       | 50% smaller than JPEG    |
| SVG    | Icons, logos, illustrations        | Yes          | Yes (CSS/SMIL) | Vector (infinitely scalable) |
| PNG    | Graphics with transparency (fallback) | Yes       | No        | Lossless    |
| JPEG   | Photos (fallback)                  | No           | No        | Lossy       |

### Responsive Images

```html
<!-- srcset with width descriptors -->
<img src="photo-800.jpg"
     srcset="photo-400.jpg 400w,
             photo-800.jpg 800w,
             photo-1200.jpg 1200w"
     sizes="(max-width: 600px) 100vw,
            (max-width: 1200px) 50vw,
            33vw"
     alt="Description" />

<!-- art direction with <picture> -->
<picture>
    <source media="(min-width: 800px)" srcset="hero-wide.avif" type="image/avif" />
    <source media="(min-width: 800px)" srcset="hero-wide.webp" type="image/webp" />
    <img src="hero-narrow.jpg" alt="Hero" />
</picture>
```

### Lazy Loading

```html
<!-- native lazy loading (below-the-fold images) -->
<img src="photo.jpg" loading="lazy" alt="..." />

<!-- NEVER lazy-load the LCP image -->
<img src="hero.jpg" loading="eager" fetchpriority="high" alt="Hero" />
```

`loading="lazy"` defers loading until the image is near the viewport (browser-defined threshold).

---

## JavaScript Performance

### Bundle Optimization

- **Code splitting** — split by route, by component, or by vendor. Load only what the current page needs.
- **Tree shaking** — eliminate unused exports (works with ES modules, not CommonJS).
- **Dynamic imports** — `import('./module.js')` loads on demand.
- **Minification** — remove whitespace, shorten variables (Terser, esbuild, SWC).
- **Compression** — Brotli (preferred) or Gzip at the server/CDN level.

```javascript
// route-level code splitting in React
const Dashboard = React.lazy(() => import('./Dashboard'));

function App() {
    return (
        <Suspense fallback={<Spinner />}>
            <Dashboard />
        </Suspense>
    );
}
```

### Long Tasks

A **long task** is any task on the main thread that takes > 50ms. Long tasks block user interactions.

**Breaking up long tasks:**

```javascript
// yield to the main thread
function yieldToMain() {
    return new Promise(resolve => setTimeout(resolve, 0));
}

async function processItems(items) {
    for (const item of items) {
        processItem(item);
        await yieldToMain(); // let browser handle pending events
    }
}

// scheduler.yield() — modern API (Chrome 129+)
async function processItems(items) {
    for (const item of items) {
        processItem(item);
        await scheduler.yield();
    }
}
```

### Memory Leaks

Common sources:
- **Detached DOM nodes** — references to removed elements.
- **Forgotten event listeners** — listeners not cleaned up on unmount.
- **Closures** — capturing large objects in long-lived closures.
- **Global variables** — unintentional globals via missing `let`/`const`.
- **Timers** — `setInterval` / `setTimeout` not cleared.

Debug with DevTools → Memory → Heap Snapshot (compare snapshots to find leaks).

---

## CSS Performance

### Avoid Layout Thrashing

Reading layout properties (e.g., `offsetHeight`, `getBoundingClientRect()`) after DOM writes forces the browser to
recalculate layout synchronously.

```javascript
// BAD — forces layout on every iteration
elements.forEach(el => {
    const height = el.offsetHeight; // read → forces layout
    el.style.height = height + 10 + 'px'; // write
});

// GOOD — batch reads, then batch writes
const heights = elements.map(el => el.offsetHeight); // all reads
elements.forEach((el, i) => {
    el.style.height = heights[i] + 10 + 'px'; // all writes
});
```

### Properties and Performance Cost

| Cost level     | Properties                                              |
|----------------|---------------------------------------------------------|
| Layout (most expensive) | `width`, `height`, `top`, `left`, `margin`, `padding`, `font-size` |
| Paint          | `color`, `background`, `box-shadow`, `border-radius`   |
| Composite only (cheapest) | `transform`, `opacity`, `filter`               |

**For animations, always prefer `transform` and `opacity`** — they skip layout and paint, running on the compositor
thread (GPU).

```css
/* BAD — triggers layout on every frame */
.animate { transition: left 0.3s; }

/* GOOD — compositor only */
.animate { transition: transform 0.3s; }
```

### `will-change`

Hints the browser to promote an element to its own compositor layer ahead of time:

```css
.card {
    will-change: transform; /* prepare for animation */
}
.card.animating {
    transform: scale(1.05);
}
```

Use sparingly — each layer consumes GPU memory. Remove after animation completes.

### `content-visibility`

Skips rendering of off-screen elements entirely:

```css
.section {
    content-visibility: auto;
    contain-intrinsic-size: auto 500px; /* estimated height for scrollbar accuracy */
}
```

Can dramatically reduce initial rendering cost on long pages.

---

## Caching Strategies

### HTTP Cache Headers

| Header           | Purpose                                                   |
|------------------|-----------------------------------------------------------|
| `Cache-Control`  | Primary cache directive                                   |
| `ETag`           | Content hash — enables conditional requests               |
| `Last-Modified`  | Timestamp — enables conditional requests                  |

**Common patterns:**

```
# Immutable assets (hashed filenames like app.a1b2c3.js)
Cache-Control: public, max-age=31536000, immutable

# HTML pages (must revalidate every time)
Cache-Control: no-cache
# ("no-cache" means "revalidate before using", NOT "don't cache")

# Sensitive data
Cache-Control: no-store
```

### Service Worker Caching

```javascript
// cache-first strategy (good for static assets)
self.addEventListener('fetch', event => {
    event.respondWith(
        caches.match(event.request)
            .then(cached => cached || fetch(event.request))
    );
});
```

| Strategy          | Best for                        | Freshness  |
|-------------------|---------------------------------|------------|
| Cache-first       | Static assets, fonts            | Low        |
| Network-first     | API calls, dynamic content      | High       |
| Stale-while-revalidate | Balance of speed and freshness | Medium  |

---

## Fonts

### Optimization

- Use `font-display: swap` — show fallback font immediately, swap when custom font loads.
- Preload critical fonts: `<link rel="preload" as="font" href="..." crossorigin>`.
- Use WOFF2 format (best compression).
- Subset fonts — include only the characters you need.
- Self-host fonts instead of loading from third-party CDNs (saves a `preconnect`).

```css
@font-face {
    font-family: 'Inter';
    src: url('/fonts/inter.woff2') format('woff2');
    font-display: swap;
    unicode-range: U+0000-00FF; /* Latin subset */
}
```

### `font-display` Values

| Value      | Behavior                                                      |
|------------|---------------------------------------------------------------|
| `auto`     | Browser default (usually `block`)                             |
| `block`    | Invisible text for up to 3s, then swap (FOIT)                |
| `swap`     | Fallback immediately, swap when ready (FOUT)                 |
| `fallback` | Very short block (~100ms), short swap period, then keep fallback |
| `optional` | Very short block, browser may not swap at all if slow        |

---

## Measuring Performance

### Lab Tools (Synthetic)

| Tool               | Type          | Use case                           |
|--------------------|---------------|------------------------------------|
| Lighthouse         | Chrome DevTools / CLI | Overall performance audit    |
| WebPageTest        | Online        | Detailed waterfall, filmstrip      |
| Chrome DevTools Performance tab | In-browser | Flame chart, long tasks, layout shifts |

### Field Tools (Real User Monitoring)

| Tool                       | Type              |
|----------------------------|-------------------|
| Chrome UX Report (CrUX)   | Public dataset    |
| web-vitals JS library      | npm package       |
| Google Search Console      | Core Web Vitals report |

```javascript
import { onLCP, onINP, onCLS } from 'web-vitals';

onLCP(console.log);
onINP(console.log);
onCLS(console.log);
```

### Performance API

```javascript
// measure custom timing
performance.mark('start');
// ... work ...
performance.mark('end');
performance.measure('myTask', 'start', 'end');

// get navigation timing
const nav = performance.getEntriesByType('navigation')[0];
console.log('TTFB:', nav.responseStart - nav.requestStart);

// observe long tasks
const observer = new PerformanceObserver(list => {
    for (const entry of list.getEntries()) {
        console.log('Long task:', entry.duration);
    }
});
observer.observe({ type: 'longtask', buffered: true });
```

---

## Common Interview Questions

### What is the difference between `preload`, `prefetch`, and `preconnect`?

- **`preload`** — fetch a resource needed on the **current** page with high priority (fonts, LCP image, critical script).
- **`prefetch`** — fetch a resource for a **future** navigation at idle time (low priority).
- **`preconnect`** — establish a connection (DNS + TCP + TLS) to a third-party origin before the browser discovers it needs to.

### How do you optimize LCP?

1. Identify the LCP element (DevTools → Performance → Timings).
2. Ensure it's in the initial HTML (not injected by JS).
3. Preload the resource (`<link rel="preload">`).
4. Set `fetchpriority="high"` on the LCP `<img>`.
5. Don't lazy-load it.
6. Inline critical CSS, defer the rest.
7. Reduce TTFB (CDN, server optimization, caching).

### What is layout thrashing and how do you avoid it?

Layout thrashing occurs when JavaScript reads layout properties and writes to the DOM in an interleaved pattern, forcing
the browser to recalculate layout multiple times synchronously. Solution: batch all reads first, then all writes —
or use `requestAnimationFrame` to defer writes to the next frame.

### What is tree shaking?

Dead code elimination for ES modules. The bundler (webpack, Rollup, esbuild) analyzes `import`/`export` statements
and removes code that is never imported. Requires ES modules (`import`/`export`), not CommonJS (`require`). Side
effects in modules can prevent tree shaking — use `"sideEffects": false` in `package.json`.

### Why use `transform` for animations instead of `top`/`left`?

`top`/`left` changes trigger **layout** → **paint** → **composite** on every frame. `transform` only triggers
**composite** — it runs on the GPU without touching the main thread. The result: smoother 60fps animations with no
jank.

### What is CLS and how do you fix it?

CLS measures unexpected layout shifts. Common causes: images without dimensions, dynamically injected content,
late-loading fonts, ads without reserved space. Fix by always specifying dimensions (`width`/`height` or
`aspect-ratio`), reserving space for dynamic content, using `font-display: swap` with preloaded fonts, and avoiding
DOM insertions above existing content.
