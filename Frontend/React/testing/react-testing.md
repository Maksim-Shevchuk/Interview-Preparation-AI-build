# React Testing

Testing React components: what to test, how to test, and the tooling ecosystem. Interviewers want to see that you test
**behavior, not implementation details**.

## Testing Philosophy

> "The more your tests resemble the way your software is used, the more confidence they can give you."
> — Kent C. Dodds

- Test **what the user sees and does** — not internal state, method names, or component structure.
- Write tests at the **right level** — prefer integration tests that render a feature, not unit tests for every tiny
  component.
- Avoid testing **implementation details** — don't test state values, internal methods, or how many times a function
  was called unless it's an observable side effect.

## Testing Pyramid for React

```
       ╱  E2E  ╲          Cypress, Playwright — full browser, real backend
      ╱─────────╲
     ╱Integration╲        React Testing Library — render component tree, mock API
    ╱─────────────╲
   ╱     Unit      ╲      Jest/Vitest — pure functions, hooks, utilities
  ╱─────────────────╲
```

## Tooling

| Tool                       | Purpose                              |
|----------------------------|--------------------------------------|
| **Jest** / **Vitest**      | Test runner, assertions, mocking     |
| **React Testing Library**  | Render components, query DOM, fire events |
| **MSW** (Mock Service Worker) | Mock API at the network level     |
| **Playwright** / **Cypress** | E2E browser testing                |
| **user-event**             | Simulate realistic user interactions |

## React Testing Library (RTL)

### Rendering

```tsx
import { render, screen } from "@testing-library/react";
import { UserCard } from "./UserCard";

test("renders user name and email", () => {
    render(<UserCard name="Alice" email="alice@example.com" />);

    expect(screen.getByText("Alice")).toBeInTheDocument();
    expect(screen.getByText("alice@example.com")).toBeInTheDocument();
});
```

### Query Priority

RTL queries ordered by preference — use the **most accessible** query first:

| Priority | Query                  | Use for                                    |
|----------|------------------------|--------------------------------------------|
| 1        | `getByRole`            | Accessible elements (`button`, `heading`)  |
| 2        | `getByLabelText`       | Form fields with labels                    |
| 3        | `getByPlaceholderText` | Inputs with placeholder                    |
| 4        | `getByText`            | Non-interactive text content               |
| 5        | `getByDisplayValue`    | Current input/select value                 |
| 6        | `getByAltText`         | Images                                     |
| 7        | `getByTitle`           | `title` attribute                          |
| 8        | `getByTestId`          | Last resort — `data-testid` attribute      |

```tsx
// ✅ Prefer accessible queries
screen.getByRole("button", { name: "Submit" });
screen.getByRole("heading", { level: 1 });
screen.getByLabelText("Email");

// ❌ Avoid — tests implementation, not behavior
screen.getByTestId("submit-btn");
container.querySelector(".btn-primary");
```

### Query Variants

| Variant       | 0 matches     | 1 match  | 1+ matches | Async? |
|---------------|---------------|----------|------------|--------|
| `getBy`       | Throw         | Return   | Throw      | No     |
| `queryBy`     | Return `null` | Return   | Throw      | No     |
| `findBy`      | Throw         | Return   | Throw      | Yes    |
| `getAllBy`     | Throw         | Array    | Array      | No     |
| `queryAllBy`  | `[]`          | Array    | Array      | No     |
| `findAllBy`   | Throw         | Array    | Array      | Yes    |

- **`getBy`** — element must exist now.
- **`queryBy`** — assert element does NOT exist: `expect(screen.queryByText("Error")).not.toBeInTheDocument()`.
- **`findBy`** — wait for element to appear (async, uses `waitFor` internally).

### User Events

```tsx
import userEvent from "@testing-library/user-event";

test("submits form with user data", async () => {
    const user = userEvent.setup();
    const onSubmit = jest.fn();

    render(<LoginForm onSubmit={onSubmit} />);

    await user.type(screen.getByLabelText("Email"), "alice@example.com");
    await user.type(screen.getByLabelText("Password"), "secret123");
    await user.click(screen.getByRole("button", { name: "Log In" }));

    expect(onSubmit).toHaveBeenCalledWith({
        email: "alice@example.com",
        password: "secret123",
    });
});
```

**`userEvent` vs `fireEvent`:** `userEvent` simulates realistic browser behavior (focus, keydown, keypress, keyup,
input, change). `fireEvent` dispatches a single synthetic event. Prefer `userEvent`.

### Async Testing

```tsx
test("loads and displays users", async () => {
    render(<UserList />);

    // Wait for loading to finish
    expect(screen.getByText("Loading...")).toBeInTheDocument();

    // findBy waits for the element to appear
    const items = await screen.findAllByRole("listitem");
    expect(items).toHaveLength(3);

    // Assert loading is gone
    expect(screen.queryByText("Loading...")).not.toBeInTheDocument();
});

// Explicit waitFor for complex assertions
import { waitFor } from "@testing-library/react";

await waitFor(() => {
    expect(screen.getByText("Success")).toBeInTheDocument();
});
```

## Mocking

### Mock Functions (Jest/Vitest)

```tsx
const handleClick = jest.fn();
render(<Button onClick={handleClick}>Click</Button>);

await user.click(screen.getByRole("button"));

expect(handleClick).toHaveBeenCalledTimes(1);
```

### Mock API with MSW

```tsx
import { http, HttpResponse } from "msw";
import { setupServer } from "msw/node";

const server = setupServer(
    http.get("/api/users", () => {
        return HttpResponse.json([
            { id: 1, name: "Alice" },
            { id: 2, name: "Bob" },
        ]);
    }),
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test("fetches and displays users", async () => {
    render(<UserList />);
    expect(await screen.findByText("Alice")).toBeInTheDocument();
    expect(screen.getByText("Bob")).toBeInTheDocument();
});

// Override for error scenario
test("shows error on API failure", async () => {
    server.use(
        http.get("/api/users", () => HttpResponse.json(null, { status: 500 })),
    );
    render(<UserList />);
    expect(await screen.findByText("Failed to load")).toBeInTheDocument();
});
```

**Why MSW over mocking `fetch`/`axios`?** — MSW intercepts at the network level. Your component code runs exactly as
in production, including serialization and error handling.

### Mocking Modules

```tsx
// Mock a module
jest.mock("./api", () => ({
    fetchUsers: jest.fn().mockResolvedValue([{ id: 1, name: "Alice" }]),
}));

// Mock a hook
jest.mock("./hooks/useAuth", () => ({
    useAuth: () => ({ user: { id: 1, name: "Alice" }, isAuthenticated: true }),
}));
```

## Testing Custom Hooks

Use `renderHook` from RTL:

```tsx
import { renderHook, act } from "@testing-library/react";
import { useCounter } from "./useCounter";

test("increments counter", () => {
    const { result } = renderHook(() => useCounter(0));

    expect(result.current.count).toBe(0);

    act(() => {
        result.current.increment();
    });

    expect(result.current.count).toBe(1);
});
```

## Testing with Providers

Wrap components with necessary context providers:

```tsx
function renderWithProviders(ui: React.ReactElement) {
    return render(ui, {
        wrapper: ({ children }) => (
            <QueryClientProvider client={new QueryClient()}>
                <ThemeProvider theme="light">
                    {children}
                </ThemeProvider>
            </QueryClientProvider>
        ),
    });
}

test("renders with theme", () => {
    renderWithProviders(<ThemedButton />);
    // ...
});
```

## What to Test — Checklist

| Test                                        | Example                                          |
|---------------------------------------------|--------------------------------------------------|
| Component renders without crashing          | `render(<Comp />)` — smoke test                  |
| User sees expected content                  | `getByText`, `getByRole`                         |
| User interactions work                      | Click, type, select → assert result              |
| Conditional rendering                       | Toggle visibility, error/loading/success states  |
| Form validation                             | Invalid input → error message shown              |
| API integration                             | Mock API → assert data rendered                  |
| Error states                                | Mock API error → assert error UI                 |
| Accessibility                               | Elements have proper roles, labels               |

## Common Interview Questions

1. **What is React Testing Library?** — A testing utility that renders components and queries them the way users
   would (by text, role, label), encouraging tests that don't depend on implementation details.
2. **What query should I use?** — Prefer `getByRole` > `getByLabelText` > `getByText`. Use `getByTestId` as last
   resort. Match how a user or screen reader finds elements.
3. **`getBy` vs `queryBy` vs `findBy`?** — `getBy` throws if not found (assert existence). `queryBy` returns null
   (assert absence). `findBy` waits asynchronously (assert appearance after async operation).
4. **How to test async components?** — Use `findBy` queries (they wait) or `waitFor` for complex assertions. Mock
   API calls with MSW.
5. **`userEvent` vs `fireEvent`?** — `userEvent` simulates full browser interaction (focus, keystrokes, click).
   `fireEvent` dispatches a single event. Prefer `userEvent` for realistic tests.
6. **What should I NOT test?** — Internal state values, CSS class names, implementation details, third-party library
   internals. Test observable behavior that users care about.

## Related

- [Components, JSX, and Virtual DOM](../fundamentals/components-jsx-and-virtual-dom.md) — component fundamentals
- [Hooks in Depth](../hooks/hooks-in-depth.md) — testing custom hooks
- [Component Patterns](../patterns/component-patterns.md) — patterns that affect testing strategy

## Resources

- [React Testing Library Docs](https://testing-library.com/docs/react-testing-library/intro/)
- [Kent C. Dodds — Testing React](https://kentcdodds.com/blog/common-mistakes-with-react-testing-library)
- [MSW Docs](https://mswjs.io/)
- [React Docs — Testing](https://react.dev/learn/testing)
