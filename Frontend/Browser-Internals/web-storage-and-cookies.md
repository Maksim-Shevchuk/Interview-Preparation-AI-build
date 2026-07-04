# Web Storage and Cookies

Three основных механизма хранения данных в браузере: **Cookies**, **localStorage**, **sessionStorage**. Частый вопрос
на интервью — в чём разница и когда какой использовать.

## Сравнительная таблица

| Feature              | Cookie                        | localStorage                 | sessionStorage               |
|----------------------|-------------------------------|------------------------------|------------------------------|
| **Capacity**         | ~4 KB per cookie              | ~5–10 MB                     | ~5–10 MB                     |
| **Lifetime**         | Until `Expires`/`Max-Age` or session | Until explicitly deleted | Until tab/window is closed   |
| **Sent to server**   | ✅ Automatically with every HTTP request | ❌ Never              | ❌ Never                     |
| **Scope**            | Domain + Path                 | Origin (protocol + domain + port) | Origin + **tab**        |
| **Accessible from**  | Server (headers) + Client (JS)| Client (JS) only             | Client (JS) only             |
| **Shared across tabs**| ✅ Yes                       | ✅ Yes                       | ❌ No (per tab)              |
| **API**              | `document.cookie` (string)    | `getItem`/`setItem`/`removeItem` | Same as localStorage    |

## Cookies

### Setting Cookies

```javascript
// Client-side
document.cookie = "theme=dark; path=/; max-age=31536000"; // 1 year
document.cookie = "lang=en; path=/; max-age=31536000";

// Reading — returns ALL cookies as one string
document.cookie; // "theme=dark; lang=en"

// Deleting — set max-age to 0 or expires in the past
document.cookie = "theme=; max-age=0";
```

```http
// Server-side (Set-Cookie header)
Set-Cookie: sessionId=abc123; Path=/; HttpOnly; Secure; SameSite=Strict; Max-Age=3600
```

### Cookie Attributes

| Attribute    | Purpose                                                              |
|--------------|----------------------------------------------------------------------|
| `Path`       | URL path scope — cookie sent only for requests to this path and below |
| `Domain`     | Domain scope — `.example.com` includes subdomains                    |
| `Max-Age`    | Lifetime in seconds (overrides `Expires`)                            |
| `Expires`    | Absolute expiration date (`Thu, 01 Jan 2026 00:00:00 GMT`)           |
| `HttpOnly`   | Not accessible via `document.cookie` — prevents XSS theft           |
| `Secure`     | Sent only over HTTPS                                                 |
| `SameSite`   | CSRF protection — `Strict`, `Lax` (default), or `None`              |

### `SameSite` Values

| Value    | Cross-site requests      | Use case                               |
|----------|--------------------------|----------------------------------------|
| `Strict` | Never sent               | Banking, sensitive actions              |
| `Lax`    | Sent on top-level navigation (GET links), blocked on POST/iframe/AJAX | Default, good balance |
| `None`   | Always sent (requires `Secure`) | Third-party integrations, embedded widgets |

### Session vs Persistent Cookies

- **Session cookie** — no `Max-Age` or `Expires` → deleted when browser closes.
- **Persistent cookie** — has `Max-Age` or `Expires` → survives browser restarts.

### When to Use Cookies

- **Authentication** — session IDs, JWT tokens (with `HttpOnly` + `Secure` + `SameSite`).
- **Server-side personalization** — the server needs the value on every request (language, A/B test bucket).
- **Tracking / analytics** — third-party cookies (being phased out).

## localStorage

Persistent key-value storage scoped to the **origin**. Data survives page reloads, tab closes, and browser restarts.

### API

```javascript
// Store
localStorage.setItem("user", JSON.stringify({ name: "Alice", age: 30 }));

// Retrieve
const user = JSON.parse(localStorage.getItem("user")); // { name: "Alice", age: 30 }

// Remove one key
localStorage.removeItem("user");

// Clear everything for this origin
localStorage.clear();

// Number of stored keys
localStorage.length;

// Iterate
for (let i = 0; i < localStorage.length; i++) {
    const key = localStorage.key(i);
    console.log(key, localStorage.getItem(key));
}
```

### `storage` Event — Cross-Tab Sync

When `localStorage` changes in **another tab** of the same origin, a `storage` event fires:

```javascript
window.addEventListener("storage", (event) => {
    console.log(event.key);       // changed key
    console.log(event.oldValue);  // previous value
    console.log(event.newValue);  // new value
    console.log(event.url);       // URL of the tab that made the change
});

// Note: the event does NOT fire in the tab that made the change — only in other tabs.
```

### When to Use localStorage

- **User preferences** — theme, language, sidebar collapsed state.
- **Cached data** — draft form content, shopping cart (non-sensitive).
- **Tokens** — only if `HttpOnly` cookies are not an option (less secure — vulnerable to XSS).
- Anything that should **persist** across sessions and be **shared across tabs**.

## sessionStorage

Same API as `localStorage`, but data is scoped to the **browser tab/window** and cleared when the tab is closed.

```javascript
sessionStorage.setItem("step", "2");
sessionStorage.getItem("step"); // "2"

// Opens a new tab via link — sessionStorage is NOT shared
// Duplicating a tab — sessionStorage IS copied (one-time snapshot)
```

### When to Use sessionStorage

- **Multi-step forms** — preserve wizard state within a single tab session.
- **One-time data** — a redirect token, temporary state that should not leak to other tabs.
- **Per-tab state** — different filters/views in different tabs of the same app.

## IndexedDB (Brief)

For larger / structured data beyond what Web Storage handles:

| Feature       | Web Storage (local/session) | IndexedDB                          |
|---------------|-----------------------------|------------------------------------|
| Capacity      | ~5–10 MB                    | Hundreds of MB+                    |
| Data model    | String key → String value   | Object stores, indexes, cursors    |
| API           | Synchronous                 | Asynchronous (event/Promise-based) |
| Transactions  | No                          | Yes (ACID within a store)          |
| Use case      | Simple preferences          | Offline apps, large datasets, files|

Libraries that simplify IndexedDB: **Dexie.js**, **idb** (by Jake Archibald).

## Security Considerations

### XSS and Storage

| Storage        | Accessible via JS | XSS risk                                               |
|----------------|:-:|---------------------------------------------------------------|
| `localStorage` | ✅ | Attacker script can read/steal all stored data               |
| `sessionStorage` | ✅ | Same — but limited to the tab                             |
| Cookie (`HttpOnly`) | ❌ | Cannot be read by JS — safest for tokens              |
| Cookie (no `HttpOnly`) | ✅ | Same risk as localStorage                          |

**Best practice for auth tokens:**
- **Preferred:** `HttpOnly` + `Secure` + `SameSite=Strict` cookie — not accessible via JS.
- **If cookie is not possible** (e.g., cross-origin SPA): store in memory (variable) and use `localStorage` only for
  refresh tokens with short expiry. Accept the XSS risk and mitigate via CSP.

### CSRF and Cookies

Cookies are sent automatically → vulnerable to CSRF. Mitigations:
- `SameSite=Strict` or `SameSite=Lax`.
- CSRF tokens (synchronizer token pattern).
- Check `Origin` / `Referer` headers on the server.

`localStorage`/`sessionStorage` are **not** vulnerable to CSRF — they are never sent automatically.

## Practical Patterns

### Storing Objects

```javascript
// Web Storage only stores strings — serialize with JSON
const save = (key, value) => localStorage.setItem(key, JSON.stringify(value));
const load = (key) => JSON.parse(localStorage.getItem(key));

save("settings", { theme: "dark", fontSize: 14 });
load("settings"); // { theme: "dark", fontSize: 14 }
```

### Storage Wrapper with Expiry

`localStorage` has no built-in expiry. Common pattern:

```javascript
function setWithExpiry(key, value, ttlMs) {
    localStorage.setItem(key, JSON.stringify({
        value,
        expiry: Date.now() + ttlMs,
    }));
}

function getWithExpiry(key) {
    const raw = localStorage.getItem(key);
    if (!raw) return null;

    const { value, expiry } = JSON.parse(raw);
    if (Date.now() > expiry) {
        localStorage.removeItem(key);
        return null;
    }
    return value;
}
```

### React Hook

```tsx
function useLocalStorage<T>(key: string, initialValue: T) {
    const [value, setValue] = useState<T>(() => {
        const stored = localStorage.getItem(key);
        return stored ? JSON.parse(stored) : initialValue;
    });

    useEffect(() => {
        localStorage.setItem(key, JSON.stringify(value));
    }, [key, value]);

    return [value, setValue] as const;
}

const [theme, setTheme] = useLocalStorage("theme", "light");
```

## Common Interview Questions

1. **Cookie vs localStorage vs sessionStorage?** — Cookies: small (4 KB), sent to server, configurable lifetime.
   localStorage: large (5–10 MB), client-only, persistent, shared across tabs. sessionStorage: same as localStorage
   but scoped to one tab and cleared when tab closes.
2. **Where to store JWT tokens?** — Preferred: `HttpOnly` + `Secure` cookie (not accessible via JS, immune to XSS).
   Alternative: in-memory variable + refresh token. Avoid localStorage for sensitive tokens if XSS is a concern.
3. **What is `HttpOnly`?** — Cookie attribute that blocks `document.cookie` access from JavaScript. Protects the
   cookie from XSS attacks.
4. **What is `SameSite`?** — Cookie attribute that controls whether the cookie is sent with cross-site requests.
   `Strict` = never, `Lax` = only on top-level GET navigation, `None` = always (requires `Secure`).
5. **Can localStorage trigger events in other tabs?** — Yes. The `storage` event fires in other tabs of the same
   origin when localStorage changes. It does not fire in the tab that made the change.
6. **What is the difference between session cookies and persistent cookies?** — Session cookies have no
   `Expires`/`Max-Age` and are deleted when the browser closes. Persistent cookies have an expiration and survive
   browser restarts.

## Related

- [ES6+ Features](../JavaScript/es6-features.md) — `structuredClone` for deep cloning stored objects
- [Hooks in Depth](../React/hooks/hooks-in-depth.md) — `useLocalStorage` custom hook pattern

## Resources

- [MDN — Web Storage API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Storage_API)
- [MDN — Document.cookie](https://developer.mozilla.org/en-US/docs/Web/API/Document/cookie)
- [MDN — IndexedDB](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API)
- [OWASP — Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
