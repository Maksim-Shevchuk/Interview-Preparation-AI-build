# Advanced Types

Mapped types, conditional types, template literal types, and discriminated unions. These topics separate junior from
senior TS knowledge and appear in more advanced interviews.

## Mapped Types

Transform properties of an existing type by iterating over its keys with `in keyof`:

```typescript
// Built-in Readonly — implemented as a mapped type
type Readonly<T> = {
    readonly [K in keyof T]: T[K];
};

// Built-in Partial
type Partial<T> = {
    [K in keyof T]?: T[K];
};

// Custom: make all properties nullable
type Nullable<T> = {
    [K in keyof T]: T[K] | null;
};

interface User {
    name: string;
    age: number;
}

type NullableUser = Nullable<User>;
// { name: string | null; age: number | null }
```

### Key Remapping (`as`)

```typescript
// Prefix all keys with "get"
type Getters<T> = {
    [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

type UserGetters = Getters<User>;
// { getName: () => string; getAge: () => number }

// Filter keys by value type
type StringKeysOnly<T> = {
    [K in keyof T as T[K] extends string ? K : never]: T[K];
};

type UserStrings = StringKeysOnly<User>;
// { name: string } — age is excluded because it's number
```

## Conditional Types

`T extends U ? X : Y` — the ternary operator at the type level.

```typescript
type IsString<T> = T extends string ? true : false;

type A = IsString<"hello">; // true
type B = IsString<42>;      // false

// Built-in NonNullable
type NonNullable<T> = T extends null | undefined ? never : T;

type C = NonNullable<string | null | undefined>; // string
```

### Distributive Conditional Types

When `T` is a **union**, the conditional type is applied to each member individually:

```typescript
type ToArray<T> = T extends unknown ? T[] : never;

type Result = ToArray<string | number>;
// string[] | number[]  (NOT (string | number)[])

// To prevent distribution, wrap in a tuple:
type ToArrayNonDist<T> = [T] extends [unknown] ? T[] : never;
type Result2 = ToArrayNonDist<string | number>;
// (string | number)[]
```

### `infer` — Extracting Types

`infer` declares a type variable within a conditional type:

```typescript
// Extract return type of a function
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type A = MyReturnType<() => string>;         // string
type B = MyReturnType<(x: number) => void>;  // void

// Extract element type of an array
type ElementOf<T> = T extends (infer E)[] ? E : never;

type C = ElementOf<string[]>;  // string
type D = ElementOf<number>;   // never

// Extract Promise value
type Awaited<T> = T extends Promise<infer U> ? Awaited<U> : T; // recursive!

type E = Awaited<Promise<Promise<string>>>; // string

// Extract props from React component
type PropsOf<T> = T extends React.ComponentType<infer P> ? P : never;
```

## Discriminated Unions (Tagged Unions)

A union where each member has a **common literal property** (the discriminant) that TypeScript uses for narrowing.
This pattern is extremely common in React (actions, events, API responses).

```typescript
// Each shape has a `kind` discriminant
type Shape =
    | { kind: "circle"; radius: number }
    | { kind: "square"; side: number }
    | { kind: "rectangle"; width: number; height: number };

function area(shape: Shape): number {
    switch (shape.kind) {
        case "circle":
            return Math.PI * shape.radius ** 2;    // TS knows shape has `radius`
        case "square":
            return shape.side ** 2;                 // TS knows shape has `side`
        case "rectangle":
            return shape.width * shape.height;      // TS knows both width and height
    }
}
```

### Exhaustive Checking

Ensure all union members are handled:

```typescript
function area(shape: Shape): number {
    switch (shape.kind) {
        case "circle":
            return Math.PI * shape.radius ** 2;
        case "square":
            return shape.side ** 2;
        // case "rectangle": — missing!
        default:
            const _exhaustive: never = shape;
            // Error: Type '{ kind: "rectangle"; ... }' is not assignable to type 'never'
            return _exhaustive;
    }
}
```

### Real-World: Redux / useReducer Actions

```typescript
type Action =
    | { type: "INCREMENT"; payload: number }
    | { type: "DECREMENT"; payload: number }
    | { type: "RESET" };

function reducer(state: number, action: Action): number {
    switch (action.type) {
        case "INCREMENT":
            return state + action.payload;  // payload is available
        case "DECREMENT":
            return state - action.payload;
        case "RESET":
            return 0;                       // no payload on RESET
    }
}
```

## Template Literal Types

Build string types from other string types:

```typescript
type EventName = "click" | "focus" | "blur";
type Handler = `on${Capitalize<EventName>}`;
// "onClick" | "onFocus" | "onBlur"

// String manipulation utilities (built-in)
type A = Uppercase<"hello">;    // "HELLO"
type B = Lowercase<"HELLO">;    // "hello"
type C = Capitalize<"hello">;   // "Hello"
type D = Uncapitalize<"Hello">; // "hello"

// Parsing strings
type ExtractRouteParams<T extends string> =
    T extends `${string}:${infer Param}/${infer Rest}`
        ? Param | ExtractRouteParams<`/${Rest}`>
        : T extends `${string}:${infer Param}`
            ? Param
            : never;

type Params = ExtractRouteParams<"/users/:userId/posts/:postId">;
// "userId" | "postId"
```

## Index Access Types

Look up a type by key:

```typescript
interface User {
    name: string;
    address: {
        street: string;
        city: string;
    };
}

type AddressType = User["address"];       // { street: string; city: string }
type StreetType = User["address"]["city"]; // string

// With unions
type NameOrAge = User["name" | "address"]; // string | { street: string; city: string }

// Array element type
type First = [string, number, boolean][0]; // string
type ArrayItem = string[][number];          // string
```

## `keyof` and `typeof`

```typescript
// keyof — union of all keys
interface User { name: string; age: number; }
type UserKeys = keyof User; // "name" | "age"

// typeof — extract type from a value
const config = { api: "https://...", timeout: 3000 } as const;
type Config = typeof config;
// { readonly api: "https://..."; readonly timeout: 3000 }

// Combined pattern: derive types from runtime values
const STATUS = {
    ACTIVE: "active",
    INACTIVE: "inactive",
    PENDING: "pending",
} as const;

type StatusValue = (typeof STATUS)[keyof typeof STATUS];
// "active" | "inactive" | "pending"
```

## Recursive Types

```typescript
// JSON-compatible value
type JSONValue =
    | string
    | number
    | boolean
    | null
    | JSONValue[]
    | { [key: string]: JSONValue };

// Deep partial
type DeepPartial<T> = T extends object
    ? { [K in keyof T]?: DeepPartial<T[K]> }
    : T;

// Deep readonly
type DeepReadonly<T> = T extends object
    ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
    : T;
```

## Common Interview Questions

1. **What is a mapped type?** — A type that transforms each property of another type using `[K in keyof T]` syntax.
   `Partial`, `Readonly`, `Record` are built-in mapped types.
2. **What is a conditional type?** — `T extends U ? X : Y` — a type-level ternary. When applied to unions, it
   distributes over each member.
3. **What does `infer` do?** — Declares a type variable inside a conditional type that TS infers from the structure.
   Used to extract return types, element types, Promise values, etc.
4. **What is a discriminated union?** — A union of object types sharing a common literal property (tag/discriminant)
   that TS uses for narrowing in `switch`/`if` statements.
5. **How to ensure exhaustive handling of a union?** — Assign the remaining value to `never` in the `default`
   branch. If any member is unhandled, TS throws a compile error.
6. **What is `keyof typeof obj`?** — `typeof obj` extracts the type from a runtime value. `keyof` then gets its
   keys as a union. Combined, they derive a type from a runtime constant.

## Related

- [Type System Basics](./type-system-basics.md) — unions, `never`, `as const`
- [Generics](./generics.md) — utility types are built with mapped + conditional types
- [Type Narrowing](./type-narrowing.md) — how TS narrows discriminated unions

## Resources

- [TypeScript Handbook — Mapped Types](https://www.typescriptlang.org/docs/handbook/2/mapped-types.html)
- [TypeScript Handbook — Conditional Types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html)
- [TypeScript Handbook — Template Literal Types](https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html)
