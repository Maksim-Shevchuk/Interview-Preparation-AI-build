# Generics

Generics let you write reusable, type-safe code that works with **any type** without sacrificing type information.
They are TypeScript's equivalent of Java generics — and Java developers transitioning to TS should feel at home.

## Basic Syntax

```typescript
// Generic function — T is a type parameter
function identity<T>(value: T): T {
    return value;
}

identity<string>("hello"); // explicit: T = string
identity(42);              // inferred: T = number

// Generic arrow function (note the trailing comma in TSX files to disambiguate from JSX)
const identity = <T,>(value: T): T => value;
```

## Generic Interfaces and Types

```typescript
// Generic interface
interface ApiResponse<T> {
    data: T;
    status: number;
    message: string;
}

const userResponse: ApiResponse<User> = {
    data: { name: "Alice", age: 30 },
    status: 200,
    message: "OK",
};

// Generic type alias
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };
```

## Generic Constraints (`extends`)

Restrict what types are accepted:

```typescript
// T must have a `length` property
function logLength<T extends { length: number }>(item: T): void {
    console.log(item.length);
}

logLength("hello");    // OK — string has length
logLength([1, 2, 3]);  // OK — array has length
logLength(42);         // Error — number has no length

// T must be a key of obj
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
    return obj[key];
}

const user = { name: "Alice", age: 30 };
getProperty(user, "name"); // OK, returns string
getProperty(user, "foo");  // Error — "foo" is not a key of user
```

## Default Type Parameters

```typescript
interface PaginatedResponse<T, M = Record<string, unknown>> {
    items: T[];
    total: number;
    page: number;
    meta: M;
}

// M defaults to Record<string, unknown>
const response: PaginatedResponse<User> = { /* ... */ };

// Override M explicitly
const response2: PaginatedResponse<User, { cached: boolean }> = { /* ... */ };
```

## Generic Classes

```typescript
class Stack<T> {
    private items: T[] = [];

    push(item: T): void {
        this.items.push(item);
    }

    pop(): T | undefined {
        return this.items.pop();
    }

    peek(): T | undefined {
        return this.items[this.items.length - 1];
    }
}

const numStack = new Stack<number>();
numStack.push(1);
numStack.push("hello"); // Error — Argument of type 'string' is not assignable to parameter of type 'number'
```

## Multiple Type Parameters

```typescript
function map<T, U>(arr: T[], fn: (item: T) => U): U[] {
    return arr.map(fn);
}

// T = number, U = string (both inferred)
map([1, 2, 3], n => n.toString()); // string[]

// Key-value pair
type Pair<K, V> = { key: K; value: V };
const entry: Pair<string, number> = { key: "age", value: 30 };
```

## Built-in Utility Types

TypeScript ships with many generic utility types. These are the most frequently asked about:

### Transforming Properties

| Utility                      | Effect                                           | Example                         |
|------------------------------|--------------------------------------------------|---------------------------------|
| `Partial<T>`                 | All properties optional                          | `Partial<User>` — `{ name?: string; age?: number }` |
| `Required<T>`                | All properties required                          | `Required<Partial<User>>` → `User` |
| `Readonly<T>`                | All properties readonly                          | `Readonly<User>` — cannot reassign |
| `Record<K, V>`               | Object type with keys `K` and values `V`         | `Record<string, number>` — `{ [key: string]: number }` |

### Picking / Omitting

| Utility                      | Effect                                           | Example                         |
|------------------------------|--------------------------------------------------|---------------------------------|
| `Pick<T, K>`                 | Keep only specified keys                         | `Pick<User, "name">` → `{ name: string }` |
| `Omit<T, K>`                 | Remove specified keys                            | `Omit<User, "age">` → `{ name: string }` |

### Union Manipulation

| Utility                      | Effect                                           | Example                         |
|------------------------------|--------------------------------------------------|---------------------------------|
| `Exclude<T, U>`              | Remove members from union                        | `Exclude<"a" \| "b" \| "c", "a">` → `"b" \| "c"` |
| `Extract<T, U>`              | Keep only members assignable to `U`              | `Extract<string \| number, string>` → `string` |
| `NonNullable<T>`             | Remove `null` and `undefined`                    | `NonNullable<string \| null>` → `string` |

### Function Types

| Utility                      | Effect                                           | Example                         |
|------------------------------|--------------------------------------------------|---------------------------------|
| `ReturnType<T>`              | Extract return type of function                  | `ReturnType<typeof fetch>` → `Promise<Response>` |
| `Parameters<T>`              | Extract parameter types as tuple                 | `Parameters<typeof parseInt>` → `[string, number?]` |

### Practical Examples

```typescript
interface User {
    id: number;
    name: string;
    email: string;
    role: "admin" | "user";
}

// PATCH endpoint — all fields optional
type UpdateUserDto = Partial<Omit<User, "id">>; // { name?: string; email?: string; role?: ... }

// Create — everything required except id (auto-generated)
type CreateUserDto = Omit<User, "id">; // { name: string; email: string; role: ... }

// Display — only name and email
type UserPreview = Pick<User, "name" | "email">; // { name: string; email: string }

// Lookup map
type UserMap = Record<number, User>; // { [id: number]: User }
```

## Generic Patterns in React

```typescript
// Generic component
interface ListProps<T> {
    items: T[];
    renderItem: (item: T) => React.ReactNode;
    keyExtractor: (item: T) => string;
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
    return (
        <ul>
            {items.map(item => (
                <li key={keyExtractor(item)}>{renderItem(item)}</li>
            ))}
        </ul>
    );
}

// Usage — T is inferred as User
<List
    items={users}
    renderItem={user => <span>{user.name}</span>}
    keyExtractor={user => user.id.toString()}
/>

// Generic custom hook
function useLocalStorage<T>(key: string, initialValue: T): [T, (value: T) => void] {
    const [stored, setStored] = useState<T>(() => {
        const item = localStorage.getItem(key);
        return item ? JSON.parse(item) : initialValue;
    });

    const setValue = (value: T) => {
        setStored(value);
        localStorage.setItem(key, JSON.stringify(value));
    };

    return [stored, setValue];
}

const [theme, setTheme] = useLocalStorage<"light" | "dark">("theme", "light");
```

## Common Interview Questions

1. **What are generics and why use them?** — Type parameters that make code reusable without losing type safety.
   Instead of using `any`, generics preserve the actual type through the function/class.
2. **`Partial<T>` vs `Required<T>`?** — `Partial` makes all properties optional. `Required` makes all properties
   required. They are inverses.
3. **What is `keyof`?** — A type operator that produces a union of all property names (keys) of a type.
   `keyof User` → `"id" | "name" | "email" | "role"`.
4. **How to constrain a generic?** — Use `extends`: `<T extends SomeType>`. This restricts T to types that are
   assignable to `SomeType`.
5. **`Pick` vs `Omit`?** — `Pick<T, K>` keeps only the specified keys. `Omit<T, K>` removes them. They are
   inverses.
6. **How do generics work in React components?** — Same as regular generics. The component's props interface is
   parameterized, and the type is usually inferred from the props passed.

## Related

- [Type System Basics](./type-system-basics.md) — unions, intersections, literal types
- [Interfaces vs Type Aliases](./interfaces-vs-types.md) — `interface` vs `type` with generics
- [Advanced Types](./advanced-types.md) — conditional types, mapped types, `infer`

## Resources

- [TypeScript Handbook — Generics](https://www.typescriptlang.org/docs/handbook/2/generics.html)
- [TypeScript Handbook — Utility Types](https://www.typescriptlang.org/docs/handbook/utility-types.html)
- [Total TypeScript — Generics Tutorial](https://www.totaltypescript.com/tutorials/beginners-typescript)
