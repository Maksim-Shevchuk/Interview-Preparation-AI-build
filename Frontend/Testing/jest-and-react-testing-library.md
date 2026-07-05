# Frontend Testing: Jest and React Testing Library

This file covers Jest as a test runner/assertion library and React Testing Library (RTL) patterns in depth. For RTL
query priority, query variants, and basic component testing examples see also
[React Testing](../React/testing/react-testing.md).

---

## Jest

### What Jest Is

Jest is a JavaScript testing framework that provides:
- **Test runner** — discovers and executes test files.
- **Assertion library** — `expect()` with built-in matchers.
- **Mocking** — functions, modules, timers.
- **Code coverage** — built-in Istanbul instrumentation.
- **Snapshot testing** — serializes output and compares to a saved snapshot.
- **Parallel execution** — runs test files in separate worker processes.

### Test Structure

```javascript
describe('OrderService', () => {
    // runs once before all tests in this describe block
    beforeAll(() => { /* setup shared resources */ });

    // runs before each test
    beforeEach(() => { /* reset state */ });

    // runs after each test
    afterEach(() => { /* cleanup */ });

    // runs once after all tests
    afterAll(() => { /* teardown shared resources */ });

    it('should calculate total with tax', () => {
        const total = calculateTotal(100, 0.2);
        expect(total).toBe(120);
    });

    it('should throw on negative price', () => {
        expect(() => calculateTotal(-1, 0.2)).toThrow('Price must be positive');
    });

    // skip or focus
    it.skip('skipped test', () => { });
    it.only('only this test runs', () => { });

    // parameterized tests
    it.each([
        [100, 0.1, 110],
        [200, 0.2, 240],
        [0, 0.5, 0],
    ])('calculateTotal(%i, %f) = %i', (price, tax, expected) => {
        expect(calculateTotal(price, tax)).toBe(expected);
    });
});
```

### Lifecycle Hook Execution Order

```
beforeAll (outer)
  beforeAll (inner describe)
    beforeEach (outer)
      beforeEach (inner)
        test
      afterEach (inner)
    afterEach (outer)
  afterAll (inner describe)
afterAll (outer)
```

`beforeEach`/`afterEach` at the outer level run for **every** test, including those in nested `describe` blocks.

---

## Matchers

### Common Matchers

```javascript
// exact equality
expect(value).toBe(42);              // === (primitives, same reference)
expect(obj).toEqual({ a: 1 });       // deep equality (objects/arrays)
expect(obj).toStrictEqual({ a: 1 }); // deep + checks undefined properties and array holes

// truthiness
expect(value).toBeTruthy();
expect(value).toBeFalsy();
expect(value).toBeNull();
expect(value).toBeUndefined();
expect(value).toBeDefined();

// numbers
expect(value).toBeGreaterThan(3);
expect(value).toBeGreaterThanOrEqual(3);
expect(value).toBeLessThan(10);
expect(value).toBeCloseTo(0.3, 5);   // floating point (0.1 + 0.2)

// strings
expect(str).toMatch(/pattern/);
expect(str).toContain('substr');

// arrays / iterables
expect(arr).toContain(item);         // strict equality
expect(arr).toContainEqual({ a: 1 });// deep equality
expect(arr).toHaveLength(3);

// objects
expect(obj).toHaveProperty('key');
expect(obj).toHaveProperty('nested.key', 'value');
expect(obj).toMatchObject({ a: 1 }); // subset match

// exceptions
expect(() => fn()).toThrow();
expect(() => fn()).toThrow('message');
expect(() => fn()).toThrow(TypeError);

// negation
expect(value).not.toBe(42);
```

### `toBe` vs `toEqual` vs `toStrictEqual`

```javascript
const a = { x: 1, y: [2] };
const b = { x: 1, y: [2] };

expect(a).not.toBe(b);          // different references
expect(a).toEqual(b);           // same structure — passes
expect(a).toStrictEqual(b);     // same structure, strict — passes

// toEqual ignores undefined properties, toStrictEqual does not
expect({ a: 1 }).toEqual({ a: 1, b: undefined });       // passes
expect({ a: 1 }).not.toStrictEqual({ a: 1, b: undefined }); // fails — extra key
```

### Asymmetric Matchers

Useful for partial matching inside `toEqual`, `toHaveBeenCalledWith`, etc.:

```javascript
expect(user).toEqual({
    id: expect.any(Number),
    name: expect.any(String),
    email: expect.stringContaining('@'),
    roles: expect.arrayContaining(['admin']),
    metadata: expect.objectContaining({ version: 2 }),
    optional: expect.anything(),   // any value except undefined/null
});

// in mock assertions
expect(mockFn).toHaveBeenCalledWith(
    expect.objectContaining({ type: 'ORDER_CREATED' }),
    expect.any(Function),
);
```

### jest-dom Matchers (for DOM assertions)

`@testing-library/jest-dom` extends Jest with DOM-specific matchers:

```javascript
expect(element).toBeInTheDocument();
expect(element).toBeVisible();
expect(element).toBeDisabled();
expect(element).toBeEnabled();
expect(element).toBeChecked();
expect(element).toHaveAttribute('href', '/home');
expect(element).toHaveClass('active');
expect(element).toHaveStyle({ color: 'red' });
expect(element).toHaveTextContent('Hello');
expect(element).toHaveValue('test@example.com');
expect(element).toHaveFocus();
expect(element).toBeRequired();
expect(element).toBeValid();
expect(element).toBeInvalid();
expect(element).toBeEmptyDOMElement();
expect(element).toHaveAccessibleName('Submit');
expect(element).toHaveAccessibleDescription('Click to submit the form');
expect(element).toHaveErrorMessage('Email is required');
```

---

## Mocking

### Mock Functions

```javascript
// create a standalone mock
const mockFn = jest.fn();
mockFn('arg1', 'arg2');

expect(mockFn).toHaveBeenCalled();
expect(mockFn).toHaveBeenCalledTimes(1);
expect(mockFn).toHaveBeenCalledWith('arg1', 'arg2');
expect(mockFn).toHaveBeenLastCalledWith('arg1', 'arg2');
expect(mockFn).toHaveBeenNthCalledWith(1, 'arg1', 'arg2');
expect(mockFn).toHaveReturnedWith(undefined);

// return values
mockFn.mockReturnValue(42);
mockFn.mockReturnValueOnce(1).mockReturnValueOnce(2).mockReturnValue(99);

// async return values
mockFn.mockResolvedValue({ data: 'ok' });
mockFn.mockResolvedValueOnce({ data: 'first' });
mockFn.mockRejectedValue(new Error('fail'));

// custom implementation
mockFn.mockImplementation((x) => x * 2);
mockFn.mockImplementationOnce((x) => x * 3);

// reset
mockFn.mockClear();   // clear call history and return values
mockFn.mockReset();   // mockClear + remove implementation
mockFn.mockRestore(); // mockReset + restore original (only for spies)
```

### Spying on Methods

```javascript
const obj = {
    calculate(x) { return x * 2; },
};

// spy — tracks calls but preserves original implementation
const spy = jest.spyOn(obj, 'calculate');
obj.calculate(5); // returns 10 (original)

expect(spy).toHaveBeenCalledWith(5);

// spy + replace implementation
jest.spyOn(obj, 'calculate').mockReturnValue(999);
obj.calculate(5); // returns 999

// spy on getter/setter
jest.spyOn(obj, 'name', 'get').mockReturnValue('mocked');
```

### Module Mocking

```javascript
// automatic mock — all exports become jest.fn()
jest.mock('./orderService');

// manual mock — provide implementation
jest.mock('./orderService', () => ({
    createOrder: jest.fn().mockResolvedValue({ id: 1 }),
    getOrder: jest.fn().mockResolvedValue({ id: 1, status: 'pending' }),
}));

// partial mock — mock some exports, keep others real
jest.mock('./utils', () => ({
    ...jest.requireActual('./utils'),   // keep all real exports
    formatDate: jest.fn(() => '2025-01-01'),  // mock only this one
}));
```

### `__mocks__` Directory

File-based manual mocks:

```
src/
  services/
    __mocks__/
      api.js          ← used when jest.mock('./api') is called
    api.js
```

For node_modules, place `__mocks__` in the project root:

```
__mocks__/
  axios.js            ← auto-used (no jest.mock needed for node_modules in __mocks__)
src/
```

### Mocking ES Modules and Named Exports

```javascript
// named exports
import * as utils from './utils';
jest.spyOn(utils, 'formatDate').mockReturnValue('2025-01-01');

// default export
jest.mock('./api', () => ({
    __esModule: true,
    default: jest.fn().mockResolvedValue({ data: 'ok' }),
    namedExport: jest.fn(),
}));
```

---

## Timers

```javascript
// fake all timers
jest.useFakeTimers();

test('debounces search input', () => {
    const callback = jest.fn();
    const debounced = debounce(callback, 300);

    debounced('query');
    expect(callback).not.toHaveBeenCalled();

    jest.advanceTimersByTime(300);
    expect(callback).toHaveBeenCalledWith('query');
});

test('setInterval runs multiple times', () => {
    const callback = jest.fn();
    setInterval(callback, 1000);

    jest.advanceTimersByTime(3000);
    expect(callback).toHaveBeenCalledTimes(3);
});

// other timer utilities
jest.runAllTimers();          // fast-forward until all timers are exhausted
jest.runOnlyPendingTimers();  // run only currently pending timers (avoids infinite loops)
jest.advanceTimersToNextTimer(); // advance to the next timer

// restore real timers
jest.useRealTimers();
```

### Fake Timers with `async` Code

```javascript
jest.useFakeTimers();

test('timeout with async', async () => {
    const promise = fetchWithTimeout('/api/data', 5000);

    jest.advanceTimersByTime(5000);

    await expect(promise).rejects.toThrow('Timeout');
});
```

---

## Snapshot Testing

Serializes output and saves to a `.snap` file. On subsequent runs, compares the current output to the saved snapshot.

```javascript
test('renders correctly', () => {
    const { container } = render(<Button variant="primary">Click</Button>);
    expect(container.firstChild).toMatchSnapshot();
});

// inline snapshot (stored in the test file itself)
test('formats address', () => {
    expect(formatAddress({ city: 'NYC', zip: '10001' })).toMatchInlineSnapshot(`
        "NYC, 10001"
    `);
});
```

**When to use snapshots:**
- Serialized data structures (API responses, config objects).
- Small, stable component output.
- Error messages.

**When NOT to use snapshots:**
- Large component trees — snapshots become unreadable and get rubber-stamped.
- Frequently changing UI — constant snapshot updates erode confidence.
- When a specific assertion (`toHaveTextContent`, `toBeVisible`) would be clearer.

```bash
# update outdated snapshots
jest --updateSnapshot   # or press 'u' in watch mode
```

---

## Code Coverage

```bash
jest --coverage
```

Jest uses Istanbul under the hood. Coverage is reported as:

| Metric      | What it measures                                      |
|-------------|-------------------------------------------------------|
| Statements  | Percentage of statements executed                     |
| Branches    | Percentage of if/else/switch branches taken           |
| Functions   | Percentage of functions called                        |
| Lines       | Percentage of lines executed                          |

### Configuration

```javascript
// jest.config.js
module.exports = {
    collectCoverageFrom: [
        'src/**/*.{js,jsx,ts,tsx}',
        '!src/**/*.d.ts',
        '!src/index.tsx',
        '!src/**/*.stories.{js,tsx}',
    ],
    coverageThresholds: {
        global: {
            branches: 80,
            functions: 80,
            lines: 80,
            statements: 80,
        },
    },
};
```

100% coverage does not mean 100% correctness — it only means every line was executed, not that every edge case
was verified.

---

## Jest Configuration

### Key Options

```javascript
// jest.config.js
module.exports = {
    // test environment
    testEnvironment: 'jsdom',           // 'jsdom' for React, 'node' for backend

    // file patterns
    testMatch: ['**/__tests__/**/*.[jt]s?(x)', '**/?(*.)+(spec|test).[jt]s?(x)'],
    testPathIgnorePatterns: ['/node_modules/', '/dist/'],

    // module resolution
    moduleNameMapper: {
        '^@/(.*)$': '<rootDir>/src/$1',                   // path aliases
        '\\.(css|less|scss)$': 'identity-obj-proxy',      // CSS modules
        '\\.(jpg|png|svg)$': '<rootDir>/__mocks__/fileMock.js', // static assets
    },

    // transforms
    transform: {
        '^.+\\.(ts|tsx)$': 'ts-jest',    // or 'babel-jest' or '@swc/jest'
    },

    // setup
    setupFiles: ['./jest.polyfills.js'],            // before test framework loads
    setupFilesAfterFramework: ['./jest.setup.js'],  // after framework, before tests
    setupFilesAfterFramework: undefined,
    // for jest-dom matchers:
    setupFilesAfterFramework: ['@testing-library/jest-dom'],

    // performance
    maxWorkers: '50%',    // use half of available CPU cores
};
```

### `setupFilesAfterFramework` (setup file)

```javascript
// jest.setup.js
import '@testing-library/jest-dom';

// global mocks
Object.defineProperty(window, 'matchMedia', {
    writable: true,
    value: jest.fn().mockImplementation(query => ({
        matches: false,
        media: query,
        onchange: null,
        addListener: jest.fn(),
        removeListener: jest.fn(),
        addEventListener: jest.fn(),
        removeEventListener: jest.fn(),
        dispatchEvent: jest.fn(),
    })),
});

// suppress console.error in tests (optional)
beforeEach(() => {
    jest.spyOn(console, 'error').mockImplementation(() => {});
});
```

---

## React Testing Library — Advanced Patterns

### Testing Forms with Validation

```tsx
test('shows validation errors on empty submit', async () => {
    const user = userEvent.setup();
    const onSubmit = jest.fn();
    render(<RegistrationForm onSubmit={onSubmit} />);

    await user.click(screen.getByRole('button', { name: /register/i }));

    expect(await screen.findByText(/email is required/i)).toBeInTheDocument();
    expect(screen.getByText(/password is required/i)).toBeInTheDocument();
    expect(onSubmit).not.toHaveBeenCalled();
});

test('submits when valid', async () => {
    const user = userEvent.setup();
    const onSubmit = jest.fn();
    render(<RegistrationForm onSubmit={onSubmit} />);

    await user.type(screen.getByLabelText(/email/i), 'alice@example.com');
    await user.type(screen.getByLabelText(/password/i), 'Str0ngP@ss!');
    await user.click(screen.getByRole('button', { name: /register/i }));

    await waitFor(() => {
        expect(onSubmit).toHaveBeenCalledWith({
            email: 'alice@example.com',
            password: 'Str0ngP@ss!',
        });
    });
});
```

### Testing Modals and Portals

```tsx
test('opens and closes modal', async () => {
    const user = userEvent.setup();
    render(<ConfirmDialog />);

    // modal not visible initially
    expect(screen.queryByRole('dialog')).not.toBeInTheDocument();

    // open
    await user.click(screen.getByRole('button', { name: /delete/i }));
    expect(screen.getByRole('dialog')).toBeInTheDocument();
    expect(screen.getByText(/are you sure/i)).toBeInTheDocument();

    // close via cancel
    await user.click(screen.getByRole('button', { name: /cancel/i }));
    await waitFor(() => {
        expect(screen.queryByRole('dialog')).not.toBeInTheDocument();
    });
});
```

### Testing Navigation (React Router)

```tsx
import { MemoryRouter } from 'react-router-dom';

test('navigates to about page', async () => {
    const user = userEvent.setup();
    render(
        <MemoryRouter initialEntries={['/']}>
            <App />
        </MemoryRouter>,
    );

    await user.click(screen.getByRole('link', { name: /about/i }));
    expect(screen.getByRole('heading', { name: /about us/i })).toBeInTheDocument();
});
```

### Testing Components with Redux

```tsx
import { configureStore } from '@reduxjs/toolkit';
import { Provider } from 'react-redux';
import { cartReducer } from './cartSlice';

function renderWithStore(ui, { preloadedState = {}, store = configureStore({
    reducer: { cart: cartReducer },
    preloadedState,
}) } = {}) {
    return render(<Provider store={store}>{ui}</Provider>);
}

test('displays cart item count', () => {
    renderWithStore(<CartBadge />, {
        preloadedState: {
            cart: { items: [{ id: 1 }, { id: 2 }] },
        },
    });

    expect(screen.getByText('2')).toBeInTheDocument();
});
```

### Testing `useEffect` Side Effects

```tsx
test('calls analytics on mount', () => {
    const trackPage = jest.fn();
    jest.spyOn(analytics, 'trackPage').mockImplementation(trackPage);

    render(<Dashboard />);

    expect(trackPage).toHaveBeenCalledWith('dashboard');
});
```

### Testing Error Boundaries

```tsx
test('renders fallback on error', () => {
    const BrokenComponent = () => { throw new Error('Crash'); };

    // suppress console.error for the expected error
    jest.spyOn(console, 'error').mockImplementation(() => {});

    render(
        <ErrorBoundary fallback={<p>Something went wrong</p>}>
            <BrokenComponent />
        </ErrorBoundary>,
    );

    expect(screen.getByText('Something went wrong')).toBeInTheDocument();
});
```

### Testing Accessibility

```tsx
import { axe, toHaveNoViolations } from 'jest-axe';
expect.extend(toHaveNoViolations);

test('form has no accessibility violations', async () => {
    const { container } = render(<LoginForm />);
    const results = await axe(container);
    expect(results).toHaveNoViolations();
});
```

### `within` — Scoped Queries

```tsx
test('each card has an edit button', () => {
    render(<CardList cards={[{ title: 'A' }, { title: 'B' }]} />);

    const cards = screen.getAllByRole('article');

    // query within a specific card
    const editA = within(cards[0]).getByRole('button', { name: /edit/i });
    const editB = within(cards[1]).getByRole('button', { name: /edit/i });

    expect(editA).toBeInTheDocument();
    expect(editB).toBeInTheDocument();
});
```

### `screen.debug()`

```tsx
test('debugging output', () => {
    render(<UserProfile user={mockUser} />);

    screen.debug();                           // prints entire DOM
    screen.debug(screen.getByRole('heading')); // prints specific element
});
```

---

## `waitFor`, `waitForElementToBeRemoved`, and `act`

### `waitFor`

Retries the callback until it passes or times out (default 1000ms):

```tsx
await waitFor(() => {
    expect(screen.getByText('Success')).toBeInTheDocument();
}, { timeout: 3000 });
```

**Don't wrap side-effect-free queries in `waitFor`** — use `findBy` instead:

```tsx
// ❌ unnecessary
await waitFor(() => screen.getByText('Hello'));

// ✅ simpler
await screen.findByText('Hello');
```

### `waitForElementToBeRemoved`

```tsx
test('loading spinner disappears after data loads', async () => {
    render(<UserList />);

    await waitForElementToBeRemoved(() => screen.queryByText('Loading...'));
    expect(screen.getByRole('list')).toBeInTheDocument();
});
```

### `act`

Ensures all state updates and effects are processed before assertions. RTL's `render`, `fireEvent`, and `userEvent`
already wrap in `act` — you rarely need it explicitly.

```tsx
// needed when updating state outside of RTL helpers
act(() => {
    store.dispatch(addItem({ id: 1 }));
});
expect(screen.getByText('1 item')).toBeInTheDocument();
```

**"not wrapped in act(...)" warning** means a state update happened after the test finished. Fix by:
1. Awaiting the operation: `await user.click(...)`.
2. Using `findBy` to wait for the update.
3. Adding cleanup: `await waitFor(() => ...)`.

---

## Vitest as an Alternative

Vitest is a Vite-native test runner that is API-compatible with Jest:

| Feature            | Jest                     | Vitest                       |
|--------------------|--------------------------|------------------------------|
| Config             | `jest.config.js`         | `vitest.config.ts` (or Vite config) |
| Transform          | babel-jest / ts-jest     | Vite (esbuild/SWC — much faster) |
| ES modules         | Experimental             | Native                       |
| Watch mode         | File-based               | Module-graph aware (faster)  |
| API compatibility  | —                        | Drop-in replacement for Jest |
| Snapshot           | `.snap` files            | Same                         |
| Coverage           | Istanbul                 | Istanbul or v8               |
| UI                 | —                        | Built-in `--ui` dashboard    |

Migration is mostly replacing `jest` with `vi` in mock calls:

```javascript
// Jest
jest.fn(); jest.mock(); jest.spyOn();

// Vitest
vi.fn(); vi.mock(); vi.spyOn();
```

---

## Common Interview Questions

### What is the difference between `jest.fn()`, `jest.mock()`, and `jest.spyOn()`?

- **`jest.fn()`** — creates a standalone mock function with no connection to real code. Use for callbacks, event
  handlers.
- **`jest.mock('./module')`** — replaces an entire module's exports with mocks. Use when you want to isolate the
  code under test from a dependency.
- **`jest.spyOn(obj, 'method')`** — wraps an existing method, tracking calls while optionally preserving the original
  implementation. Use when you want to observe calls without fully replacing behavior.

### What is the difference between `mockClear`, `mockReset`, and `mockRestore`?

- **`mockClear`** — resets `mock.calls`, `mock.instances`, `mock.results`. Implementation stays.
- **`mockReset`** — `mockClear` + removes the mock implementation (returns `undefined`).
- **`mockRestore`** — `mockReset` + restores the original method. Only works on `jest.spyOn()`.

### Why should you avoid testing implementation details?

Implementation detail tests (checking internal state, spy counts on private methods, CSS class names) break when
you **refactor** without changing behavior — giving false negatives. They also pass when behavior is broken but
implementation is preserved — giving false positives. Instead, test what the user observes: rendered output,
navigation, network requests, callbacks.

### How do you test a component that fetches data on mount?

1. Mock the API (MSW or `jest.mock`).
2. Render the component.
3. Assert loading state with `getByText('Loading...')`.
4. Wait for data with `findByText('Alice')` or `findByRole('listitem')`.
5. Assert the final state.
6. Test the error case by overriding the handler.

### How do you handle the "not wrapped in act" warning?

This warning means a state update happened outside of React's batching. Solutions:
1. Await async operations: `await user.click(...)`.
2. Use `findBy` queries that wait for updates.
3. Wrap manual state changes in `act(() => { ... })`.
4. Ensure all timers/promises resolve before the test ends.

### When should you use snapshot tests vs explicit assertions?

Use snapshots for **stable, serializable output** (data structures, small components, error messages). Use explicit
assertions (`toHaveTextContent`, `toBeVisible`) when you care about **specific behavior** or when the component is
large/changes frequently. Snapshot tests are a supplement, not a replacement for behavioral assertions.

### How do you test a custom hook?

Use `renderHook` from `@testing-library/react`. Call the hook, assert the return value, wrap state updates in `act()`:

```tsx
const { result } = renderHook(() => useCounter(0));
expect(result.current.count).toBe(0);
act(() => result.current.increment());
expect(result.current.count).toBe(1);
```

If the hook depends on context (Router, Redux), provide a `wrapper` option with the necessary providers.
