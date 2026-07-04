# Type Narrowing and Type Guards

Type narrowing is how TypeScript **refines a broad type into a more specific one** inside a conditional block. It's
the mechanism that makes union types practical and is frequently tested in interviews.

## What is Narrowing?

```typescript
function padLeft(value: string, padding: string | number): string {
    // Here, padding is string | number — too broad to call methods on

    if (typeof padding === "number") {
        // Narrowed to number
        return " ".repeat(padding) + value;
    }

    // Narrowed to string (TS eliminates number after the if-block)
    return padding + value;
}
```

TypeScript analyzes **control flow** — after each check, it knows what type remains.

## Built-in Narrowing Techniques

### `typeof` Guard

Works for primitives: `"string"`, `"number"`, `"boolean"`, `"bigint"`, `"symbol"`, `"undefined"`, `"object"`,
`"function"`.

```typescript
function print(value: string | number | boolean) {
    if (typeof value === "string") {
        console.log(value.toUpperCase()); // string
    } else if (typeof value === "number") {
        console.log(value.toFixed(2));     // number
    } else {
        console.log(value);                // boolean
    }
}
```

**Limitation:** `typeof null === "object"`, so `typeof` alone can't distinguish `null` from objects.

### `instanceof` Guard

Works for class instances:

```typescript
function logError(error: Error | string) {
    if (error instanceof Error) {
        console.log(error.message); // Error
        console.log(error.stack);
    } else {
        console.log(error);         // string
    }
}

class Cat { meow() {} }
class Dog { bark() {} }

function speak(animal: Cat | Dog) {
    if (animal instanceof Cat) {
        animal.meow(); // Cat
    } else {
        animal.bark(); // Dog
    }
}
```

### Truthiness Narrowing

```typescript
function printName(name: string | null | undefined) {
    if (name) {
        console.log(name.toUpperCase()); // string (null and undefined eliminated)
    }
}

// ⚠️ Careful: falsy values like "" and 0 are also eliminated
function process(value: string | number | null) {
    if (value) {
        // string | number — but "" and 0 are excluded here too!
    }
}
```

### Equality Narrowing

```typescript
function example(x: string | number, y: string | boolean) {
    if (x === y) {
        // Both must be string (only common type)
        x.toUpperCase(); // string
        y.toUpperCase(); // string
    }
}

// null / undefined checks
function process(value: string | null) {
    if (value !== null) {
        value.toUpperCase(); // string
    }
}

// == null checks both null and undefined
function process2(value: string | null | undefined) {
    if (value != null) {
        value.toUpperCase(); // string
    }
}
```

### `in` Operator Narrowing

Checks if a property exists in an object:

```typescript
type Fish = { swim: () => void };
type Bird = { fly: () => void };

function move(animal: Fish | Bird) {
    if ("swim" in animal) {
        animal.swim(); // Fish
    } else {
        animal.fly();  // Bird
    }
}
```

### Discriminated Union Narrowing

The most structured approach — use a **tag property** (see [Advanced Types](./advanced-types.md)):

```typescript
type ApiResult =
    | { status: "success"; data: User }
    | { status: "error"; error: string }
    | { status: "loading" };

function handle(result: ApiResult) {
    switch (result.status) {
        case "success":
            console.log(result.data.name); // data is available
            break;
        case "error":
            console.log(result.error);      // error is available
            break;
        case "loading":
            console.log("Loading...");      // no data or error
            break;
    }
}
```

## Custom Type Guards (`is`)

When built-in checks aren't enough, write a **type predicate** function:

```typescript
// Return type is `value is string` — a type predicate
function isString(value: unknown): value is string {
    return typeof value === "string";
}

function process(value: unknown) {
    if (isString(value)) {
        value.toUpperCase(); // narrowed to string
    }
}
```

### Practical Examples

```typescript
// Check if value is a specific interface
interface User {
    name: string;
    email: string;
}

function isUser(obj: unknown): obj is User {
    return (
        typeof obj === "object" &&
        obj !== null &&
        "name" in obj &&
        "email" in obj &&
        typeof (obj as User).name === "string" &&
        typeof (obj as User).email === "string"
    );
}

// Filter arrays with type narrowing
const mixed: (string | number)[] = [1, "hello", 2, "world"];

// Without type guard — result is (string | number)[]
const strings1 = mixed.filter(x => typeof x === "string");

// With type guard — result is string[]
const strings2 = mixed.filter((x): x is string => typeof x === "string");
```

### Assertion Functions (`asserts`)

Instead of returning a boolean, assert functions throw if the condition is false. After the call, TS narrows the type
in the remaining scope:

```typescript
function assertIsString(value: unknown): asserts value is string {
    if (typeof value !== "string") {
        throw new Error(`Expected string, got ${typeof value}`);
    }
}

function process(value: unknown) {
    assertIsString(value);
    // After this line, value is narrowed to string — or the function threw
    value.toUpperCase(); // OK
}

// Non-null assertion function
function assertDefined<T>(value: T | null | undefined, msg?: string): asserts value is T {
    if (value == null) {
        throw new Error(msg ?? "Value is null or undefined");
    }
}
```

## Control Flow Analysis

TypeScript tracks narrowing through assignments, returns, and throws:

```typescript
function process(value: string | number | null) {
    if (value === null) {
        return; // early return — null is eliminated below
    }
    // value: string | number

    if (typeof value === "string") {
        return value.toUpperCase();
    }
    // value: number (string is eliminated by the if-block)

    return value.toFixed(2);
}
```

### Narrowing with `never` — Exhaustive Checks

```typescript
type Shape = "circle" | "square" | "triangle";

function getArea(shape: Shape): number {
    switch (shape) {
        case "circle": return 0;
        case "square": return 0;
        // Missing "triangle"!
        default:
            const _: never = shape;
            // Error: Type '"triangle"' is not assignable to type 'never'
            return _;
    }
}
```

This pattern ensures that adding a new member to the union causes a compile error wherever the union is handled.

## Common Interview Questions

1. **What is type narrowing?** — The process by which TypeScript refines a broader type to a more specific one inside
   a conditional block, using control flow analysis.
2. **What are type guards?** — Expressions that narrow types: `typeof`, `instanceof`, `in`, equality checks,
   truthiness checks, and custom `is` predicates.
3. **What is a type predicate (`is`)?** — A return type annotation `param is Type` on a boolean function that tells
   TS to narrow the type when the function returns `true`.
4. **`typeof` vs `instanceof`?** — `typeof` works for primitives. `instanceof` works for class instances (checks
   the prototype chain). Neither works for plain interfaces.
5. **How to narrow a plain interface (no class)?** — Use the `in` operator, discriminated unions (tag property),
   or custom type guards with `is`.
6. **What is an assertion function?** — A function with return type `asserts value is Type` that narrows by throwing
   on failure instead of returning a boolean.

## Related

- [Type System Basics](./type-system-basics.md) — `unknown`, `never`, unions
- [Advanced Types](./advanced-types.md) — discriminated unions, conditional types
- [Generics](./generics.md) — type guard for generic types

## Resources

- [TypeScript Handbook — Narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html)
- [TypeScript Handbook — Type Guards](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#using-type-predicates)
