# React Portals and Refs

Portals render children into a DOM node **outside** the parent component's hierarchy. Refs provide direct access to
DOM nodes and mutable values. Both are common interview topics.

## React Portals

### What is a Portal?

A Portal lets a child component render into a **different DOM node** than its parent, while maintaining the React
component tree (context, events still bubble through the React tree, not the DOM tree).

```tsx
import { createPortal } from "react-dom";

function Modal({ children, isOpen }: { children: React.ReactNode; isOpen: boolean }) {
    if (!isOpen) return null;

    return createPortal(
        <div className="modal-overlay">
            <div className="modal-content">
                {children}
            </div>
        </div>,
        document.getElementById("modal-root")! // render here in the DOM
    );
}
```

```html
<!-- index.html -->
<body>
    <div id="root">
        <!-- React app renders here -->
    </div>
    <div id="modal-root">
        <!-- Portals render here — outside #root in the DOM -->
    </div>
</body>
```

### Why Portals?

The main problem portals solve is **CSS stacking context and overflow**:

```tsx
// ❌ Without portal — modal is clipped by parent's overflow:hidden or z-index
<div style={{ overflow: "hidden", position: "relative" }}>
    <Modal>This gets clipped!</Modal>
</div>

// ✅ With portal — modal renders at document.body level, not clipped
<div style={{ overflow: "hidden", position: "relative" }}>
    <Modal>This renders outside the parent in the DOM!</Modal>
</div>
```

### Event Bubbling Through Portals

**Key concept:** Even though the Portal renders outside the parent DOM node, events bubble through the **React
component tree** (not the DOM tree):

```tsx
function App() {
    // This onClick catches clicks from the Portal!
    return (
        <div onClick={() => console.log("Caught in parent!")}>
            <p>Parent component</p>
            <PortalChild />
        </div>
    );
}

function PortalChild() {
    return createPortal(
        <button onClick={() => console.log("Button clicked")}>
            I'm in a Portal
        </button>,
        document.getElementById("portal-root")!
    );
}

// Click the button:
// "Button clicked"
// "Caught in parent!" — event bubbled through React tree, even though DOM structure is different
```

This means Context also works through portals — a portal child can `useContext` from ancestors.

### Common Use Cases

| Use Case         | Why Portal?                                                      |
|------------------|------------------------------------------------------------------|
| **Modals**       | Escape parent `overflow: hidden`, `z-index` stacking context     |
| **Tooltips**     | Position relative to viewport, not clipped by container          |
| **Dropdowns**    | Flyout menus that extend beyond parent bounds                    |
| **Toasts**       | Notification layer at top of page                                |
| **Fullscreen overlays** | Cover the entire viewport regardless of component nesting |

### Portal with `useEffect` for Cleanup

```tsx
function Portal({ children }: { children: React.ReactNode }) {
    const [container] = useState(() => document.createElement("div"));

    useEffect(() => {
        document.body.appendChild(container);
        return () => {
            document.body.removeChild(container);
        };
    }, [container]);

    return createPortal(children, container);
}
```

This creates a fresh container, appends it to `body`, and cleans up on unmount.

## Refs — `useRef` and `forwardRef`

### `useRef` — Two Use Cases

#### 1. DOM Reference

```tsx
function TextInput() {
    const inputRef = useRef<HTMLInputElement>(null);

    const focusInput = () => {
        inputRef.current?.focus();
    };

    return (
        <>
            <input ref={inputRef} />
            <button onClick={focusInput}>Focus</button>
        </>
    );
}
```

#### 2. Mutable Value (No Re-render)

```tsx
function Timer() {
    const intervalRef = useRef<number | null>(null);
    const renderCount = useRef(0);

    renderCount.current += 1; // tracked but doesn't cause re-render

    const start = () => {
        intervalRef.current = window.setInterval(() => tick(), 1000);
    };

    const stop = () => {
        if (intervalRef.current) clearInterval(intervalRef.current);
    };

    return /* ... */;
}
```

### `useRef` vs `useState`

| Feature            | `useRef`                        | `useState`                     |
|--------------------|---------------------------------|--------------------------------|
| Causes re-render   | ❌ No                           | ✅ Yes                         |
| Persists across renders | ✅ Yes                     | ✅ Yes                         |
| Mutable            | ✅ `.current` is mutable        | ❌ Use setter function         |
| Initial value      | `useRef(initialValue)`          | `useState(initialValue)`       |
| Use for            | DOM refs, timers, previous value, any mutable value you don't want to re-render for | UI state that should trigger re-render |

### `forwardRef` — Passing Ref to Child Component

By default, refs are not passed through as regular props. Use `forwardRef` to expose a DOM node from a child:

```tsx
// Child component exposes its input DOM element
const FancyInput = forwardRef<HTMLInputElement, InputProps>(
    function FancyInput(props, ref) {
        return <input ref={ref} className="fancy" {...props} />;
    }
);

// Parent can now ref the child's input
function Form() {
    const inputRef = useRef<HTMLInputElement>(null);

    return (
        <>
            <FancyInput ref={inputRef} placeholder="Name" />
            <button onClick={() => inputRef.current?.focus()}>Focus</button>
        </>
    );
}
```

### `useImperativeHandle` — Custom Ref API

Restrict or customize what the parent can do with the ref:

```tsx
interface ModalHandle {
    open: () => void;
    close: () => void;
}

const Modal = forwardRef<ModalHandle, ModalProps>(
    function Modal(props, ref) {
        const [isOpen, setIsOpen] = useState(false);

        useImperativeHandle(ref, () => ({
            open: () => setIsOpen(true),
            close: () => setIsOpen(false),
        }));

        if (!isOpen) return null;
        return <div className="modal">{props.children}</div>;
    }
);

// Parent
function App() {
    const modalRef = useRef<ModalHandle>(null);

    return (
        <>
            <button onClick={() => modalRef.current?.open()}>Open</button>
            <Modal ref={modalRef}>Content</Modal>
        </>
    );
}
```

### React 19: `ref` as a Regular Prop

React 19 removes the need for `forwardRef` — `ref` can be passed as a regular prop:

```tsx
// React 19+ — no forwardRef needed
function FancyInput({ ref, ...props }: { ref?: React.Ref<HTMLInputElement> }) {
    return <input ref={ref} className="fancy" {...props} />;
}
```

### Callback Refs

Instead of a ref object, pass a **function** that receives the DOM node:

```tsx
function MeasuredBox() {
    const [height, setHeight] = useState(0);

    const measuredRef = useCallback((node: HTMLDivElement | null) => {
        if (node) {
            setHeight(node.getBoundingClientRect().height);
        }
    }, []);

    return (
        <>
            <div ref={measuredRef}>Content with dynamic height</div>
            <p>Height: {height}px</p>
        </>
    );
}
```

**When to use callback refs:**
- Measuring DOM elements after render.
- Conditional refs (element may or may not exist).
- Integrating with third-party DOM libraries.

## Common Interview Questions

1. **What is a React Portal?** — `createPortal(child, container)` renders a component into a DOM node outside the
   parent's DOM hierarchy, while keeping it in the React component tree (events, context still work).
2. **Why use portals?** — To escape CSS constraints like `overflow: hidden`, `z-index` stacking context. Common for
   modals, tooltips, dropdowns, toasts.
3. **Do events bubble through portals?** — Yes, through the **React component tree**, not the DOM tree. A click
   inside a portal bubbles to the React parent, even though the DOM parent is different.
4. **`useRef` vs `useState`?** — `useRef` persists a mutable value without causing re-renders. `useState` triggers
   a re-render on every change. Use `useRef` for DOM references, timers, and values you need to track without
   re-rendering.
5. **What is `forwardRef`?** — A wrapper that lets a parent pass a `ref` to a child component's inner DOM element.
   Without it, `ref` is not forwarded. React 19 removes the need by allowing `ref` as a regular prop.
6. **What is `useImperativeHandle`?** — Customizes what value is exposed to the parent via ref. Useful for exposing
   imperative methods (`.open()`, `.close()`) instead of raw DOM access.

## Related

- [Components, JSX, and Virtual DOM](./components-jsx-and-virtual-dom.md) — component tree, reconciliation
- [Synthetic Events](./synthetic-events.md) — event bubbling through portals
- [Hooks in Depth](../hooks/hooks-in-depth.md) — `useRef`, `useEffect`, `useCallback`

## Resources

- [React Docs — createPortal](https://react.dev/reference/react-dom/createPortal)
- [React Docs — useRef](https://react.dev/reference/react/useRef)
- [React Docs — forwardRef](https://react.dev/reference/react/forwardRef)
