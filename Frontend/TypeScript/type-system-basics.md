# Type System Basics

TypeScript adds a **static type system** on top of JavaScript. Types are erased at compile time — the output is plain
JS. Understanding the fundamentals is essential because every TS interview question builds on these concepts.

## Why TypeScript?

- Catches errors at **compile time** instead of runtime.
- Serves as **living documentation** — function signatures describe expected inputs/outputs.
- Enables better **IDE support** — autocompletion, refactoring, go-to-definition.
- Fully **compatible with JavaScript** — any valid JS is valid TS (gradual adoption).

## Type Annotations vs Inference

```typescript
// Explicit annotation
let name: string = "Alice";
let age: number = 30;

// Type inference — TS infers the type from the assigned value
let city = "Kyiv";        // inferred as string
let count = 0;            // inferred as number
let active = true;        // inferred as boolean

// Best practice: let TS infer when it can, annotate when it can't or when clarity helps
```

**Return type inference:**

```typescript
// TS infers return type as number
function add(a: number, b: number) {
    return a + b;
}

// Explicit return type — useful for public APIs and complex functions
function divide(a: number, b: number): number {
    return a / b;
}
```

## Primitive Types

| Type        | Example                     | Notes                                      |
|-------------|-----------------------------|--------------------------------------------|
| `string`    | `"hello"`, `` `hi` ``      | Same as JS string                          |
| `number`    | `42`, `3.14`, `NaN`         | All numbers (no int/float distinction)     |
| `boolean`   | `true`, `false`             |                                            |
| `bigint`    | `100n`                      | Arbitrary precision integers               |
| `symbol`    | `Symbol("id")`              | Unique identifiers                         |
| `null`      | `null`                      | Explicit absence                           |
| `undefined` | `undefined`                 | Uninitialized                              |

**`string` vs `String`:** Always use lowercase `string`. Uppercase `String` is the wrapper object type — almost never
what you want.

## Special Types

### `any`

Disables type checking. Avoid in production code.

```typescript
let x: any = 5;
x = "hello";       // OK
x.nonExistent();   // OK at compile time — crashes at runtime
```

### `unknown`

The type-safe counterpart of `any`. You must **narrow** the type before using the value.

```typescript
let x: unknown = getExternalData();

// x.toUpperCase();        // Error — Object is of type 'unknown'

if (typeof x === "string") {
    x.toUpperCase();        // OK — narrowed to string
}
```

**Rule of thumb:** Use `unknown` instead of `any` when you don't know the type. It forces you to validate.

### `void`

Indicates a function does not return a value.

```typescript
function log(msg: string): void {
    console.log(msg);
}
```

### `never`

Represents values that **never occur** — functions that always throw or have infinite loops, and exhausted union
branches.

```typescript
function throwError(msg: string): never {
    throw new Error(msg);
}

function infiniteLoop(): never {
    while (true) {}
}

// Exhaustive check
type Shape = "circle" | "square";
function area(shape: Shape) {
    switch (shape) {
        case "circle": return /* ... */;
        case "square": return /* ... */;
        default:
            const _exhaustive: never = shape; // Error if a case is missing
            return _exhaustive;
    }
}
```

## Object Types

### Object Literal Type

```typescript
let user: { name: string; age: number; email?: string } = {
    name: "Alice",
    age: 30,
    // email is optional
};
```

### Index Signatures

```typescript
interface StringMap {
    [key: string]: number; // any string key maps to a number value
}

const scores: StringMap = { math: 95, english: 88 };
```

## Arrays and Tuples

```typescript
// Arrays — two equivalent syntaxes
let nums: number[] = [1, 2, 3];
let names: Array<string> = ["Alice", "Bob"];

// Tuples — fixed-length arrays with specific types per position
let pair: [string, number] = ["age", 30];
pair[0]; // string
pair[1]; // number

// Named tuples (TS 4.0+) — for readability
type Point = [x: number, y: number];

// Readonly tuple
type Coord = readonly [number, number];
```

## Union and Intersection Types

### Union (`|`) — "one of"

```typescript
type ID = string | number;

function printId(id: ID) {
    if (typeof id === "string") {
        console.log(id.toUpperCase()); // narrowed to string
    } else {
        console.log(id.toFixed(2));    // narrowed to number
    }
}
```

### Intersection (`&`) — "combine all"

```typescript
type HasName = { name: string };
type HasAge = { age: number };
type Person = HasName & HasAge; // { name: string; age: number }

const p: Person = { name: "Alice", age: 30 };
```

## Literal Types

```typescript
// String literal
type Direction = "up" | "down" | "left" | "right";

// Numeric literal
type DiceRoll = 1 | 2 | 3 | 4 | 5 | 6;

// Boolean literal
type Yes = true;

// const assertion — infers the narrowest possible type
const config = {
    endpoint: "/api",
    retries: 3,
} as const;
// type: { readonly endpoint: "/api"; readonly retries: 3 }

// Without `as const`:
// type: { endpoint: string; retries: number }
```

## Enums

```typescript
// Numeric enum (auto-incremented)
enum Direction {
    Up,      // 0
    Down,    // 1
    Left,    // 2
    Right,   // 3
}

// String enum (no auto-increment — must assign each value)
enum Status {
    Active = "ACTIVE",
    Inactive = "INACTIVE",
    Pending = "PENDING",
}

// Usage
const dir: Direction = Direction.Up;
const status: Status = Status.Active;
```

**Enums vs Union Types:**

```typescript
// Union of string literals — preferred in modern TS
type Status = "ACTIVE" | "INACTIVE" | "PENDING";

// Why prefer unions?
// - No runtime overhead (enums generate JS code, unions are erased)
// - Simpler, more idiomatic
// - Works better with type narrowing
// - Enums have quirks (numeric enums allow reverse mapping, any number assignable)
```

**`const enum`** — inlined at compile time (no runtime object), but has compatibility issues with `--isolatedModules`
(used by Babel, esbuild, SWC). Generally avoid in library code.

## Type Assertions

```typescript
// "I know better than the compiler"
const input = document.getElementById("name") as HTMLInputElement;
input.value; // OK — asserted as HTMLInputElement

// Alternative syntax (not usable in JSX/TSX files)
const input2 = <HTMLInputElement>document.getElementById("name");

// Double assertion (escape hatch — avoid if possible)
const x = "hello" as unknown as number; // forces incompatible assertion

// Non-null assertion (!)
function getLength(s?: string): number {
    return s!.length; // "I guarantee s is not null/undefined"
    // ⚠️ Dangerous — prefer narrowing with if-check instead
}
```

**`as const` vs `as Type`:** `as const` narrows the type (safe). `as Type` overrides the type (potentially unsafe).

## `strictNullChecks`

With `strictNullChecks: true` (recommended, on by default in strict mode):

```typescript
let name: string = "Alice";
name = null;       // Error — Type 'null' is not assignable to type 'string'
name = undefined;  // Error

let maybeName: string | null = null; // OK — explicitly allows null
```

Without it, `null` and `undefined` are assignable to any type — a major source of bugs.

## Common Interview Questions

1. **`any` vs `unknown`?** — `any` disables type checking entirely. `unknown` is type-safe — you must narrow before
   using the value. Prefer `unknown`.
2. **What is `never`?** — The type of values that never occur: throwing functions, infinite loops, exhausted union
   branches. It's the bottom type — assignable to everything, nothing is assignable to it.
3. **Union vs intersection?** — Union (`A | B`) means the value is one of the types. Intersection (`A & B`) means
   the value has all properties from both types.
4. **Why prefer string literal unions over enums?** — No runtime overhead, simpler, better type narrowing. Enums
   generate JS code and have quirks (numeric enums accept any number).
5. **What does `as const` do?** — Creates a deeply readonly type with literal types instead of widened types.
   `{ x: 1 } as const` becomes `{ readonly x: 1 }` instead of `{ x: number }`.
6. **What is `strictNullChecks`?** — When enabled, `null` and `undefined` are not assignable to other types unless
   explicitly included in the union. Catches null-related bugs at compile time.

## Related

- [Interfaces vs Type Aliases](./interfaces-vs-types.md) — when to use `interface` vs `type`
- [Generics](./generics.md) — parameterized types
- [Type Narrowing](./type-narrowing.md) — refining types with guards

## Resources

- [TypeScript Handbook — Everyday Types](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html)
- [TypeScript Handbook — Type Declarations](https://www.typescriptlang.org/docs/handbook/2/type-declarations.html)
- [Total TypeScript — Beginners Tutorial](https://www.totaltypescript.com/tutorials/beginners-typescript)
