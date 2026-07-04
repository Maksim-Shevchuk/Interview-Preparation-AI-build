# Interfaces vs Type Aliases

One of the most common TypeScript interview questions: "When do you use `interface` vs `type`?" The answer is nuanced —
they overlap significantly but have key differences.

## Syntax Comparison

```typescript
// Interface
interface User {
    name: string;
    age: number;
    greet(): string;
}

// Type alias
type User = {
    name: string;
    age: number;
    greet(): string;
};
```

For object shapes, both work identically. The differences emerge in more advanced scenarios.

## Key Differences

### 1. Declaration Merging (Interface Only)

Interfaces with the same name in the same scope **merge automatically**:

```typescript
interface Window {
    title: string;
}

interface Window {
    appVersion: number;
}

// Result: Window has both title and appVersion
const w: Window = { title: "My App", appVersion: 1 };
```

Type aliases **cannot** merge — redeclaring throws an error:

```typescript
type Window = { title: string };
type Window = { appVersion: number }; // Error: Duplicate identifier 'Window'
```

**Use case:** Declaration merging is how libraries extend global types (e.g., adding properties to `Window`, extending
`Express.Request`). This is the primary reason to prefer `interface` for public APIs.

### 2. `extends` vs `&`

```typescript
// Interface extends interface
interface Animal {
    name: string;
}
interface Dog extends Animal {
    breed: string;
}

// Type intersection
type Animal = { name: string };
type Dog = Animal & { breed: string };

// Interface extends type ✅
interface Dog extends Animal { breed: string; }

// Type intersects interface ✅
type Dog = Animal & { breed: string };
```

Both achieve inheritance, but `extends` produces **better error messages** when there are conflicts:

```typescript
interface A { x: number }
interface B extends A { x: string } // Error: Type 'string' is not assignable to type 'number'

type A = { x: number };
type B = A & { x: string };         // No error! x becomes `never` (number & string = never)
```

### 3. Capabilities Unique to `type`

Type aliases can represent things interfaces cannot:

```typescript
// Primitives
type ID = string | number;

// Union types
type Result = "success" | "error" | "pending";

// Tuples
type Point = [number, number];

// Mapped types
type Readonly<T> = { readonly [K in keyof T]: T[K] };

// Conditional types
type NonNullable<T> = T extends null | undefined ? never : T;

// Template literal types
type EventName = `on${Capitalize<string>}`;

// Extracting types
type ArrayElement<T> = T extends (infer U)[] ? U : never;
```

### 4. `implements` — Both Work

Classes can implement both interfaces and type aliases:

```typescript
interface Printable {
    print(): void;
}

type Serializable = {
    serialize(): string;
};

class Document implements Printable, Serializable {
    print() { console.log(this); }
    serialize() { return JSON.stringify(this); }
}
```

## Comparison Table

| Feature                        | `interface`          | `type`               |
|--------------------------------|----------------------|----------------------|
| Object shapes                  | ✅                    | ✅                    |
| `extends` / inheritance        | ✅ `extends`          | ✅ `&` intersection   |
| `implements` in classes        | ✅                    | ✅ (object types only)|
| Declaration merging            | ✅                    | ❌                    |
| Union types                    | ❌                    | ✅                    |
| Primitive aliases              | ❌                    | ✅                    |
| Tuple types                    | ❌                    | ✅                    |
| Mapped / conditional types     | ❌                    | ✅                    |
| Computed properties            | ❌                    | ✅                    |
| Error messages on conflicts    | Better (`extends`)   | Silent (`never`)     |
| Performance (compiler)         | Slightly better*     | Slightly worse*      |

*Interfaces are cached by name in the compiler's type cache. Complex intersections of type aliases may be re-evaluated.
In practice, the difference is negligible for most projects.

## When to Use Which

### Use `interface` when:

- Defining **object shapes** and **class contracts** — it's the semantic choice.
- You need **declaration merging** (extending third-party types, module augmentation).
- Building a **public library API** — consumers may need to extend your types.

### Use `type` when:

- You need **unions**, **tuples**, **primitives**, **mapped types**, or **conditional types**.
- You're defining a **computed or derived type** (`Pick`, `Omit`, mapped types).
- The type is not purely an object shape.

### Pragmatic Approach (Most Teams)

Many teams adopt a simple rule:

> **Use `interface` for objects, `type` for everything else.**

This is the recommendation in the TypeScript Handbook and is a safe default.

## Module Augmentation with Interfaces

A real-world example — extending Express request:

```typescript
// types/express.d.ts
declare namespace Express {
    interface Request {
        user?: {
            id: string;
            role: string;
        };
    }
}

// Now req.user is available in all Express handlers
app.get("/profile", (req, res) => {
    const userId = req.user?.id; // TS knows about user
});
```

This only works because `interface` supports declaration merging.

## Common Interview Questions

1. **`interface` vs `type` — when to use which?** — `interface` for object shapes and class contracts (declaration
   merging, better error messages). `type` for unions, tuples, primitives, mapped/conditional types. Default:
   `interface` for objects, `type` for everything else.
2. **What is declaration merging?** — When two `interface` declarations with the same name in the same scope are
   automatically combined. Used for extending third-party types (e.g., `Window`, `Express.Request`).
3. **Can `type` do everything `interface` can?** — Almost. The only thing `type` cannot do is declaration merging.
4. **What happens with conflicting properties in `extends` vs `&`?** — `extends` produces a compile error.
   Intersection (`&`) silently creates `never` for the conflicting property.
5. **Performance difference?** — Interfaces are slightly better cached by the compiler, but the difference is
   negligible in practice. Not a reason to choose one over the other.

## Related

- [Type System Basics](./type-system-basics.md) — primitives, unions, intersections, enums
- [Generics](./generics.md) — parameterized interfaces and types
- [Advanced Types](./advanced-types.md) — mapped types, conditional types

## Resources

- [TypeScript Handbook — Interfaces vs Type Aliases](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#differences-between-type-aliases-and-interfaces)
- [TypeScript Wiki — Performance](https://github.com/microsoft/TypeScript/wiki/Performance#preferring-interfaces-over-intersections)
