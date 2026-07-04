# Redux Toolkit (RTK) — Deep Dive

Redux is the most established state management library for React. **Redux Toolkit (RTK)** is the official, opinionated
toolset that simplifies Redux — it's the only recommended way to write Redux today. This note goes deeper than the
overview in [State Management](./state-management.md).

## Core Principles

1. **Single source of truth** — the entire app state lives in one store (a single JS object).
2. **State is read-only** — the only way to change state is to dispatch an **action** (a plain object `{ type, payload }`).
3. **Changes are made with pure functions** — **reducers** take `(state, action)` and return new state. No side effects.

## Data Flow

```
UI event → dispatch(action) → middleware (thunk, logger, etc.) → reducer → new state → UI re-renders
```

```
┌──────────────────────────────────────────────────────────────┐
│                          Store                                │
│                                                              │
│  dispatch(action)                                            │
│       │                                                      │
│       ▼                                                      │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐               │
│  │Middleware │ ──▶│Middleware │ ──▶│ Reducer  │──▶ New State  │
│  │ (thunk)  │    │ (logger) │    │          │               │
│  └──────────┘    └──────────┘    └──────────┘               │
│                                       │                      │
│                                       ▼                      │
│                              useSelector() re-renders UI     │
└──────────────────────────────────────────────────────────────┘
```

## Store Setup

```tsx
// store.ts
import { configureStore } from "@reduxjs/toolkit";
import userReducer from "./features/users/userSlice";
import cartReducer from "./features/cart/cartSlice";
import { apiSlice } from "./features/api/apiSlice";

export const store = configureStore({
    reducer: {
        users: userReducer,
        cart: cartReducer,
        [apiSlice.reducerPath]: apiSlice.reducer, // RTK Query
    },
    middleware: (getDefaultMiddleware) =>
        getDefaultMiddleware().concat(apiSlice.middleware), // RTK Query middleware
    devTools: process.env.NODE_ENV !== "production",
});

// Infer types from store itself
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

```tsx
// main.tsx
import { Provider } from "react-redux";
import { store } from "./store";

<Provider store={store}>
    <App />
</Provider>
```

### Typed Hooks (TypeScript)

```tsx
// hooks.ts — use these throughout the app instead of plain useSelector/useDispatch
import { useDispatch, useSelector } from "react-redux";
import type { RootState, AppDispatch } from "./store";

export const useAppDispatch = useDispatch.withTypes<AppDispatch>();
export const useAppSelector = useSelector.withTypes<RootState>();
```

## Slices (`createSlice`)

A slice = **reducer + actions + initial state** for one feature, all in one place.

```tsx
// features/cart/cartSlice.ts
import { createSlice, PayloadAction } from "@reduxjs/toolkit";

interface CartItem {
    id: string;
    name: string;
    quantity: number;
    price: number;
}

interface CartState {
    items: CartItem[];
    discount: number;
}

const initialState: CartState = {
    items: [],
    discount: 0,
};

const cartSlice = createSlice({
    name: "cart",                // prefix for action types: "cart/addItem", "cart/removeItem"
    initialState,
    reducers: {
        // Each reducer gets auto-generated action creator
        addItem(state, action: PayloadAction<Omit<CartItem, "quantity">>) {
            const existing = state.items.find(i => i.id === action.payload.id);
            if (existing) {
                existing.quantity += 1;  // ✅ "Mutation" is OK — Immer handles immutability
            } else {
                state.items.push({ ...action.payload, quantity: 1 });
            }
        },

        removeItem(state, action: PayloadAction<string>) {
            state.items = state.items.filter(i => i.id !== action.payload);
        },

        updateQuantity(state, action: PayloadAction<{ id: string; quantity: number }>) {
            const item = state.items.find(i => i.id === action.payload.id);
            if (item) {
                item.quantity = action.payload.quantity;
            }
        },

        clearCart(state) {
            state.items = [];
            state.discount = 0;
        },

        applyDiscount(state, action: PayloadAction<number>) {
            state.discount = action.payload;
        },
    },
});

// Auto-generated action creators
export const { addItem, removeItem, updateQuantity, clearCart, applyDiscount } = cartSlice.actions;

// Reducer (default export)
export default cartSlice.reducer;
```

### Immer — Immutable Updates via "Mutations"

RTK uses **Immer** under the hood. You write mutating code, Immer produces an immutable update:

```tsx
// What you write (looks like mutation):
state.items.push(newItem);
state.user.name = "Alice";
state.items[2].quantity += 1;

// What Immer produces (actual immutable update):
// { ...state, items: [...state.items, newItem] }
// { ...state, user: { ...state.user, name: "Alice" } }
// etc.
```

**Rules:**
- Either **mutate** the `state` argument OR **return** a new value — never both.
- Immer only works inside `createSlice` / `createReducer`. Plain reducers still need manual immutable updates.

### Prepare Callback

Customize the action payload before it reaches the reducer:

```tsx
reducers: {
    addItem: {
        reducer(state, action: PayloadAction<CartItem>) {
            state.items.push(action.payload);
        },
        prepare(name: string, price: number) {
            return {
                payload: {
                    id: nanoid(),   // generate ID here, not in reducer (keep reducer pure)
                    name,
                    price,
                    quantity: 1,
                },
            };
        },
    },
},
```

## `createAsyncThunk` — Async Operations

Handles the standard async lifecycle: `pending` → `fulfilled` / `rejected`.

```tsx
import { createAsyncThunk, createSlice } from "@reduxjs/toolkit";

// Define the thunk
export const fetchUsers = createAsyncThunk(
    "users/fetchAll",                          // action type prefix
    async (_, { rejectWithValue }) => {
        try {
            const response = await fetch("/api/users");
            if (!response.ok) throw new Error("Failed to fetch");
            return (await response.json()) as User[];
        } catch (error) {
            return rejectWithValue("Failed to load users");
        }
    }
);

export const deleteUser = createAsyncThunk(
    "users/delete",
    async (userId: string) => {
        await fetch(`/api/users/${userId}`, { method: "DELETE" });
        return userId; // payload for fulfilled
    }
);

// Handle in slice
const userSlice = createSlice({
    name: "users",
    initialState: {
        items: [] as User[],
        status: "idle" as "idle" | "loading" | "succeeded" | "failed",
        error: null as string | null,
    },
    reducers: {},
    extraReducers: (builder) => {
        builder
            // fetchUsers
            .addCase(fetchUsers.pending, (state) => {
                state.status = "loading";
                state.error = null;
            })
            .addCase(fetchUsers.fulfilled, (state, action) => {
                state.status = "succeeded";
                state.items = action.payload;
            })
            .addCase(fetchUsers.rejected, (state, action) => {
                state.status = "failed";
                state.error = action.payload as string;
            })
            // deleteUser
            .addCase(deleteUser.fulfilled, (state, action) => {
                state.items = state.items.filter(u => u.id !== action.payload);
            });
    },
});
```

### Usage in Components

```tsx
function UserList() {
    const dispatch = useAppDispatch();
    const { items: users, status, error } = useAppSelector(state => state.users);

    useEffect(() => {
        if (status === "idle") {
            dispatch(fetchUsers());
        }
    }, [status, dispatch]);

    if (status === "loading") return <Spinner />;
    if (status === "failed") return <Error message={error} />;

    return (
        <ul>
            {users.map(user => (
                <li key={user.id}>
                    {user.name}
                    <button onClick={() => dispatch(deleteUser(user.id))}>Delete</button>
                </li>
            ))}
        </ul>
    );
}
```

### Thunk Lifecycle Actions

`createAsyncThunk("users/fetchAll", fn)` auto-generates three action types:

| Action type              | When dispatched            | `action.payload`            |
|--------------------------|----------------------------|-----------------------------|
| `users/fetchAll/pending`  | Before the async fn runs   | `undefined` (arg in `meta`) |
| `users/fetchAll/fulfilled`| Async fn resolved          | Return value of the fn      |
| `users/fetchAll/rejected` | Async fn threw or rejected | Error or `rejectWithValue`  |

## Selectors

### Basic Selectors

```tsx
// Inline (simple, fine for primitives)
const count = useAppSelector(state => state.cart.items.length);

// Named selector (reusable, exported from slice file)
export const selectCartItems = (state: RootState) => state.cart.items;
export const selectCartDiscount = (state: RootState) => state.cart.discount;

// In component
const items = useAppSelector(selectCartItems);
```

### Memoized Selectors (`createSelector` / Reselect)

For **derived data** — computed from state. Without memoization, the computation runs on every re-render even if input
state hasn't changed.

```tsx
import { createSelector } from "@reduxjs/toolkit"; // re-exports from Reselect

// Input selectors — extract raw slices of state
const selectItems = (state: RootState) => state.cart.items;
const selectDiscount = (state: RootState) => state.cart.discount;

// Memoized output selector — only recomputes when inputs change
export const selectCartTotal = createSelector(
    [selectItems, selectDiscount],
    (items, discount) => {
        const subtotal = items.reduce((sum, item) => sum + item.price * item.quantity, 0);
        return subtotal * (1 - discount / 100);
    }
);

// Parameterized selector
export const selectItemById = createSelector(
    [selectItems, (_state: RootState, itemId: string) => itemId],
    (items, itemId) => items.find(item => item.id === itemId)
);

// Usage
const total = useAppSelector(selectCartTotal);
const item = useAppSelector(state => selectItemById(state, productId));
```

**How it works:**
- `createSelector` caches the last result.
- Compares each input selector's result with `===` (reference equality).
- If all inputs are the same → return cached output, no recomputation.
- Default cache size = 1 (last call only).

### `useSelector` Re-render Rules

`useSelector` subscribes to the store. On every state update:
1. Runs the selector function.
2. Compares the result with the previous result using **strict equality** (`===`).
3. If different → re-render. If same → skip.

```tsx
// ❌ Creates a new array reference every time → always re-renders
const activeUsers = useAppSelector(state =>
    state.users.items.filter(u => u.isActive) // new array on every call
);

// ✅ Use createSelector → memoized, same reference if input didn't change
const selectActiveUsers = createSelector(
    [(state: RootState) => state.users.items],
    (items) => items.filter(u => u.isActive)
);
const activeUsers = useAppSelector(selectActiveUsers);
```

## Entity Adapter — Normalized State

Manage collections of entities (like a DB table) with normalized shape `{ ids: [], entities: {} }`.

```tsx
import { createEntityAdapter, createSlice } from "@reduxjs/toolkit";

interface User {
    id: string;
    name: string;
    email: string;
}

const usersAdapter = createEntityAdapter<User>({
    sortComparer: (a, b) => a.name.localeCompare(b.name), // optional
});

const userSlice = createSlice({
    name: "users",
    initialState: usersAdapter.getInitialState({
        status: "idle" as "idle" | "loading",
    }),
    // State shape: { ids: ["1","2"], entities: { "1": {...}, "2": {...} }, status: "idle" }
    reducers: {
        userAdded: usersAdapter.addOne,
        userUpdated: usersAdapter.updateOne,  // { id, changes: { name: "New" } }
        userRemoved: usersAdapter.removeOne,
        usersReceived: usersAdapter.setAll,
    },
});

// Auto-generated selectors
export const {
    selectAll: selectAllUsers,      // returns sorted array
    selectById: selectUserById,     // by ID
    selectIds: selectUserIds,       // array of IDs
    selectTotal: selectUserCount,   // count
} = usersAdapter.getSelectors((state: RootState) => state.users);
```

**Why normalized state?**
- O(1) lookup by ID (vs O(n) `find` in an array).
- No duplicate data — one source of truth per entity.
- Simpler updates — update one entity without recreating the entire array.

## Middleware

Functions that intercept dispatched actions before they reach reducers.

```
dispatch(action) → middleware1 → middleware2 → ... → reducer
```

### Default Middleware (RTK)

`configureStore` includes by default:
- **`redux-thunk`** — allows dispatching functions (for async logic).
- **Serializable check** — warns if non-serializable values are in state or actions (dev only).
- **Immutability check** — warns if state is mutated outside reducers (dev only).

### Custom Middleware

```tsx
import { Middleware } from "@reduxjs/toolkit";

const loggerMiddleware: Middleware = (storeApi) => (next) => (action) => {
    console.log("Dispatching:", action);
    const result = next(action);             // pass to next middleware / reducer
    console.log("New state:", storeApi.getState());
    return result;
};

// Analytics middleware
const analyticsMiddleware: Middleware = () => (next) => (action) => {
    if (action.type === "cart/addItem") {
        analytics.track("item_added", action.payload);
    }
    return next(action);
};

// Add to store
const store = configureStore({
    reducer: rootReducer,
    middleware: (getDefaultMiddleware) =>
        getDefaultMiddleware().concat(loggerMiddleware, analyticsMiddleware),
});
```

### Redux Thunk — How It Works

Thunk middleware checks: is the action a **function**? If yes, call it with `(dispatch, getState)`. If no, pass it
to the next middleware.

```tsx
// Thunk action creator (manual, without createAsyncThunk)
function fetchUserAndPosts(userId: string) {
    return async (dispatch: AppDispatch, getState: () => RootState) => {
        dispatch(setLoading(true));

        const user = await api.getUser(userId);
        dispatch(userReceived(user));

        // Can read current state
        const { auth } = getState();
        if (auth.isAdmin) {
            const posts = await api.getPosts(userId);
            dispatch(postsReceived(posts));
        }

        dispatch(setLoading(false));
    };
}

// Dispatch the thunk like a regular action
dispatch(fetchUserAndPosts("42"));
```

## RTK Query — Server State Management

Built into RTK. Handles data fetching, caching, invalidation, polling, optimistic updates. Replaces
`createAsyncThunk` + manual loading/error state for most API calls.

### API Slice

```tsx
import { createApi, fetchBaseQuery } from "@reduxjs/toolkit/query/react";

export const apiSlice = createApi({
    reducerPath: "api",
    baseQuery: fetchBaseQuery({
        baseUrl: "/api",
        prepareHeaders: (headers, { getState }) => {
            const token = (getState() as RootState).auth.token;
            if (token) headers.set("Authorization", `Bearer ${token}`);
            return headers;
        },
    }),
    tagTypes: ["User", "Post"],              // cache tags for invalidation
    endpoints: (builder) => ({

        // Query (GET)
        getUsers: builder.query<User[], void>({
            query: () => "/users",
            providesTags: (result) =>
                result
                    ? [...result.map(({ id }) => ({ type: "User" as const, id })), "User"]
                    : ["User"],
        }),

        getUserById: builder.query<User, string>({
            query: (id) => `/users/${id}`,
            providesTags: (result, error, id) => [{ type: "User", id }],
        }),

        // Mutation (POST/PUT/DELETE)
        createUser: builder.mutation<User, CreateUserDto>({
            query: (body) => ({
                url: "/users",
                method: "POST",
                body,
            }),
            invalidatesTags: ["User"],       // refetch all user queries after create
        }),

        updateUser: builder.mutation<User, { id: string; data: UpdateUserDto }>({
            query: ({ id, data }) => ({
                url: `/users/${id}`,
                method: "PATCH",
                body: data,
            }),
            invalidatesTags: (result, error, { id }) => [{ type: "User", id }],
        }),

        deleteUser: builder.mutation<void, string>({
            query: (id) => ({
                url: `/users/${id}`,
                method: "DELETE",
            }),
            invalidatesTags: (result, error, id) => [{ type: "User", id }],
        }),
    }),
});

// Auto-generated hooks
export const {
    useGetUsersQuery,
    useGetUserByIdQuery,
    useCreateUserMutation,
    useUpdateUserMutation,
    useDeleteUserMutation,
} = apiSlice;
```

### Usage in Components

```tsx
function UserList() {
    const { data: users, isLoading, error, refetch } = useGetUsersQuery();
    const [deleteUser] = useDeleteUserMutation();

    if (isLoading) return <Spinner />;
    if (error) return <Error />;

    return (
        <ul>
            {users?.map(user => (
                <li key={user.id}>
                    {user.name}
                    <button onClick={() => deleteUser(user.id)}>Delete</button>
                </li>
            ))}
        </ul>
    );
}
```

### Cache Invalidation — Tags

```
                    providesTags                  invalidatesTags
getUsers  ──▶  ["User", {User,1}, {User,2}]
                                                createUser ──▶ ["User"]     → refetches getUsers
                                                deleteUser(1) ──▶ [{User,1}] → refetches getUsers
getUserById(1) ──▶ [{User,1}]
                                                updateUser(1) ──▶ [{User,1}] → refetches getUserById(1) + getUsers
```

- `providesTags` — "this query provides data for these tags."
- `invalidatesTags` — "this mutation invalidates these tags." All queries providing those tags are **automatically
  refetched**.

### Polling and Refetching

```tsx
// Poll every 30 seconds
const { data } = useGetUsersQuery(undefined, {
    pollingInterval: 30000,
});

// Refetch on window focus
const { data } = useGetUsersQuery(undefined, {
    refetchOnFocus: true,      // needs setupListeners(store.dispatch)
    refetchOnReconnect: true,
});

// Manual refetch
const { refetch } = useGetUsersQuery();
<button onClick={refetch}>Refresh</button>
```

### Optimistic Updates

```tsx
updateUser: builder.mutation<User, { id: string; data: UpdateUserDto }>({
    query: ({ id, data }) => ({ url: `/users/${id}`, method: "PATCH", body: data }),
    async onQueryStarted({ id, data }, { dispatch, queryFulfilled }) {
        // Optimistically update the cache before the server responds
        const patchResult = dispatch(
            apiSlice.util.updateQueryData("getUserById", id, (draft) => {
                Object.assign(draft, data);
            })
        );
        try {
            await queryFulfilled; // wait for server confirmation
        } catch {
            patchResult.undo(); // rollback on failure
        }
    },
}),
```

## Redux DevTools

Chrome/Firefox extension for debugging Redux state.

**Features:**
- **Action log** — see every dispatched action with payload.
- **State diff** — see what changed after each action.
- **Time-travel debugging** — jump to any previous state.
- **Action replay** — replay a sequence of actions.
- **State export/import** — save/load state snapshots.

Enabled by default in `configureStore` (dev mode only).

## Project Structure

```
src/
├── app/
│   ├── store.ts          # configureStore
│   └── hooks.ts          # useAppSelector, useAppDispatch
├── features/
│   ├── auth/
│   │   ├── authSlice.ts  # createSlice + thunks
│   │   ├── authSelectors.ts
│   │   └── LoginForm.tsx
│   ├── cart/
│   │   ├── cartSlice.ts
│   │   ├── cartSelectors.ts
│   │   └── CartPage.tsx
│   └── api/
│       └── apiSlice.ts   # RTK Query
└── ...
```

**Feature-based** — each feature folder co-locates slice, selectors, thunks, and components. Recommended by RTK docs.

## `createAsyncThunk` vs RTK Query

| Aspect              | `createAsyncThunk`             | RTK Query                         |
|---------------------|--------------------------------|-----------------------------------|
| Boilerplate         | Manual: loading/error state, reducers | Auto: generated hooks, cache |
| Caching             | Manual                         | Built-in (tags, stale time)       |
| Deduplication       | Manual                         | Auto (same query = one request)   |
| Polling             | Manual `setInterval`           | `pollingInterval` option          |
| Optimistic updates  | Manual                         | `onQueryStarted` helper           |
| When to use         | Complex non-CRUD async logic   | Standard CRUD API calls           |

**Rule of thumb:** Use RTK Query for API calls. Use `createAsyncThunk` for complex async workflows that don't fit
the query/mutation model (multi-step processes, coordination between slices).

## Common Interview Questions

1. **What are the three principles of Redux?** — Single source of truth (one store), state is read-only (dispatch
   actions), changes via pure functions (reducers).
2. **What is a Redux slice?** — A `createSlice` object that bundles reducer logic, action creators, and initial
   state for one feature. Action types are auto-generated from the slice name.
3. **How does Immer work in RTK?** — Immer wraps the state in a Proxy. You write "mutating" code, Immer tracks
   the changes and produces an immutable update behind the scenes. Works only inside `createSlice`/`createReducer`.
4. **What is `createAsyncThunk`?** — Creates a thunk with auto-generated `pending`/`fulfilled`/`rejected` action
   types. Handles the async lifecycle. Handle these actions in `extraReducers`.
5. **What are selectors and why memoize them?** — Functions that extract/derive data from state. `createSelector`
   memoizes derived computations — only recomputes when input selectors return new references. Prevents unnecessary
   re-renders when used with `useSelector`.
6. **How does `useSelector` decide to re-render?** — Runs the selector on every store update, compares the result
   with `===`. New reference → re-render. Same reference → skip. For derived data, use `createSelector` to stabilize
   references.
7. **What is RTK Query?** — Built-in data fetching/caching layer in RTK. Auto-generates hooks for queries and
   mutations. Handles caching, invalidation (via tags), deduplication, polling, and optimistic updates.
8. **How does cache invalidation work in RTK Query?** — Queries declare `providesTags`, mutations declare
   `invalidatesTags`. When a mutation invalidates a tag, all queries providing that tag are automatically refetched.
9. **What is middleware in Redux?** — Functions that intercept dispatched actions before they reach reducers.
   `redux-thunk` (dispatching functions for async logic) is included by default. Custom middleware for logging,
   analytics, error reporting.
10. **When is Redux overkill?** — Small apps, simple state, state that is mostly local. Consider Context (infrequent
    global state), Zustand (simpler API), or React Query (if your "state" is mostly server data).

## Related

- [State Management](./state-management.md) — overview of all state management approaches
- [Hooks in Depth](../hooks/hooks-in-depth.md) — `useReducer`, `useContext`
- [Performance Optimization](../performance/performance-optimization.md) — avoiding re-renders with selectors

## Resources

- [Redux Toolkit Official Docs](https://redux-toolkit.js.org/)
- [Redux Essentials Tutorial](https://redux.js.org/tutorials/essentials/part-1-overview-concepts)
- [RTK Query Overview](https://redux-toolkit.js.org/rtk-query/overview)
- [Mark Erikson — Idiomatic Redux](https://blog.isquaredsoftware.com/series/idiomatic-redux/)
