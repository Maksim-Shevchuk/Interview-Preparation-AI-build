# Event Loop

JavaScript is **single-threaded** but achieves concurrency through the **event loop**. This is one of the most popular
interview topics — interviewers love asking about the order of `console.log` outputs.

## Core Architecture

```
┌──────────────────────────────┐
│         Call Stack            │  ← executes one frame at a time
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│        Event Loop             │  ← checks: stack empty? → pick next task
└──────────┬───────────────────┘
           │
     ┌─────┴──────┐
     ▼            ▼
┌─────────┐  ┌──────────────┐
│Microtask│  │  Macrotask   │
│  Queue  │  │   Queue      │
│(Job Q)  │  │ (Task Q)     │
└─────────┘  └──────────────┘
```

### Components

| Component          | Description                                                          |
|--------------------|----------------------------------------------------------------------|
| **Call Stack**     | LIFO stack of execution contexts. JS runs the top frame to completion |
| **Web APIs**       | Browser-provided: `setTimeout`, `fetch`, DOM events, etc. Run async operations off the main thread |
| **Macrotask Queue** | Queue of callbacks from: `setTimeout`, `setInterval`, `setImmediate` (Node), I/O, UI rendering |
| **Microtask Queue** | Queue of callbacks from: `Promise.then/catch/finally`, `queueMicrotask()`, `MutationObserver` |

## Event Loop Algorithm

One "tick" of the event loop:

1. Execute the current **macrotask** (e.g., script evaluation, a `setTimeout` callback) to completion.
2. Drain the **entire microtask queue** — execute all microtasks, including any new microtasks added during this phase.
3. **Render** (if needed) — the browser may update the UI (requestAnimationFrame, layout, paint).
4. Pick the next **macrotask** from the queue and go to step 1.

**Key insight:** Microtasks are processed **between** macrotasks and always **before** the next macrotask. Microtasks
added during microtask processing are also executed in the same phase.

## Classic Interview Output Question

```javascript
console.log("1");                          // sync

setTimeout(() => console.log("2"), 0);     // macrotask

Promise.resolve().then(() => {
    console.log("3");                      // microtask
    Promise.resolve().then(() => {
        console.log("4");                  // microtask (added during microtask phase)
    });
});

console.log("5");                          // sync
```

**Output: `1, 5, 3, 4, 2`**

Explanation:
1. Sync code runs: `1`, `5`.
2. Call stack empty → drain microtask queue: `3`. New microtask added → `4`.
3. Microtask queue empty → pick next macrotask: `2`.

## More Complex Example

```javascript
console.log("start");

setTimeout(() => console.log("timeout1"), 0);

Promise.resolve()
    .then(() => console.log("promise1"))
    .then(() => console.log("promise2"));

setTimeout(() => console.log("timeout2"), 0);

queueMicrotask(() => console.log("microtask1"));

console.log("end");
```

**Output: `start, end, promise1, microtask1, promise2, timeout1, timeout2`**

Explanation:
1. Sync: `start`, `end`.
2. Microtasks: `promise1` (first `.then`), `microtask1` (`queueMicrotask`). After `promise1` resolves, `promise2` is
   queued → `promise2`.
3. Macrotasks: `timeout1`, `timeout2`.

## `async`/`await` and the Event Loop

`async`/`await` is syntactic sugar over Promises. Everything after `await` is essentially in a `.then()` callback
(microtask).

```javascript
async function foo() {
    console.log("foo start");   // sync
    await Promise.resolve();
    console.log("foo end");     // microtask (like .then())
}

console.log("script start");
foo();
console.log("script end");

// Output: "script start", "foo start", "script end", "foo end"
```

## `setTimeout(fn, 0)` — Not Really 0ms

- `setTimeout(fn, 0)` schedules `fn` as a macrotask. It will run after the current call stack clears AND all microtasks
  are drained.
- Browsers clamp minimum delay to ~4ms for nested `setTimeout` (after 5th nesting level).
- `0ms` means "as soon as possible after current work", not immediately.

## `requestAnimationFrame` (rAF)

- Runs **before** the next repaint, typically at 60fps (~16.6ms intervals).
- Not a macrotask, not a microtask — it's in the **rendering phase** of the event loop.
- Use for visual/animation updates.

```javascript
requestAnimationFrame(() => {
    // runs before next paint, after microtasks
    element.style.transform = "translateX(100px)";
});
```

## Node.js Differences

Node's event loop has **6 phases** (simplified):

```
timers → pending callbacks → idle/prepare → poll → check → close callbacks
```

| API                    | Phase       | Notes                                    |
|------------------------|-------------|------------------------------------------|
| `setTimeout`           | timers      | Same as browser                          |
| `setImmediate`         | check       | Runs after I/O events (Node-only)        |
| `process.nextTick`     | _(special)_ | Runs before any other microtask — higher priority than Promise |
| `Promise.then`         | microtasks  | Runs after `process.nextTick` callbacks  |

```javascript
// Node.js
setTimeout(() => console.log("timeout"), 0);
setImmediate(() => console.log("immediate"));
process.nextTick(() => console.log("nextTick"));
Promise.resolve().then(() => console.log("promise"));

// Output: "nextTick", "promise", "timeout" or "immediate" (timer vs check order is non-deterministic at top level)
```

## Common Interview Questions

1. **What is the event loop?** — A mechanism that lets single-threaded JS handle async operations. It continuously
   checks: if the call stack is empty, pick the next task from the queue.
2. **Microtasks vs macrotasks?** — Microtasks (Promises, queueMicrotask) are drained completely between each macrotask
   (setTimeout, I/O). Microtasks have higher priority.
3. **Why does `setTimeout(fn, 0)` not run immediately?** — It schedules a macrotask. It runs only after the current
   call stack is empty and all microtasks are processed.
4. **What happens if a microtask queues another microtask?** — It gets processed in the same microtask phase. This
   can block rendering if microtasks keep adding more microtasks (starvation).
5. **What is `requestAnimationFrame`?** — Schedules a callback before the next browser repaint. Runs in the rendering
   phase, after microtasks, before the next macrotask.
6. **Predict the output of [code with mixed sync/async/setTimeout/Promise]** — Apply the algorithm: sync first →
   drain microtasks → pick next macrotask → repeat.

## Related

- [Promises and Async/Await](./promises-and-async-await.md) — Promise API details
- [Closures and Scope](./closures-and-scope.md) — closures in callbacks

## Resources

- [MDN — The event loop](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Event_loop)
- [Jake Archibald — Tasks, microtasks, queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)
- [Philip Roberts — What the heck is the event loop anyway? (JSConf)](https://www.youtube.com/watch?v=8aGhZQkoFbQ)
