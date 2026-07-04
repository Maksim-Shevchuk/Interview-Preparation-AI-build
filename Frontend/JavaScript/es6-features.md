# ES6+ Features

A summary of the most important features added in ES2015 (ES6) and later editions, frequently asked in interviews.
Covers the features themselves — for deeper dives on closures, Promises, and prototypes, see the dedicated notes.

## ES2015 (ES6) — The Big Update

### Arrow Functions

```javascript
// Concise syntax
const add = (a, b) => a + b;
const square = x => x * x;           // single param — parens optional
const getObj = () => ({ key: "val" }); // returning object literal — wrap in parens

// Key differences from regular functions:
// 1. No own `this` — captures from enclosing scope (lexical this)
// 2. No `arguments` object — use rest params instead
// 3. Cannot be used as constructors (no `new`)
// 4. No `prototype` property
```

### Destructuring

```javascript
// Array destructuring
const [first, second, ...rest] = [1, 2, 3, 4, 5];
// first = 1, second = 2, rest = [3, 4, 5]

const [a, , b] = [1, 2, 3]; // skip elements — a = 1, b = 3

// Object destructuring
const { name, age, city = "Unknown" } = { name: "Alice", age: 30 };
// name = "Alice", age = 30, city = "Unknown" (default value)

// Renaming
const { name: userName } = { name: "Bob" }; // userName = "Bob"

// Nested
const { address: { street } } = { address: { street: "Main St" } };

// In function parameters (very common in React)
function UserCard({ name, age, role = "user" }) {
    return `${name}, ${age}, ${role}`;
}
```

### Spread and Rest Operators (`...`)

```javascript
// Spread — expands iterable into individual elements
const arr1 = [1, 2, 3];
const arr2 = [...arr1, 4, 5];          // [1, 2, 3, 4, 5]

const obj1 = { a: 1, b: 2 };
const obj2 = { ...obj1, c: 3, b: 99 }; // { a: 1, b: 99, c: 3 } — later props override

// Shallow copy
const copy = [...arr1];
const objCopy = { ...obj1 };

// Rest — collects remaining elements
function sum(...nums) {
    return nums.reduce((a, b) => a + b, 0);
}
sum(1, 2, 3); // 6

const { a, ...others } = { a: 1, b: 2, c: 3 }; // others = { b: 2, c: 3 }
```

### Template Literals

```javascript
const name = "World";
const greeting = `Hello, ${name}!`;     // string interpolation
const multiline = `
  Line 1
  Line 2
`;

// Tagged templates
function highlight(strings, ...values) {
    return strings.reduce((result, str, i) =>
        `${result}${str}<mark>${values[i] || ""}</mark>`, "");
}
highlight`Hello ${name}, age ${30}`; // "Hello <mark>World</mark>, age <mark>30</mark>"
```

### `let` and `const`

See [Closures and Scope](./closures-and-scope.md) for full details.

### Classes

See [Prototypes and `this`](./prototypes-and-this.md) for full details.

### Promises

See [Promises and Async/Await](./promises-and-async-await.md) for full details.

### Modules (`import` / `export`)

```javascript
// Named exports
export const PI = 3.14;
export function add(a, b) { return a + b; }

// Default export (one per module)
export default class Calculator { /* ... */ }

// Import
import Calculator from "./calculator.js";             // default
import { PI, add } from "./math.js";                   // named
import { add as sum } from "./math.js";                // renamed
import * as math from "./math.js";                     // namespace
import("./module.js").then(mod => mod.default);        // dynamic import (lazy loading)
```

### `Symbol`

```javascript
const sym = Symbol("description");
typeof sym; // "symbol"
Symbol("a") === Symbol("a"); // false — every Symbol is unique

// Use cases:
// 1. Unique property keys (no name collisions)
const id = Symbol("id");
const obj = { [id]: 123, name: "Alice" };
obj[id]; // 123 — not visible in for...in or Object.keys()

// 2. Well-known symbols — customize language behavior
class MyArray {
    static [Symbol.hasInstance](instance) {
        return Array.isArray(instance);
    }
}
[] instanceof MyArray; // true

// 3. Symbol.iterator — make objects iterable
```

### Iterators and `for...of`

```javascript
// Any object with [Symbol.iterator]() is iterable
const iterable = {
    [Symbol.iterator]() {
        let i = 0;
        return {
            next() {
                return i < 3
                    ? { value: i++, done: false }
                    : { done: true };
            }
        };
    }
};

for (const val of iterable) {
    console.log(val); // 0, 1, 2
}
```

### Generators

```javascript
function* range(start, end) {
    for (let i = start; i < end; i++) {
        yield i; // pauses execution, returns value
    }
}

const gen = range(1, 4);
gen.next(); // { value: 1, done: false }
gen.next(); // { value: 2, done: false }
gen.next(); // { value: 3, done: false }
gen.next(); // { value: undefined, done: true }

// Generators are iterable
[...range(1, 4)]; // [1, 2, 3]

// Infinite sequences
function* fibonacci() {
    let [a, b] = [0, 1];
    while (true) {
        yield a;
        [a, b] = [b, a + b];
    }
}
```

### `Map` and `Set`

```javascript
// Map — key-value pairs with any key type (objects, functions, etc.)
const map = new Map();
map.set("key", "value");
map.set(42, "number key");
map.get("key");     // "value"
map.has(42);        // true
map.size;           // 2
map.delete(42);

// vs Object: Map preserves insertion order, allows non-string keys, has .size, no prototype pollution

// Set — unique values
const set = new Set([1, 2, 3, 3, 3]);
set.size;           // 3
set.add(4);
set.has(2);         // true
set.delete(1);

// Common use: deduplicate array
const unique = [...new Set([1, 1, 2, 3, 3])]; // [1, 2, 3]
```

### `WeakMap` and `WeakSet`

- Keys must be objects (not primitives).
- Keys are **weakly held** — if no other reference to the key exists, it can be garbage collected.
- Not iterable, no `.size` — by design (GC timing is non-deterministic).
- Use case: storing metadata for DOM elements or objects without preventing GC.

## ES2016–ES2024 Highlights

| Year   | Feature                              | Example                                          |
|--------|--------------------------------------|--------------------------------------------------|
| ES2016 | `Array.prototype.includes`           | `[1,2,3].includes(2)` → `true`                   |
| ES2016 | Exponentiation operator              | `2 ** 10` → `1024`                                |
| ES2017 | `async`/`await`                      | See [Promises](./promises-and-async-await.md)     |
| ES2017 | `Object.entries/values`              | `Object.entries({a:1})` → `[["a",1]]`            |
| ES2018 | Rest/spread for objects              | `const {a, ...rest} = obj`                        |
| ES2019 | `Array.flat/flatMap`                 | `[1,[2,[3]]].flat(Infinity)` → `[1,2,3]`         |
| ES2020 | Optional chaining `?.`               | `user?.address?.street`                           |
| ES2020 | Nullish coalescing `??`              | `null ?? "default"` → `"default"`                 |
| ES2020 | `Promise.allSettled`                 | See [Promises](./promises-and-async-await.md)     |
| ES2020 | `BigInt`                             | `9007199254740993n`                               |
| ES2021 | `Promise.any`                        | See [Promises](./promises-and-async-await.md)     |
| ES2021 | Logical assignment `??=`, `||=`, `&&=` | `x ??= 5` (assign if nullish)                 |
| ES2021 | `String.replaceAll`                  | `"aaa".replaceAll("a","b")` → `"bbb"`            |
| ES2022 | Top-level `await`                    | `const data = await fetch(url)` in modules        |
| ES2022 | `Object.hasOwn(obj, prop)`           | Replaces `obj.hasOwnProperty(prop)`               |
| ES2022 | `Array.at(-1)`                       | Last element: `arr.at(-1)`                        |
| ES2022 | Class fields, private `#field`       | `class Foo { #x = 0; get x() { return this.#x; } }` |
| ES2022 | `structuredClone(obj)`               | Deep clone (handles circular refs, Date, Map, etc.) |
| ES2023 | `Array.findLast/findLastIndex`       | `[1,2,3].findLast(x => x < 3)` → `2`            |
| ES2023 | Immutable array methods              | `toSorted()`, `toReversed()`, `toSpliced()`, `with()` |
| ES2024 | `Object.groupBy`                     | `Object.groupBy(arr, item => item.category)`      |

### Optional Chaining and Nullish Coalescing (ES2020)

These two are extremely common in React code:

```javascript
// Optional chaining — short-circuits to undefined if any part is null/undefined
const street = user?.address?.street;    // no TypeError if user or address is null
const first = arr?.[0];                  // optional element access
const result = obj?.method?.();          // optional method call

// Nullish coalescing — default value for null/undefined ONLY (not "" or 0)
const name = user.name ?? "Anonymous";   // "Anonymous" only if name is null/undefined
const count = data.count ?? 0;           // 0 only if count is null/undefined

// Compare with || which treats all falsy as "missing":
"" || "default"   // "default" (empty string is falsy)
"" ?? "default"   // ""        (empty string is NOT nullish)
0 || 42           // 42        (0 is falsy)
0 ?? 42           // 0         (0 is NOT nullish)
```

### `structuredClone` (ES2022)

```javascript
const original = { date: new Date(), nested: { arr: [1, 2, 3] } };
const clone = structuredClone(original);

clone.nested.arr.push(4);
original.nested.arr; // [1, 2, 3] — deep clone, no shared references

// Handles: Date, Map, Set, ArrayBuffer, RegExp, circular references
// Does NOT handle: functions, DOM nodes, symbols, Error objects
```

## Common Interview Questions

1. **What are the main features of ES6?** — `let`/`const`, arrow functions, classes, template literals,
   destructuring, spread/rest, Promises, modules, `Symbol`, `Map`/`Set`, iterators/generators.
2. **Difference between `...` as spread vs rest?** — Spread expands (in function calls, array/object literals).
   Rest collects (in function parameters, destructuring).
3. **What is `Symbol` used for?** — Unique property keys (no collisions), well-known symbols to customize language
   behavior (`Symbol.iterator`, `Symbol.hasInstance`).
4. **`??` vs `||`?** — `||` returns the right side for any falsy value. `??` returns the right side only for
   `null`/`undefined`. Use `??` when `0`, `""`, or `false` are valid values.
5. **What is optional chaining?** — `?.` operator that short-circuits to `undefined` instead of throwing when
   accessing properties of `null`/`undefined`.
6. **How do you deep clone an object?** — `structuredClone()` (ES2022). Before that: `JSON.parse(JSON.stringify(obj))`
   (loses functions, dates, undefined) or libraries like Lodash `cloneDeep`.

## Related

- [Types and Coercion](./types-and-coercion.md) — `Symbol`, `BigInt` types
- [Closures and Scope](./closures-and-scope.md) — `let`/`const`, block scoping
- [Prototypes and `this`](./prototypes-and-this.md) — classes, arrow function `this`
- [Promises and Async/Await](./promises-and-async-await.md) — Promise API, async/await
- [Event Loop](./event-loop.md) — how async scheduling works

## Resources

- [MDN — JavaScript reference](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference)
- [tc39/proposals — Finished proposals](https://github.com/tc39/proposals/blob/main/finished-proposals.md)
- [javascript.info](https://javascript.info/)
