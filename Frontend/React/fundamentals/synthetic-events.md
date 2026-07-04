# Synthetic Events — React vs Native Events

React wraps the browser's native event system with its own **Synthetic Event** layer. Interviewers test whether you
understand the differences, event delegation, and edge cases.

## What is a SyntheticEvent?

A cross-browser wrapper around the native `Event` object. Every event handler in React receives a `SyntheticEvent`
instead of the native DOM event.

```tsx
function Button() {
    const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
        console.log(e);              // SyntheticEvent
        console.log(e.nativeEvent);  // native MouseEvent (access if needed)
        console.log(e.target);       // DOM element that fired the event
        console.log(e.currentTarget);// DOM element with the handler attached
    };

    return <button onClick={handleClick}>Click</button>;
}
```

### SyntheticEvent Interface

```typescript
interface SyntheticEvent<T = Element> {
    // Same API as native Event
    type: string;
    target: EventTarget;
    currentTarget: T;
    bubbles: boolean;
    cancelable: boolean;
    defaultPrevented: boolean;
    eventPhase: number;
    timeStamp: number;

    // Methods
    preventDefault(): void;
    stopPropagation(): void;
    isPropagationStopped(): boolean;
    isDefaultPrevented(): boolean;

    // Access to underlying native event
    nativeEvent: Event;
}
```

Specialized interfaces extend this: `React.MouseEvent`, `React.KeyboardEvent`, `React.ChangeEvent`,
`React.FormEvent`, `React.FocusEvent`, `React.DragEvent`, `React.TouchEvent`, etc.

## React vs Native: Key Differences

| Aspect                  | React (SyntheticEvent)                | Native DOM Event                     |
|-------------------------|---------------------------------------|--------------------------------------|
| **Naming**              | camelCase: `onClick`, `onChange`       | lowercase: `onclick`, `onchange`     |
| **Handler value**       | Function: `onClick={handleClick}`     | String: `onclick="handleClick()"`    |
| **Event delegation**    | Events delegated to **root** (React 17+) | Attached to individual elements   |
| **Cross-browser**       | ✅ Normalized                         | ❌ Browser quirks                    |
| **`onChange`**           | Fires on **every keystroke**          | Fires on **blur** (loses focus)      |
| **`onInput`**           | Same as `onChange` in React           | Fires on every keystroke             |
| **`false` return**      | Does nothing                          | Prevents default in inline handlers  |
| **Prevent default**     | `e.preventDefault()`                  | `e.preventDefault()` or `return false` |
| **Stop propagation**    | Only stops React's synthetic bubbling | Stops native bubbling                |
| **Event pooling**       | Removed in React 17+                 | N/A                                  |

## Event Delegation in React

### How It Works

React does **NOT** attach event listeners to individual DOM nodes. Instead, it uses **event delegation**:

```
React 16:   All events delegated to document
React 17+:  All events delegated to the React root container
```

```
┌─────────────── Root container ───────────────┐
│  Single event listener (onClick, etc.)       │  ← React listens HERE
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ <button> │  │  <input> │  │  <div>   │  │  ← No individual listeners
│  └──────────┘  └──────────┘  └──────────┘  │
└──────────────────────────────────────────────┘
```

**Why?**
- **Performance** — one listener instead of thousands (imagine a list with 10K items).
- **Memory** — fewer event subscriptions.
- **Dynamic elements** — works for components added/removed without re-attaching listeners.
- **Multiple React roots** — React 17+ moved from `document` to root container so multiple React apps on one page
  don't interfere with each other.

### Implications

```tsx
// React synthetic event bubbles through React's virtual tree, not the DOM tree
// This matters when mixing React and non-React code

function App() {
    return (
        <div onClick={() => console.log("React div")}>
            <ChildWithPortal />
        </div>
    );
}

// Even though a Portal renders somewhere else in the DOM,
// the event still bubbles through the React component tree (to the parent div)
```

## `onChange` — The Big Difference

The most important behavioral difference:

```tsx
// React onChange fires on EVERY keystroke (like native "input" event)
<input onChange={e => console.log(e.target.value)} />
// User types "abc": logs "a", "ab", "abc"

// Native onchange fires only when the field LOSES FOCUS (blur)
// <input onchange="console.log(this.value)">
// User types "abc" and clicks elsewhere: logs "abc" once
```

React deliberately changed this behavior because the native `change` event is inconsistent across form elements
and rarely what developers actually want.

For `<select>` and `<input type="checkbox">`, both React and native fire on each change — the difference is mainly
noticeable on text inputs.

## `preventDefault` and `stopPropagation`

```tsx
// Prevent default behavior (e.g., form submission, link navigation)
function Form() {
    const handleSubmit = (e: React.FormEvent) => {
        e.preventDefault(); // prevents page reload
        // handle form data...
    };

    return <form onSubmit={handleSubmit}>...</form>;
}

// Stop event from bubbling up the React tree
function Child() {
    const handleClick = (e: React.MouseEvent) => {
        e.stopPropagation(); // parent's onClick won't fire
    };

    return <button onClick={handleClick}>Click</button>;
}
```

### `stopPropagation` Gotcha — React vs Native

```tsx
function App() {
    useEffect(() => {
        // Native listener on document
        document.addEventListener("click", () => {
            console.log("native document click");
        });
    }, []);

    return (
        <button onClick={(e) => {
            e.stopPropagation(); // stops REACT bubbling only
            // Native document listener STILL fires! (React 17+: depends on root position)
        }}>
            Click
        </button>
    );
}
```

To also stop the native event: `e.nativeEvent.stopImmediatePropagation()`.

## Capture Phase

React supports the capture phase (event traveling down from root to target):

```tsx
// Capture: fires top-down (before target)
<div onClickCapture={() => console.log("1: div capture")}>
    <button onClick={() => console.log("2: button bubble")}>
        Click
    </button>
</div>
// Output: "1: div capture", "2: button bubble"
```

Any event handler has a capture variant: `onClickCapture`, `onChangeCapture`, `onFocusCapture`, etc.

## Event Pooling (Historical — React 16 and Earlier)

In React 16 and earlier, `SyntheticEvent` objects were **pooled and reused** for performance. After the handler
finished, all properties were nullified:

```jsx
// React 16 — BROKEN
function handleClick(e) {
    setTimeout(() => {
        console.log(e.target); // null — event was returned to pool
    }, 100);
}

// Fix in React 16: e.persist()
function handleClick(e) {
    e.persist(); // remove from pool
    setTimeout(() => {
        console.log(e.target); // works
    }, 100);
}
```

**React 17+ removed pooling entirely.** `e.persist()` still exists but is a no-op. This is a common interview
question — know the history.

## Mixing React and Native Events

Sometimes you need native listeners (e.g., for events React doesn't support, or for non-React DOM):

```tsx
function Dropdown() {
    const ref = useRef<HTMLDivElement>(null);

    useEffect(() => {
        // Close on outside click — native listener needed
        function handleClickOutside(e: MouseEvent) {
            if (ref.current && !ref.current.contains(e.target as Node)) {
                setOpen(false);
            }
        }

        document.addEventListener("mousedown", handleClickOutside);
        return () => document.removeEventListener("mousedown", handleClickOutside);
    }, []);

    return <div ref={ref}>...</div>;
}
```

**Rules for mixing:**
- Avoid mixing when possible — use React events.
- If you must use native: add in `useEffect`, remove in cleanup.
- `e.stopPropagation()` in React does NOT stop native propagation (and vice versa).
- Events registered natively on `document` fire **before** React's delegated handler in React 17+ (because React
  delegates to the root, not `document`).

## Passive Event Listeners

For performance-sensitive events like `scroll`, `touchstart`, `wheel`:

```tsx
useEffect(() => {
    const handler = (e: Event) => { /* ... */ };

    // Passive = promise not to call preventDefault() → browser can optimize scrolling
    element.addEventListener("scroll", handler, { passive: true });
    return () => element.removeEventListener("scroll", handler);
}, []);
```

React does NOT have a built-in way to set `{ passive: true }` on synthetic events. Use native listeners for this.

React 17+ marks `onScroll`, `onTouchStart`, `onWheel` as **passive by default** at the document level, but if you
need `preventDefault()` on these, you must use a native listener with `{ passive: false }`.

## Common Event Types in TypeScript

```typescript
// Mouse
onClick:       React.MouseEvent<HTMLButtonElement>
onDoubleClick: React.MouseEvent<HTMLDivElement>
onMouseEnter:  React.MouseEvent<HTMLDivElement>

// Keyboard
onKeyDown:     React.KeyboardEvent<HTMLInputElement>
onKeyUp:       React.KeyboardEvent<HTMLInputElement>

// Form
onChange:       React.ChangeEvent<HTMLInputElement>
onSubmit:       React.FormEvent<HTMLFormElement>

// Focus
onFocus:       React.FocusEvent<HTMLInputElement>
onBlur:        React.FocusEvent<HTMLInputElement>

// Drag
onDrag:        React.DragEvent<HTMLDivElement>
onDrop:        React.DragEvent<HTMLDivElement>

// Touch
onTouchStart:  React.TouchEvent<HTMLDivElement>

// Clipboard
onCopy:        React.ClipboardEvent<HTMLDivElement>
onPaste:       React.ClipboardEvent<HTMLInputElement>

// Scroll
onScroll:      React.UIEvent<HTMLDivElement>
```

## Common Interview Questions

1. **What is a SyntheticEvent?** — React's cross-browser wrapper around the native DOM event. Normalizes behavior
   across browsers. Provides the same interface (`target`, `preventDefault`, `stopPropagation`). Access native event
   via `e.nativeEvent`.
2. **How does React handle events differently from the native DOM?** — React uses event delegation (single listener
   on root), camelCase naming (`onClick`), `onChange` fires on every keystroke (not on blur), and wraps events in
   SyntheticEvent.
3. **What is event delegation and why does React use it?** — One listener on the root container handles all events
   via bubbling. Better performance (fewer listeners), automatic cleanup, works with dynamic elements.
4. **What is event pooling?** — In React 16, SyntheticEvents were reused (properties nullified after handler). Fixed
   with `e.persist()`. Removed entirely in React 17+. Know this as a historical fact.
5. **Difference between React `onChange` and native `onchange`?** — React fires on every keystroke (like native
   `input` event). Native fires on blur. React normalized this because native `change` behavior was inconsistent.
6. **Does `e.stopPropagation()` in React stop native events?** — No. It only stops propagation within React's
   synthetic event system. Native listeners on `document` will still fire. Use `e.nativeEvent.stopImmediatePropagation()`
   to stop both.
7. **Why did React 17 move delegation from `document` to the root container?** — To support multiple React roots on
   one page and prevent React apps from interfering with each other's events.

## Related

- [Components, JSX, and Virtual DOM](./components-jsx-and-virtual-dom.md) — event handling in JSX, reconciliation
- [Hooks in Depth](../hooks/hooks-in-depth.md) — `useEffect` for native event listeners
- [Event Loop](../../JavaScript/event-loop.md) — how async event callbacks are scheduled

## Resources

- [React Docs — Responding to Events](https://react.dev/learn/responding-to-events)
- [React Docs — SyntheticEvent](https://react.dev/reference/react-dom/components/common#react-event-object)
- [React Blog — React 17 Event Delegation Changes](https://legacy.reactjs.org/blog/2020/08/10/react-v17-rc.html#changes-to-event-delegation)
