# Promises and Async/Await

Promises are the foundation of modern asynchronous JavaScript. `async`/`await` (ES2017) is syntactic sugar that makes
Promise-based code look synchronous. Both are heavily tested in interviews.

## Promise Fundamentals

A Promise is an object representing the eventual completion or failure of an async operation.

### States

```
          ┌── fulfilled (resolved with a value)
pending ──┤
          └── rejected (rejected with a reason/error)
```

- A promise is **settled** when it is either fulfilled or rejected.
- Once settled, a promise **cannot change state** — it is immutable.

### Creating a Promise

```javascript
const promise = new Promise((resolve, reject) => {
    // async operation
    const success = true;
    if (success) {
        resolve("result");  // → fulfilled
    } else {
        reject(new Error("something went wrong")); // → rejected
    }
});
```

### Consuming a Promise

```javascript
promise
    .then(value => console.log(value))     // handles fulfillment
    .catch(error => console.error(error))  // handles rejection
    .finally(() => console.log("done"));   // runs regardless of outcome
```

`.then()` returns a **new Promise**, enabling chaining.

## Promise Chaining

Each `.then()` returns a new Promise. The return value of one `.then()` becomes the input of the next:

```javascript
fetch("/api/user")
    .then(response => response.json())     // returns a Promise
    .then(user => fetch(`/api/posts/${user.id}`))
    .then(response => response.json())
    .then(posts => console.log(posts))
    .catch(error => console.error(error)); // catches ANY error in the chain
```

**Rules of `.then()` return values:**
- Return a value → next `.then()` receives it.
- Return a Promise → next `.then()` waits for it and receives its resolved value.
- Throw an error → chain skips to the nearest `.catch()`.

## Error Handling

```javascript
// ❌ Anti-pattern: unhandled rejection
somePromise.then(value => {
    // if this throws, nothing catches it
});

// ✅ Always add .catch()
somePromise
    .then(value => process(value))
    .catch(error => handleError(error));

// ✅ .catch() in the middle — chain continues after recovery
fetch("/api")
    .then(res => res.json())
    .catch(err => {
        console.warn("fetch failed, using fallback");
        return { fallback: true };          // recovery value
    })
    .then(data => render(data));            // receives fallback data if fetch failed
```

### `unhandledrejection` Event

```javascript
window.addEventListener("unhandledrejection", event => {
    console.error("Unhandled rejection:", event.reason);
    event.preventDefault(); // prevents default browser logging
});
```

## Static Methods

### `Promise.all(iterable)`

Waits for **all** promises to fulfill. Rejects on **first** rejection (fail-fast).

```javascript
const [users, posts] = await Promise.all([
    fetch("/api/users").then(r => r.json()),
    fetch("/api/posts").then(r => r.json()),
]);
// Both requests run in parallel. If either fails, the whole thing rejects.
```

### `Promise.allSettled(iterable)` (ES2020)

Waits for **all** promises to settle (fulfill or reject). Never short-circuits.

```javascript
const results = await Promise.allSettled([
    fetch("/api/users"),
    fetch("/api/failing-endpoint"),
]);
// results: [
//   { status: "fulfilled", value: Response },
//   { status: "rejected",  reason: Error }
// ]
```

### `Promise.race(iterable)`

Resolves/rejects with the **first** settled promise.

```javascript
const result = await Promise.race([
    fetch("/api/data"),
    new Promise((_, reject) => setTimeout(() => reject(new Error("Timeout")), 5000)),
]);
```

### `Promise.any(iterable)` (ES2021)

Resolves with the **first fulfilled** promise. Rejects only if **all** reject (with `AggregateError`).

```javascript
const fastest = await Promise.any([
    fetch("https://cdn1.example.com/data"),
    fetch("https://cdn2.example.com/data"),
]);
```

### Comparison Table

| Method            | Resolves when              | Rejects when                 | Short-circuits? |
|-------------------|----------------------------|------------------------------|-----------------|
| `Promise.all`     | All fulfill                | First rejection              | Yes (on reject) |
| `Promise.allSettled` | All settle              | Never                        | No              |
| `Promise.race`    | First settles              | First settles (if rejected)  | Yes             |
| `Promise.any`     | First fulfills             | All reject                   | Yes (on fulfill)|

## `async`/`await`

### Basics

```javascript
async function fetchUser(id) {
    const response = await fetch(`/api/users/${id}`); // pauses until resolved
    const user = await response.json();
    return user; // wrapped in Promise.resolve(user)
}
```

- `async` function **always returns a Promise**.
- `await` pauses execution of the async function and yields control back to the caller.
- `await` unwraps the Promise — you get the resolved value directly.

### Error Handling with try/catch

```javascript
async function loadData() {
    try {
        const response = await fetch("/api/data");
        if (!response.ok) throw new Error(`HTTP ${response.status}`);
        return await response.json();
    } catch (error) {
        console.error("Failed to load:", error);
        return null; // fallback
    }
}
```

### Sequential vs Parallel

```javascript
// ❌ Sequential — each await waits for the previous one
async function sequential() {
    const users = await fetchUsers();  // waits...
    const posts = await fetchPosts();  // waits... (starts AFTER users finishes)
    return { users, posts };
}

// ✅ Parallel — both start immediately
async function parallel() {
    const [users, posts] = await Promise.all([
        fetchUsers(),
        fetchPosts(),
    ]);
    return { users, posts };
}
```

### Top-Level `await`

Available in ES modules (`.mjs` or `"type": "module"` in package.json):

```javascript
// module.mjs
const config = await fetch("/config.json").then(r => r.json());
export default config;
```

## Promisifying Callbacks

Converting callback-based APIs to Promise-based:

```javascript
function readFileAsync(path) {
    return new Promise((resolve, reject) => {
        fs.readFile(path, "utf-8", (err, data) => {
            if (err) reject(err);
            else resolve(data);
        });
    });
}

// Node.js built-in utility:
const { promisify } = require("util");
const readFile = promisify(fs.readFile);
```

## Common Mistakes

```javascript
// ❌ Forgetting to return in .then() chain
promise.then(val => {
    fetch("/api");        // Promise is created but not returned — next .then gets undefined
});

// ❌ Nesting .then() (callback hell with Promises)
fetch(url).then(res => {
    res.json().then(data => {
        process(data).then(result => { /* ... */ });
    });
});

// ✅ Flatten the chain
fetch(url)
    .then(res => res.json())
    .then(data => process(data))
    .then(result => { /* ... */ });

// ❌ await inside forEach (forEach doesn't await)
items.forEach(async item => {
    await process(item); // runs in parallel, not sequential!
});

// ✅ Use for...of for sequential
for (const item of items) {
    await process(item);
}

// ✅ Use Promise.all for parallel
await Promise.all(items.map(item => process(item)));
```

## Common Interview Questions

1. **What is a Promise?** — An object representing a future value. Has three states: pending, fulfilled, rejected.
   Once settled, immutable.
2. **Difference between `.then().catch()` and `.then(onFulfilled, onRejected)`?** — `.then(f, r)` won't catch errors
   thrown inside `f` itself. Chained `.catch()` catches errors from both the original promise AND the `.then()`
   handler.
3. **`Promise.all` vs `Promise.allSettled`?** — `all` fails fast on first rejection; `allSettled` waits for
   everything and returns status objects.
4. **What does `async` function return?** — Always a Promise. Return value is wrapped in `Promise.resolve()`.
5. **How to run async operations in parallel?** — `Promise.all([op1(), op2()])`. Don't `await` each one sequentially
   if they're independent.
6. **Why shouldn't you use `await` inside `forEach`?** — `forEach` ignores the returned Promises. Use `for...of` for
   sequential processing or `Promise.all(arr.map(...))` for parallel.

## Related

- [Event Loop](./event-loop.md) — microtask queue and Promise scheduling
- [ES6+ Features](./es6-features.md) — Promises were introduced in ES2015

## Resources

- [MDN — Using Promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises)
- [MDN — async function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)
- [javascript.info — Promises, async/await](https://javascript.info/async)
