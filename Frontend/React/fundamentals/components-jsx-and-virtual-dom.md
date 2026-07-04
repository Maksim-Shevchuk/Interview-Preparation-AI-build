# Components, JSX, and Virtual DOM

The foundation of React: what components are, how JSX works under the hood, and how the Virtual DOM enables efficient
UI updates.

## JSX

JSX is a **syntax extension** for JavaScript that looks like HTML but compiles to regular function calls.

```jsx
// JSX
const element = <h1 className="title">Hello, {name}!</h1>;

// Compiles to (React 17+ automatic runtime):
import { jsx as _jsx } from "react/jsx-runtime";
const element = _jsx("h1", { className: "title", children: ["Hello, ", name, "!"] });

// Before React 17 (classic runtime):
const element = React.createElement("h1", { className: "title" }, "Hello, ", name, "!");
```

### JSX Rules

1. **Return a single root element** — use `<>...</>` (Fragment) to avoid wrapper divs.
2. **Close all tags** — `<img />`, `<br />`, `<input />`.
3. **camelCase for HTML attributes** — `className`, `htmlFor`, `onClick`, `tabIndex`.
4. **Expressions in `{}`** — any valid JS expression: `{user.name}`, `{isActive && <Badge />}`.
5. **`style` takes an object** — `style={{ color: "red", fontSize: 16 }}` (not a string).

### Conditional Rendering

```jsx
// && operator (short-circuit)
{isLoggedIn && <Dashboard />}

// Ternary
{isLoggedIn ? <Dashboard /> : <Login />}

// Early return
function Page({ user }) {
    if (!user) return <Login />;
    return <Dashboard user={user} />;
}
```

**Gotcha with `&&`:** `{count && <List />}` renders `0` when count is 0 (because `0` is a valid React child). Fix:
`{count > 0 && <List />}`.

### Lists and Keys

```jsx
const items = users.map(user => (
    <li key={user.id}>{user.name}</li>
));
```

**Why keys matter:**
- Keys help React identify which items changed, were added, or removed during reconciliation.
- Must be **stable, unique among siblings, and predictable** (not random).
- **Never use array index as key** if the list can be reordered, filtered, or items inserted — it causes bugs with
  component state and incorrect DOM reuse.
- Keys are not passed as a prop to the component — they are consumed by React internally.

## Components

### Function Components (Standard)

```tsx
interface UserCardProps {
    name: string;
    age: number;
    role?: string; // optional
}

function UserCard({ name, age, role = "user" }: UserCardProps) {
    return (
        <div>
            <h2>{name}</h2>
            <p>Age: {age}, Role: {role}</p>
        </div>
    );
}
```

### Class Components (Legacy)

Still found in older codebases and error boundaries:

```tsx
class UserCard extends React.Component<UserCardProps, UserCardState> {
    state: UserCardState = { expanded: false };

    render() {
        return <div>{this.props.name}</div>;
    }
}
```

Class components are **not recommended** for new code. Function components + hooks cover all use cases except
`componentDidCatch` / `getDerivedStateFromError` (error boundaries).

## Props

Props are **read-only** — a component must never modify its own props.

```tsx
// Children prop
function Card({ children, title }: { children: React.ReactNode; title: string }) {
    return (
        <div className="card">
            <h3>{title}</h3>
            {children}
        </div>
    );
}

<Card title="Profile">
    <p>Content goes here</p>
</Card>
```

### `React.ReactNode` vs `React.ReactElement` vs `JSX.Element`

| Type                | Includes                                         | Use case           |
|---------------------|--------------------------------------------------|--------------------|
| `React.ReactNode`   | Elements, strings, numbers, booleans, null, undefined, arrays | `children` prop    |
| `React.ReactElement`| Only React elements (the return of `createElement`) | Strict element prop|
| `JSX.Element`       | Same as `ReactElement` but inferred from JSX      | Return types       |

## Virtual DOM and Reconciliation

### What is the Virtual DOM?

A lightweight **in-memory representation** of the real DOM. React components return **React elements** (plain JS objects
describing the UI), not actual DOM nodes.

```javascript
// A React element is just an object:
{
    type: "div",
    props: {
        className: "card",
        children: [
            { type: "h1", props: { children: "Title" } },
            { type: "p",  props: { children: "Body" } },
        ]
    }
}
```

### Reconciliation Algorithm (Diffing)

When state changes, React:

1. **Renders** — calls the component function to produce a new element tree (Virtual DOM).
2. **Diffs** — compares the new tree with the previous tree.
3. **Commits** — applies only the **minimal set of changes** to the real DOM.

**Diffing heuristics (O(n) instead of O(n³)):**

1. **Different element types** → tear down the old tree and build a new one. `<div>` → `<span>` replaces the entire
   subtree (including children and state).
2. **Same element type** → keep the DOM node, update only the changed attributes. Recursively diff children.
3. **Lists with keys** → use keys to match old and new children. Without keys, React re-renders all items when order
   changes.

### Render ≠ DOM Update

"Rendering" in React means **calling the component function**, not updating the DOM. A component can re-render many
times without any DOM changes if the output hasn't changed. This distinction is important for performance discussions.

## React Fiber

Fiber is the **reconciliation engine** introduced in React 16. It replaced the old synchronous, recursive reconciler
with an incremental one.

Key concepts:
- Each component instance is represented by a **Fiber node** (a mutable data structure, unlike elements which are
  immutable).
- Work is broken into **units** — React can pause, prioritize, and resume work.
- Enables **concurrent features** (React 18+): transitions, Suspense, selective hydration.
- Two phases:
  - **Render phase** (interruptible) — calculates changes, calls component functions.
  - **Commit phase** (synchronous) — applies DOM mutations, runs `useLayoutEffect`.

## Component Lifecycle (Function Components)

```
Mount:    Component function called → DOM updated → useEffect runs
Update:   State/props change → function called → DOM updated → useEffect cleanup → useEffect runs
Unmount:  useEffect cleanup runs → component removed from DOM
```

See [Hooks](../hooks/hooks-in-depth.md) for details on `useEffect`.

## Common Interview Questions

1. **What is JSX?** — A syntax extension that compiles to `React.createElement()` (or `jsx()` in React 17+). It
   produces React elements — plain JS objects describing the UI tree.
2. **What is the Virtual DOM?** — An in-memory representation of the real DOM. React diffs the old and new virtual
   trees and applies the minimum necessary changes to the real DOM.
3. **How does reconciliation work?** — React compares element types: different type → replace subtree; same type →
   update attributes and recurse into children. Keys help identify list items.
4. **Why are keys important?** — Keys let React track which list items were added, removed, or moved. Without proper
   keys, React may reuse DOM nodes incorrectly, causing state bugs.
5. **What is React Fiber?** — The reconciliation engine (React 16+) that breaks rendering work into incremental
   units, allowing React to pause and prioritize work. Enables concurrent features.
6. **Function vs class components?** — Function components with hooks are the modern standard. Class components are
   legacy. The only feature exclusive to classes is error boundaries (`componentDidCatch`).

## Related

- [Hooks in Depth](../hooks/hooks-in-depth.md) — `useState`, `useEffect`, lifecycle with hooks
- [Performance Optimization](../performance/performance-optimization.md) — `React.memo`, memoization
- [Patterns](../patterns/component-patterns.md) — composition, HOC, render props

## Resources

- [React Docs — Describing the UI](https://react.dev/learn/describing-the-ui)
- [React Docs — Preserving and Resetting State](https://react.dev/learn/preserving-and-resetting-state)
- [React Docs — Reconciliation](https://legacy.reactjs.org/docs/reconciliation.html)
- [Andrew Clark — React Fiber Architecture](https://github.com/acdlite/react-fiber-architecture)
