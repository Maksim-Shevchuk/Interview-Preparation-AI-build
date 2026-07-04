# Performance Optimization

React is fast by default, but at scale, unnecessary re-renders and large bundles become real problems. Interviewers test
whether you understand **what** causes re-renders and **when** to optimize.

## When Does a Component Re-render?

A component re-renders when:

1. **Its state changes** (`useState`, `useReducer`).
2. **Its parent re-renders** (props may or may not have changed — React re-renders children by default).
3. **A context it consumes changes** (`useContext`).

**Re-render ≠ DOM update.** Re-rendering calls the component function and produces a new Virtual DOM tree. React then
diffs it — if nothing changed, no DOM mutations happen. Still, the function call + diffing has a cost for complex trees.

## `React.memo`

Wraps a component to **skip re-rendering** when props haven't changed (shallow comparison):

```tsx
interface ItemProps {
    title: string;
    onDelete: (id: string) => void;
}

const Item = React.memo(function Item({ title, onDelete }: ItemProps) {
    console.log("Item rendered");
    return <div>{title} <button onClick={() => onDelete(title)}>X</button></div>;
});
```

**`React.memo` is wasted if:**
- Props include **new object/array/function references** every render (not stable).
- The component is trivially cheap to render.
- The component almost always receives different props.

### Custom Comparison

```tsx
const Item = React.memo(
    function Item({ data, onClick }: ItemProps) { /* ... */ },
    (prevProps, nextProps) => prevProps.data.id === nextProps.data.id, // true = skip render
);
```

## `useMemo` and `useCallback` — Stabilizing References

```tsx
function UserList({ users, filter }: Props) {
    // ✅ Expensive computation — memoize the result
    const filteredUsers = useMemo(
        () => users.filter(u => u.name.includes(filter)).sort(byName),
        [users, filter],
    );

    // ✅ Stable function reference for memo'd child
    const handleDelete = useCallback((id: string) => {
        setUsers(prev => prev.filter(u => u.id !== id));
    }, []);

    return filteredUsers.map(user => (
        <MemoizedItem key={user.id} user={user} onDelete={handleDelete} />
    ));
}
```

### When NOT to Memoize

- Simple/cheap computations.
- Values that change on every render anyway (memoization overhead > savings).
- Primitive props to non-memo'd components.

**Guideline:** Profile first, memoize second. React DevTools Profiler shows which components re-render and how long
they take.

## Memoization Deep Dive

Interviewers like to go deep into how memoization works internally. Here's what's under the hood.

### How `React.memo` Works Internally

`React.memo` wraps a component in a special **fiber type** (`REACT_MEMO_TYPE`). During reconciliation, before calling
the component function, React runs a **shallow comparison** of previous and next props.

```tsx
// React.memo signature
function memo<P>(
    Component: React.FC<P>,
    arePropsEqual?: (prevProps: P, nextProps: P) => boolean
): React.NamedExoticComponent<P>;
```

**Parameters:**
1. `Component` — the function component to memoize.
2. `arePropsEqual` (optional) — custom comparison function. Returns `true` if props are equal (skip render), `false`
   to re-render. If omitted, React uses **shallow equality**.

### What is Shallow Comparison?

React compares each prop key-value pair using `Object.is()` (same as `===` except for `NaN` and `-0`):

```javascript
// Simplified shallowEqual implementation (React's actual logic)
function shallowEqual(objA, objB) {
    // Same reference → equal
    if (Object.is(objA, objB)) return true;

    // Not both objects → not equal
    if (typeof objA !== "object" || objA === null ||
        typeof objB !== "object" || objB === null) return false;

    const keysA = Object.keys(objA);
    const keysB = Object.keys(objB);

    // Different number of keys → not equal
    if (keysA.length !== keysB.length) return false;

    // Compare each key with Object.is (NO deep comparison)
    for (const key of keysA) {
        if (!Object.hasOwn(objB, key) || !Object.is(objA[key], objB[key])) {
            return false;
        }
    }

    return true;
}
```

**What shallow comparison catches:**

```tsx
// ✅ Primitives — compared by value
<Item title="hello" count={5} />  // same primitives → skip render

// ❌ New object reference every render — different reference → re-renders
<Item style={{ color: "red" }} />  // { color: "red" } !== { color: "red" }

// ❌ New array reference
<Item items={filteredItems} />     // new array from .filter() each render

// ❌ New function reference (inline arrow)
<Item onClick={() => doSomething()} />  // new function each render

// ✅ Stable function reference (useCallback)
const handleClick = useCallback(() => doSomething(), []);
<Item onClick={handleClick} />     // same reference → skip render
```

### How `useMemo` Works Internally

React stores memoized values in the component's **fiber node** (in the `memoizedState` linked list — same structure
used by all hooks).

```typescript
// Simplified useMemo internal logic
function useMemo<T>(factory: () => T, deps: DependencyList): T {
    const hook = getOrCreateHook(); // from the fiber's hook linked list

    const prevDeps = hook.memoizedState?.[1];

    if (prevDeps && areDepsEqual(deps, prevDeps)) {
        return hook.memoizedState[0]; // return cached value
    }

    const value = factory(); // recompute
    hook.memoizedState = [value, deps]; // cache [value, deps]
    return value;
}

// Dependency comparison — also uses Object.is per element
function areDepsEqual(nextDeps, prevDeps) {
    for (let i = 0; i < nextDeps.length; i++) {
        if (!Object.is(nextDeps[i], prevDeps[i])) return false;
    }
    return true;
}
```

**Key insights:**
- `useMemo` stores `[cachedValue, deps]` in the hook slot.
- On re-render, it compares each dependency with `Object.is`. If all match → return cached value, skip factory call.
- If any dep changed → call factory, store new value and deps.
- **`useMemo` does NOT guarantee the value won't be recomputed** — React may discard cached values under memory
  pressure (documented in React docs). Treat it as a performance hint, not a semantic guarantee.

### How `useCallback` Works Internally

`useCallback(fn, deps)` is literally `useMemo(() => fn, deps)`:

```typescript
// Simplified
function useCallback<T extends Function>(callback: T, deps: DependencyList): T {
    return useMemo(() => callback, deps);
}
```

It caches the **function reference itself**, not the function's return value.

### Cost of Memoization

Memoization is not free:

| Cost                          | Description                                              |
|-------------------------------|----------------------------------------------------------|
| **Memory**                    | Cached value + deps array stored in fiber                |
| **Comparison overhead**       | `Object.is` per dep on every render                      |
| **Garbage collection**        | Old values retained until deps change                    |
| **Code complexity**           | More imports, dependency arrays to maintain              |

**When the cost exceeds the benefit:**
- Cheap computations (string concatenation, simple math).
- Deps that change on every render (memoization never hits cache).
- Components that always receive new props anyway.

### Decision Flowchart

```
Is the component expensive to render or does it render large subtrees?
├── No → Don't memoize. Re-render is cheap.
└── Yes → Are its props stable (primitives, memoized references)?
    ├── Yes → React.memo will work. Add it.
    └── No → Can you stabilize props with useMemo/useCallback?
        ├── Yes → Do it, then add React.memo.
        └── No (props always change) → React.memo is wasted. Consider restructuring.
```

## React Compiler (React 19+)

React Compiler (formerly React Forget) **automatically memoizes** components and hooks at build time, potentially making
manual `useMemo`, `useCallback`, and `React.memo` unnecessary. Still being adopted — but the future direction of React.

## Keys and Reconciliation

```tsx
// ❌ Index as key — state bugs when list order changes
{items.map((item, index) => <Item key={index} item={item} />)}

// ✅ Stable, unique key
{items.map(item => <Item key={item.id} item={item} />)}

// Key reset trick — change key to force remount (reset internal state)
<Form key={selectedUserId} user={selectedUser} />
```

## Code Splitting and Lazy Loading

### `React.lazy` + `Suspense`

```tsx
// Split at the route level
const Dashboard = lazy(() => import("./pages/Dashboard"));
const Settings = lazy(() => import("./pages/Settings"));

function App() {
    return (
        <Suspense fallback={<Spinner />}>
            <Routes>
                <Route path="/dashboard" element={<Dashboard />} />
                <Route path="/settings" element={<Settings />} />
            </Routes>
        </Suspense>
    );
}
```

Webpack/Vite creates separate chunks for each `import()` — loaded on demand.

### Dynamic Import with Named Exports

```tsx
const Chart = lazy(() =>
    import("./components/Chart").then(module => ({ default: module.Chart }))
);
```

## Virtualization (Windowing)

Render only the **visible items** in a large list. Essential for lists with 1000+ items.

```tsx
import { FixedSizeList } from "react-window";

function VirtualList({ items }: { items: Item[] }) {
    return (
        <FixedSizeList
            height={600}
            width="100%"
            itemCount={items.length}
            itemSize={50}
        >
            {({ index, style }) => (
                <div style={style}>{items[index].name}</div>
            )}
        </FixedSizeList>
    );
}
```

Libraries: `react-window` (lightweight), `@tanstack/react-virtual` (headless).

## Avoiding Unnecessary Re-renders — Checklist

| Problem                                    | Solution                                                |
|--------------------------------------------|---------------------------------------------------------|
| Parent re-render cascades to children      | `React.memo` on expensive children                      |
| New function reference every render        | `useCallback` for handlers passed to memo'd children    |
| New object/array created every render      | `useMemo` to stabilize the reference                    |
| Inline object in JSX (`style={{...}}`)     | Extract to a constant or `useMemo`                      |
| Context value changes too often            | Split context, `useMemo` on value, use external store   |
| Component renders but DOM doesn't change   | Usually harmless — optimize only if slow                |
| Large list rendering                       | Virtualization (`react-window`)                         |
| Large bundle size                          | Code splitting with `lazy()` / dynamic `import()`       |

## `useTransition` and `useDeferredValue`

For keeping the UI responsive during heavy state updates (React 18+):

```tsx
const [isPending, startTransition] = useTransition();

function handleFilter(value: string) {
    setInput(value);                       // urgent: update input immediately
    startTransition(() => {
        setFilteredItems(filterItems(value)); // non-urgent: can be interrupted
    });
}
```

## Measuring Performance

### React DevTools Profiler

- **Flamegraph** — see which components rendered and how long each took.
- **Why did this render?** — enable in settings to see the cause (state change, props change, parent re-render).
- **Highlight updates** — visual overlay of re-rendering components.

### Browser DevTools

- **Performance tab** — record and inspect frame timings, layout shifts, long tasks.
- **Lighthouse** — audit for performance, accessibility, SEO.

## Common Interview Questions

1. **What causes a React component to re-render?** — State change, parent re-render, or context change. NOT prop
   changes directly — parent re-render triggers it regardless.
2. **What is `React.memo` and what parameters does it accept?** — `memo(Component, arePropsEqual?)`. Wraps a
   component to skip re-render when props haven't changed. First param: the component. Optional second param:
   custom comparison function `(prevProps, nextProps) => boolean` (return `true` to skip render). Default comparison
   is shallow equality via `Object.is` on each prop.
3. **What is shallow comparison? How does it differ from deep?** — Shallow compares each key-value pair with
   `Object.is` (reference equality for objects). `{ a: 1 }` vs `{ a: 1 }` = NOT equal (different references).
   Deep comparison would recursively compare nested properties — React does NOT do this for performance reasons.
4. **`useMemo` vs `useCallback`?** — `useMemo` memoizes a **value** (calls factory function, caches result).
   `useCallback` memoizes a **function reference** (is literally `useMemo(() => fn, deps)`). Both compare
   dependencies with `Object.is` per element.
5. **How does `useMemo` work internally?** — Stores `[cachedValue, deps]` in the fiber's hook linked list. On
   re-render, compares each dep via `Object.is`. All match → return cached value. Any changed → call factory,
   cache new result. React may drop the cache under memory pressure — it's a hint, not a guarantee.
6. **When should you NOT memoize?** — Cheap computations, deps that change every render (cache never hits),
   components that always receive new props. Memoization has overhead: memory for cached values, `Object.is`
   comparison per dep on every render.
7. **What is code splitting?** — Breaking the bundle into smaller chunks loaded on demand. `React.lazy` + `Suspense`
   for component-level splitting, dynamic `import()` for utility code.
8. **What is virtualization?** — Rendering only the visible portion of a large list. Keeps DOM node count low.
   Libraries: `react-window`, `@tanstack/react-virtual`.

## Related

- [Components, JSX, and Virtual DOM](../fundamentals/components-jsx-and-virtual-dom.md) — reconciliation, Fiber
- [Hooks in Depth](../hooks/hooks-in-depth.md) — `useMemo`, `useCallback`, `useTransition`
- [State Management](../state-management/state-management.md) — selectors, avoiding context re-renders

## Resources

- [React Docs — Optimizing Performance](https://react.dev/learn/render-and-commit)
- [React Docs — React.memo](https://react.dev/reference/react/memo)
- [Kent C. Dodds — Before You memo()](https://kentcdodds.com/blog/before-you-memo)
