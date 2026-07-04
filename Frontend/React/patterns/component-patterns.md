# Component Patterns

Reusable patterns for structuring React components. Interviewers test whether you can recognize the right pattern for a
given problem and explain the trade-offs.

## Composition vs Inheritance

React strongly favors **composition** over inheritance. You never need class inheritance for UI components.

```tsx
// ✅ Composition via children
function Card({ children, title }: { children: React.ReactNode; title: string }) {
    return (
        <div className="card">
            <h3>{title}</h3>
            <div className="card-body">{children}</div>
        </div>
    );
}

<Card title="Profile">
    <Avatar user={user} />
    <UserInfo user={user} />
</Card>
```

### Slots Pattern (Named Children)

```tsx
interface LayoutProps {
    header: React.ReactNode;
    sidebar: React.ReactNode;
    children: React.ReactNode;
}

function Layout({ header, sidebar, children }: LayoutProps) {
    return (
        <div className="layout">
            <header>{header}</header>
            <aside>{sidebar}</aside>
            <main>{children}</main>
        </div>
    );
}

<Layout
    header={<NavBar />}
    sidebar={<Menu items={menuItems} />}
>
    <Dashboard />
</Layout>
```

**Benefit:** Parent controls what goes where, child components remain decoupled.

## Controlled vs Uncontrolled Components

### Controlled — parent owns the state

```tsx
function ControlledInput() {
    const [value, setValue] = useState("");

    return (
        <input
            value={value}                           // React controls the value
            onChange={e => setValue(e.target.value)}  // every change goes through state
        />
    );
}
```

### Uncontrolled — DOM owns the state

```tsx
function UncontrolledInput() {
    const inputRef = useRef<HTMLInputElement>(null);

    function handleSubmit() {
        console.log(inputRef.current?.value); // read from DOM on demand
    }

    return <input ref={inputRef} defaultValue="initial" />;
}
```

| Aspect           | Controlled                              | Uncontrolled                        |
|------------------|-----------------------------------------|-------------------------------------|
| State lives in   | React (`useState`)                      | DOM                                 |
| Read value       | From state variable                     | From `ref.current.value`            |
| Validation       | On every change (instant feedback)      | On submit (batch validation)        |
| Use when         | Dynamic forms, conditional logic        | Simple forms, third-party DOM libs  |

**Rule of thumb:** Default to controlled. Use uncontrolled for file inputs (`<input type="file">`) or integrating with
non-React code.

## Higher-Order Components (HOC)

A function that takes a component and returns an enhanced component:

```tsx
function withAuth<P extends object>(WrappedComponent: React.ComponentType<P>) {
    return function AuthenticatedComponent(props: P) {
        const { user } = useAuth();

        if (!user) return <Navigate to="/login" />;
        return <WrappedComponent {...props} />;
    };
}

const ProtectedDashboard = withAuth(Dashboard);
```

**Pros:** Reusable cross-cutting logic (auth, logging, theming).
**Cons:** Wrapper hell, props collision, harder to debug, poor TypeScript support.
**Status:** Mostly replaced by **custom hooks**. Still seen in older codebases and libraries (React Router's
`withRouter`, Redux's `connect`).

## Render Props

A component that receives a **function as prop** (or children) to control what it renders:

```tsx
interface MouseTrackerProps {
    children: (pos: { x: number; y: number }) => React.ReactNode;
}

function MouseTracker({ children }: MouseTrackerProps) {
    const [pos, setPos] = useState({ x: 0, y: 0 });

    useEffect(() => {
        const handler = (e: MouseEvent) => setPos({ x: e.clientX, y: e.clientY });
        window.addEventListener("mousemove", handler);
        return () => window.removeEventListener("mousemove", handler);
    }, []);

    return <>{children(pos)}</>;
}

// Usage
<MouseTracker>
    {({ x, y }) => <p>Mouse: {x}, {y}</p>}
</MouseTracker>
```

**Status:** Also mostly replaced by custom hooks. The same logic as a hook:

```tsx
function useMousePosition() {
    const [pos, setPos] = useState({ x: 0, y: 0 });
    useEffect(() => { /* same listener */ }, []);
    return pos;
}
```

## Compound Components

A group of components that work together, sharing implicit state. Like `<select>` + `<option>` in HTML.

```tsx
interface TabsContextValue {
    activeTab: string;
    setActiveTab: (id: string) => void;
}

const TabsContext = createContext<TabsContextValue | null>(null);

function Tabs({ children, defaultTab }: { children: React.ReactNode; defaultTab: string }) {
    const [activeTab, setActiveTab] = useState(defaultTab);
    return (
        <TabsContext.Provider value={{ activeTab, setActiveTab }}>
            <div className="tabs">{children}</div>
        </TabsContext.Provider>
    );
}

function TabList({ children }: { children: React.ReactNode }) {
    return <div className="tab-list" role="tablist">{children}</div>;
}

function Tab({ id, children }: { id: string; children: React.ReactNode }) {
    const { activeTab, setActiveTab } = useContext(TabsContext)!;
    return (
        <button
            role="tab"
            aria-selected={activeTab === id}
            onClick={() => setActiveTab(id)}
            className={activeTab === id ? "active" : ""}
        >
            {children}
        </button>
    );
}

function TabPanel({ id, children }: { id: string; children: React.ReactNode }) {
    const { activeTab } = useContext(TabsContext)!;
    if (activeTab !== id) return null;
    return <div role="tabpanel">{children}</div>;
}

// Attach sub-components
Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;

// Usage — clean, declarative API
<Tabs defaultTab="profile">
    <Tabs.List>
        <Tabs.Tab id="profile">Profile</Tabs.Tab>
        <Tabs.Tab id="settings">Settings</Tabs.Tab>
    </Tabs.List>
    <Tabs.Panel id="profile"><ProfilePage /></Tabs.Panel>
    <Tabs.Panel id="settings"><SettingsPage /></Tabs.Panel>
</Tabs>
```

**Use case:** UI component libraries (tabs, accordions, dropdowns, menus).

## Container / Presentational Pattern

Separate **logic** (data fetching, state) from **presentation** (rendering):

```tsx
// Presentational — pure UI, receives data via props
function UserCard({ user, onFollow }: { user: User; onFollow: () => void }) {
    return (
        <div>
            <h2>{user.name}</h2>
            <button onClick={onFollow}>Follow</button>
        </div>
    );
}

// Container — handles data/logic
function UserCardContainer({ userId }: { userId: string }) {
    const { data: user } = useQuery({ queryKey: ["user", userId], queryFn: () => fetchUser(userId) });
    const followMutation = useMutation({ mutationFn: () => followUser(userId) });

    if (!user) return <Skeleton />;
    return <UserCard user={user} onFollow={() => followMutation.mutate()} />;
}
```

**Modern take:** With hooks, the separation often happens naturally — the custom hook is the "container" and the
component is the "presentation." Explicit Container/Presentational splitting is less common now but the principle
(separating concerns) remains important.

## Custom Hooks as the Universal Pattern

Most patterns above (HOC, render props, container logic) are now expressed as custom hooks:

```tsx
// Encapsulate logic
function useUser(userId: string) {
    const { data, isLoading, error } = useQuery({
        queryKey: ["user", userId],
        queryFn: () => fetchUser(userId),
    });
    return { user: data, isLoading, error };
}

// Encapsulate form logic
function useForm<T>(initialValues: T) {
    const [values, setValues] = useState(initialValues);
    const handleChange = useCallback((field: keyof T, value: T[keyof T]) => {
        setValues(prev => ({ ...prev, [field]: value }));
    }, []);
    const reset = useCallback(() => setValues(initialValues), [initialValues]);
    return { values, handleChange, reset };
}
```

## Error Boundaries

The **only** pattern that still requires a class component:

```tsx
class ErrorBoundary extends React.Component<
    { children: React.ReactNode; fallback: React.ReactNode },
    { hasError: boolean }
> {
    state = { hasError: false };

    static getDerivedStateFromError(): { hasError: boolean } {
        return { hasError: true };
    }

    componentDidCatch(error: Error, info: React.ErrorInfo) {
        console.error("Caught by ErrorBoundary:", error, info.componentStack);
    }

    render() {
        if (this.state.hasError) return this.props.fallback;
        return this.props.children;
    }
}

// Usage
<ErrorBoundary fallback={<p>Something went wrong</p>}>
    <Dashboard />
</ErrorBoundary>
```

Libraries like `react-error-boundary` provide a function-component-friendly wrapper.

## Pattern Selection Guide

| Problem                                       | Pattern                 |
|------------------------------------------------|-------------------------|
| Reuse layout structure                         | Composition / Slots     |
| Share stateful logic across components         | Custom Hook             |
| Group related components with shared state     | Compound Components     |
| Protect routes / add cross-cutting behavior    | HOC (or custom hook)    |
| Separate data logic from UI                    | Container/Presentational|
| Catch rendering errors                         | Error Boundary          |
| Flexible child rendering based on parent state | Render Props (rare now) |

## Common Interview Questions

1. **Composition vs inheritance in React?** — React uses composition (children, props, hooks). Inheritance is never
   needed for UI components. Composition is more flexible and avoids tight coupling.
2. **Controlled vs uncontrolled components?** — Controlled: React state drives the input value. Uncontrolled: DOM
   owns the state, accessed via ref. Default to controlled.
3. **What is a HOC?** — A function `(Component) => EnhancedComponent`. Adds behavior without modifying the original.
   Mostly replaced by hooks but still used in some libraries.
4. **What are compound components?** — Components that share implicit state via Context and are used together
   (like `<Tabs>` + `<Tab>` + `<TabPanel>`). Provide a clean declarative API.
5. **What is an Error Boundary?** — A class component that catches JavaScript errors in its child tree during
   rendering and displays a fallback UI. Cannot be a function component (no hook equivalent for `componentDidCatch`).
6. **HOC vs hooks for reusable logic?** — Hooks are simpler, more composable, better typed, and don't add wrapper
   components to the tree. Prefer hooks for new code.

## Related

- [Hooks in Depth](../hooks/hooks-in-depth.md) — custom hooks
- [State Management](../state-management/state-management.md) — Context, prop drilling alternatives
- [Components, JSX, and Virtual DOM](../fundamentals/components-jsx-and-virtual-dom.md) — component basics

## Resources

- [React Docs — Thinking in React](https://react.dev/learn/thinking-in-react)
- [React Docs — Reusing Logic with Custom Hooks](https://react.dev/learn/reusing-logic-with-custom-hooks)
- [Kent C. Dodds — Compound Components](https://kentcdodds.com/blog/compound-components-with-react-hooks)
