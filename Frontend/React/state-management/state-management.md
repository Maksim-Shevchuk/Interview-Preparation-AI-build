# State Management

When and how to manage state in React — from local `useState` to global stores. Interviewers test whether you understand
the trade-offs and can choose the right tool for the scale of the problem.

## State Categories

| Category         | Scope                          | Examples                         | Typical tool           |
|------------------|--------------------------------|----------------------------------|------------------------|
| **Local**        | Single component               | Form input, toggle, modal open   | `useState`, `useReducer` |
| **Lifted**       | Parent + few children          | Active tab, selected item        | Lift state up          |
| **Global/Shared**| Many unrelated components      | Auth user, theme, locale         | Context, Zustand, Redux|
| **Server**       | Backend data cache             | User list, product catalog       | React Query, SWR       |
| **URL**          | Shareable, bookmarkable state  | Filters, pagination, search      | URL params, `useSearchParams` |

**Rule of thumb:** Keep state as **local** as possible. Lift or globalize only when needed.

## Context API

### Creating and Providing

```tsx
interface AuthContextValue {
    user: User | null;
    login: (credentials: Credentials) => Promise<void>;
    logout: () => void;
}

const AuthContext = createContext<AuthContextValue | null>(null);

// Custom hook with safety check
function useAuth(): AuthContextValue {
    const ctx = useContext(AuthContext);
    if (!ctx) throw new Error("useAuth must be used within AuthProvider");
    return ctx;
}

function AuthProvider({ children }: { children: React.ReactNode }) {
    const [user, setUser] = useState<User | null>(null);

    const login = useCallback(async (credentials: Credentials) => {
        const user = await api.login(credentials);
        setUser(user);
    }, []);

    const logout = useCallback(() => setUser(null), []);

    const value = useMemo(() => ({ user, login, logout }), [user, login, logout]);

    return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}
```

### Context Performance Problem

**Every consumer re-renders when the context value changes — even if they only use a part of it.**

```tsx
// ❌ Problem: theme changes re-render components that only use auth
const AppContext = createContext({ theme: "light", user: null, locale: "en" });

// ✅ Fix 1: Split into separate contexts
<ThemeContext.Provider value={theme}>
    <AuthContext.Provider value={user}>
        <LocaleContext.Provider value={locale}>
            {children}
        </LocaleContext.Provider>
    </AuthContext.Provider>
</ThemeContext.Provider>

// ✅ Fix 2: Memoize the value object
const value = useMemo(() => ({ user, login, logout }), [user, login, logout]);
<AuthContext.Provider value={value}>
```

### When Context is Enough

- Small-to-medium apps.
- State that changes **infrequently** (theme, auth, locale).
- State consumed by **many components** but updated in **few places**.

### When Context is NOT Enough

- **Frequently changing state** (every keystroke, animation frames) — Context re-renders all consumers.
- **Large state objects** — no built-in selector mechanism (unlike Redux or Zustand).
- **Complex update logic** — better suited for reducers or external stores.

## Redux / Redux Toolkit (RTK)

The most established state management library. Redux Toolkit is the modern, opinionated way to write Redux.

### Core Concepts

```
Action → Dispatch → Reducer → Store → UI
```

- **Store** — single source of truth (one global state object).
- **Action** — plain object `{ type: string, payload?: any }` describing what happened.
- **Reducer** — pure function `(state, action) => newState`.
- **Dispatch** — sends an action to the store.
- **Selector** — function that extracts a slice of state.

### Redux Toolkit Slice

```tsx
import { createSlice, PayloadAction } from "@reduxjs/toolkit";

interface CounterState { value: number }

const counterSlice = createSlice({
    name: "counter",
    initialState: { value: 0 } as CounterState,
    reducers: {
        increment(state) {
            state.value += 1; // Immer allows "mutation" — produces immutable update under the hood
        },
        incrementBy(state, action: PayloadAction<number>) {
            state.value += action.payload;
        },
        reset: () => ({ value: 0 }),
    },
});

export const { increment, incrementBy, reset } = counterSlice.actions;
export default counterSlice.reducer;
```

### Store Setup

```tsx
import { configureStore } from "@reduxjs/toolkit";

const store = configureStore({
    reducer: {
        counter: counterReducer,
        // other slices...
    },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

### Usage in Components

```tsx
import { useSelector, useDispatch } from "react-redux";

function Counter() {
    const count = useSelector((state: RootState) => state.counter.value);
    const dispatch = useDispatch();

    return (
        <div>
            <p>{count}</p>
            <button onClick={() => dispatch(increment())}>+</button>
        </div>
    );
}
```

### RTK Query (Server State)

```tsx
import { createApi, fetchBaseQuery } from "@reduxjs/toolkit/query/react";

const api = createApi({
    baseQuery: fetchBaseQuery({ baseUrl: "/api" }),
    endpoints: (builder) => ({
        getUsers: builder.query<User[], void>({
            query: () => "/users",
        }),
        createUser: builder.mutation<User, CreateUserDto>({
            query: (body) => ({ url: "/users", method: "POST", body }),
        }),
    }),
});

export const { useGetUsersQuery, useCreateUserMutation } = api;
```

## Zustand

Lightweight alternative — minimal boilerplate, no Provider needed:

```tsx
import { create } from "zustand";

interface CounterStore {
    count: number;
    increment: () => void;
    decrement: () => void;
    reset: () => void;
}

const useCounterStore = create<CounterStore>((set) => ({
    count: 0,
    increment: () => set((state) => ({ count: state.count + 1 })),
    decrement: () => set((state) => ({ count: state.count - 1 })),
    reset: () => set({ count: 0 }),
}));

// In component — automatic selector, only re-renders when `count` changes
function Counter() {
    const count = useCounterStore((state) => state.count);
    const increment = useCounterStore((state) => state.increment);

    return <button onClick={increment}>{count}</button>;
}
```

## React Query / TanStack Query (Server State)

Separates **server state** (async, cached, shared) from **client state**:

```tsx
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";

function UserList() {
    const { data, isLoading, error } = useQuery({
        queryKey: ["users"],
        queryFn: () => fetch("/api/users").then(r => r.json()),
        staleTime: 5 * 60 * 1000, // data is fresh for 5 minutes
    });

    if (isLoading) return <Spinner />;
    if (error) return <ErrorMessage error={error} />;
    return <ul>{data.map(user => <li key={user.id}>{user.name}</li>)}</ul>;
}

// Mutations with cache invalidation
function CreateUser() {
    const queryClient = useQueryClient();
    const mutation = useMutation({
        mutationFn: (newUser: CreateUserDto) =>
            fetch("/api/users", { method: "POST", body: JSON.stringify(newUser) }),
        onSuccess: () => {
            queryClient.invalidateQueries({ queryKey: ["users"] }); // refetch user list
        },
    });

    return <button onClick={() => mutation.mutate({ name: "Alice" })}>Create</button>;
}
```

**Key features:** automatic caching, deduplication, background refetching, stale-while-revalidate, optimistic updates,
pagination/infinite scroll support.

## Comparison

| Feature              | Context           | Redux Toolkit      | Zustand            | React Query        |
|----------------------|-------------------|--------------------|--------------------|--------------------|
| Boilerplate          | Low               | Medium             | Very low           | Low                |
| Bundle size          | 0 (built-in)     | ~11 KB             | ~1 KB              | ~13 KB             |
| DevTools             | React DevTools    | Redux DevTools     | Redux DevTools ext | React Query DevTools |
| Selectors            | No (full re-render)| `useSelector`     | Built-in           | N/A                |
| Async / Server state | Manual            | RTK Query          | Middleware          | Primary focus      |
| Best for             | Infrequent global | Complex client     | Simple global      | Server state       |

## Common Interview Questions

1. **When to use Context vs Redux/Zustand?** — Context for infrequently changing global state (theme, auth). External
   stores for frequently updated state, complex logic, or when you need selectors to avoid re-renders.
2. **What is prop drilling and how to fix it?** — Passing props through many intermediate components that don't use
   them. Fix with Context, component composition (`children`), or external state.
3. **Client state vs server state?** — Client state is synchronous and owned by the app (form values, UI state).
   Server state is async, cached, shared, and owned by the backend. React Query handles server state specifically.
4. **How does Redux work?** — Single store holds the state tree. Components dispatch actions. Reducers (pure
   functions) produce new state. UI subscribes via selectors.
5. **What is `useSelector` and how does it prevent re-renders?** — Extracts a slice of Redux state. The component
   only re-renders when the selected value changes (strict equality by default).
6. **Context performance problem?** — All consumers re-render on any context value change. Mitigate by splitting
   contexts, memoizing value objects, and keeping context state minimal.

## Related

- [Hooks in Depth](../hooks/hooks-in-depth.md) — `useState`, `useReducer`, `useContext`
- [Performance Optimization](../performance/performance-optimization.md) — avoiding unnecessary re-renders
- [Component Patterns](../patterns/component-patterns.md) — composition as an alternative to prop drilling

## Resources

- [React Docs — Managing State](https://react.dev/learn/managing-state)
- [Redux Toolkit Docs](https://redux-toolkit.js.org/)
- [Zustand Docs](https://zustand-demo.pmnd.rs/)
- [TanStack Query Docs](https://tanstack.com/query/latest)
