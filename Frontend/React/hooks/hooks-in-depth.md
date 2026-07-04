# Hooks in Depth

Hooks let function components use state, side effects, and other React features. This note covers all major hooks,
their rules, and common pitfalls — the most frequently tested React topic on interviews.

## Rules of Hooks

1. **Only call hooks at the top level** — never inside loops, conditions, or nested functions. React relies on the
   **call order** being the same on every render.
2. **Only call hooks from React functions** — function components or custom hooks. Not from regular JS functions.

```jsx
// ❌ Violates rule 1
if (isLoggedIn) {
    const [user, setUser] = useState(null); // breaks hook order on re-render
}

// ✅ Correct — always call, use the value conditionally
const [user, setUser] = useState(null);
if (isLoggedIn) {
    // use `user` here
}
```

## `useState`

```tsx
const [count, setCount] = useState(0);

// Direct update
setCount(5);

// Functional update — use when new state depends on previous state
setCount(prev => prev + 1);

// Lazy initialization — expensive initial value computed once
const [data, setData] = useState(() => computeExpensiveValue());
```

**Key behaviors:**
- `setState` is **asynchronous** — the value doesn't change until the next render.
- React **batches** multiple `setState` calls within the same event handler into a single re-render (React 18+: batches
  everywhere, including promises and setTimeout).
- `setState` with the **same value** (by `Object.is`) skips re-render.
- For objects/arrays: always create a **new reference** — React compares by reference, not deep equality.

```tsx
// ❌ Mutating state directly — React won't detect the change
user.name = "Bob";
setUser(user); // same reference → no re-render

// ✅ Create a new object
setUser({ ...user, name: "Bob" });
setItems(prev => [...prev, newItem]);
setItems(prev => prev.filter(item => item.id !== id));
```

## `useEffect`

Runs **side effects** after render: data fetching, subscriptions, DOM manipulation.

```tsx
useEffect(() => {
    // Effect runs after render

    return () => {
        // Cleanup — runs before next effect and on unmount
    };
}, [dependencies]);
```

### Dependency Array

| Dependency array  | When effect runs                          | Equivalent lifecycle      |
|-------------------|-------------------------------------------|---------------------------|
| `undefined` (omitted) | After **every** render                | —                         |
| `[]` (empty)      | Only after **first** render (mount)       | `componentDidMount`       |
| `[a, b]`          | After mount + when `a` or `b` changes    | `componentDidUpdate` (partial) |

**Cleanup function** runs:
- Before the next execution of the effect (when deps change).
- On component unmount.

### Common Patterns

```tsx
// Data fetching
useEffect(() => {
    let cancelled = false; // prevent state update on unmounted component

    async function fetchData() {
        const res = await fetch(`/api/users/${userId}`);
        const data = await res.json();
        if (!cancelled) setUser(data);
    }

    fetchData();
    return () => { cancelled = true; };
}, [userId]);

// Event listener
useEffect(() => {
    const handler = (e: KeyboardEvent) => {
        if (e.key === "Escape") closeModal();
    };
    window.addEventListener("keydown", handler);
    return () => window.removeEventListener("keydown", handler);
}, [closeModal]);

// Timer
useEffect(() => {
    const id = setInterval(() => tick(), 1000);
    return () => clearInterval(id);
}, []);
```

### Common Mistakes

```tsx
// ❌ Object/array as dependency — new reference every render → infinite loop
useEffect(() => {
    fetch(options.url); // options is created inline → always "new"
}, [options]); // runs every render!

// ✅ Destructure to primitives
useEffect(() => {
    fetch(url);
}, [url]);

// ❌ Missing dependency
const [count, setCount] = useState(0);
useEffect(() => {
    const id = setInterval(() => setCount(count + 1), 1000); // stale closure!
    return () => clearInterval(id);
}, []); // count is missing

// ✅ Use functional update
useEffect(() => {
    const id = setInterval(() => setCount(prev => prev + 1), 1000);
    return () => clearInterval(id);
}, []); // no dependency on count needed
```

## `useRef`

Holds a **mutable value** that persists across renders without causing re-renders when changed.

```tsx
// DOM reference
const inputRef = useRef<HTMLInputElement>(null);
useEffect(() => {
    inputRef.current?.focus(); // access DOM node
}, []);
return <input ref={inputRef} />;

// Mutable value (no re-render on change)
const renderCount = useRef(0);
renderCount.current += 1; // persists, doesn't trigger render

// Store previous value
function usePrevious<T>(value: T): T | undefined {
    const ref = useRef<T>();
    useEffect(() => {
        ref.current = value;
    });
    return ref.current;
}
```

## `useMemo` and `useCallback`

### `useMemo` — memoize a computed value

```tsx
const sortedItems = useMemo(
    () => items.slice().sort((a, b) => a.name.localeCompare(b.name)),
    [items], // recompute only when items changes
);
```

### `useCallback` — memoize a function reference

```tsx
const handleClick = useCallback((id: string) => {
    setSelected(id);
}, []);

// Equivalent to:
const handleClick = useMemo(() => (id: string) => {
    setSelected(id);
}, []);
```

**When to use:**
- `useMemo` — expensive computations, derived data, referential equality for objects passed as props/deps.
- `useCallback` — functions passed to memoized children (`React.memo`) or used in dependency arrays.

**When NOT to use:** Don't memoize everything. Memoization has its own cost (memory + comparison). Only use when
there's a measurable performance problem or to stabilize a reference in a dependency array.

See [Performance Optimization](../performance/performance-optimization.md) for more details.

## `useReducer`

Alternative to `useState` for complex state logic:

```tsx
type State = { count: number; step: number };
type Action =
    | { type: "INCREMENT" }
    | { type: "DECREMENT" }
    | { type: "SET_STEP"; payload: number };

function reducer(state: State, action: Action): State {
    switch (action.type) {
        case "INCREMENT": return { ...state, count: state.count + state.step };
        case "DECREMENT": return { ...state, count: state.count - state.step };
        case "SET_STEP":  return { ...state, step: action.payload };
    }
}

function Counter() {
    const [state, dispatch] = useReducer(reducer, { count: 0, step: 1 });

    return (
        <>
            <p>{state.count}</p>
            <button onClick={() => dispatch({ type: "INCREMENT" })}>+</button>
        </>
    );
}
```

**`useState` vs `useReducer`:**
- `useState` — simple values, few transitions.
- `useReducer` — multiple related state values, complex transitions, state logic you want to test in isolation.

## `useContext`

Consumes a React Context value:

```tsx
const ThemeContext = createContext<"light" | "dark">("light");

function App() {
    return (
        <ThemeContext.Provider value="dark">
            <Page />
        </ThemeContext.Provider>
    );
}

function Button() {
    const theme = useContext(ThemeContext); // "dark"
    return <button className={theme}>Click</button>;
}
```

See [State Management](../state-management/state-management.md) for Context patterns and alternatives.

## `useLayoutEffect`

Same API as `useEffect`, but fires **synchronously after DOM mutations, before the browser paints**. Use for DOM
measurements and synchronous visual updates.

```tsx
useLayoutEffect(() => {
    const { height } = ref.current.getBoundingClientRect();
    setHeight(height); // no visual flicker — runs before paint
}, []);
```

| Hook              | When it runs                         | Use case                     |
|-------------------|--------------------------------------|------------------------------|
| `useEffect`       | After paint (asynchronous)           | Data fetching, subscriptions |
| `useLayoutEffect` | Before paint (synchronous)           | DOM measurements, tooltips   |

## React 18+ Hooks

### `useTransition`

Marks a state update as **non-urgent** — keeps the UI responsive during heavy renders:

```tsx
const [isPending, startTransition] = useTransition();

function handleSearch(query: string) {
    setInputValue(query);            // urgent — update input immediately

    startTransition(() => {
        setSearchResults(filter(query)); // non-urgent — can be interrupted
    });
}

return isPending ? <Spinner /> : <Results data={searchResults} />;
```

### `useDeferredValue`

Defers re-rendering of a value — similar to debouncing but integrated with React's scheduler:

```tsx
const deferredQuery = useDeferredValue(query);
// deferredQuery lags behind query during heavy renders
// React re-renders with the old value first, then updates in the background
```

### `useId`

Generates a unique ID stable across server and client (for SSR hydration):

```tsx
function Input({ label }: { label: string }) {
    const id = useId();
    return (
        <>
            <label htmlFor={id}>{label}</label>
            <input id={id} />
        </>
    );
}
```

## Custom Hooks

Extract reusable logic into functions prefixed with `use`:

```tsx
function useDebounce<T>(value: T, delay: number): T {
    const [debounced, setDebounced] = useState(value);

    useEffect(() => {
        const timer = setTimeout(() => setDebounced(value), delay);
        return () => clearTimeout(timer);
    }, [value, delay]);

    return debounced;
}

// Usage
const debouncedSearch = useDebounce(searchTerm, 300);
```

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
```

## Common Interview Questions

1. **What are the rules of hooks?** — Call at the top level only (no conditions/loops). Call only from React
   functions. React relies on call order to associate hooks with state.
2. **`useEffect` cleanup — when does it run?** — Before the next effect execution (when deps change) and on
   unmount. Used for unsubscribing, clearing timers, aborting fetches.
3. **`useEffect` vs `useLayoutEffect`?** — `useEffect` runs asynchronously after paint. `useLayoutEffect` runs
   synchronously before paint. Use `useLayoutEffect` for DOM measurements to avoid flicker.
4. **When to use `useMemo`/`useCallback`?** — When you have expensive computations, or need a stable reference for
   dependency arrays or `React.memo` children. Don't over-use — memoization has a cost.
5. **What is a stale closure in hooks?** — When an effect or callback captures an outdated value from a previous
   render. Fix with functional updates (`setCount(prev => prev + 1)`) or adding the value to the dependency array.
6. **`useState` vs `useReducer`?** — `useState` for simple state. `useReducer` for complex state with multiple
   transitions or when you want to separate state logic from the component.

## Related

- [Components, JSX, and Virtual DOM](../fundamentals/components-jsx-and-virtual-dom.md) — component lifecycle
- [Performance Optimization](../performance/performance-optimization.md) — `React.memo`, memoization strategy
- [State Management](../state-management/state-management.md) — `useContext`, global state

## Resources

- [React Docs — Hooks Reference](https://react.dev/reference/react/hooks)
- [React Docs — You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect)
- [Dan Abramov — A Complete Guide to useEffect](https://overreacted.io/a-complete-guide-to-useeffect/)
