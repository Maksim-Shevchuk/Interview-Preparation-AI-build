# Closures and Scope

Closures and scoping rules are fundamental to JavaScript. Almost every interview touches on hoisting, the difference
between `var`/`let`/`const`, and how closures capture variables.

## Scope Types

### Global Scope

Variables declared outside any function or block. In browsers — attached to `window`; in Node — to `global` (or
`globalThis` universally).

### Function Scope (`var`)

`var` declarations are scoped to the **nearest enclosing function** (or global if none). They are **hoisted** to the top
of that function — the declaration is moved up, but the assignment stays in place.

```javascript
function example() {
    console.log(x); // undefined (not ReferenceError — x is hoisted)
    var x = 10;
    console.log(x); // 10
}
```

Equivalent to:

```javascript
function example() {
    var x;           // declaration hoisted
    console.log(x);  // undefined
    x = 10;          // assignment stays
    console.log(x);  // 10
}
```

### Block Scope (`let` / `const`)

`let` and `const` (ES2015) are scoped to the **nearest enclosing block** `{}` — `if`, `for`, `while`, or standalone
blocks.

```javascript
if (true) {
    let a = 1;
    const b = 2;
}
console.log(a); // ReferenceError: a is not defined
```

## `var` vs `let` vs `const`

| Feature               | `var`                  | `let`                | `const`              |
|-----------------------|------------------------|----------------------|----------------------|
| Scope                 | Function               | Block                | Block                |
| Hoisting              | Yes (initialized `undefined`) | Yes (TDZ) | Yes (TDZ)           |
| Re-declaration        | Allowed                | Not allowed (same scope) | Not allowed      |
| Re-assignment         | Allowed                | Allowed              | Not allowed          |
| Creates `window` prop | Yes (global scope)     | No                   | No                   |

### Temporal Dead Zone (TDZ)

`let` and `const` are hoisted but **not initialized**. Accessing them before the declaration line throws
`ReferenceError`. The zone between the start of the block and the declaration is the TDZ.

```javascript
{
    // TDZ for x starts here
    console.log(x); // ReferenceError: Cannot access 'x' before initialization
    let x = 10;     // TDZ ends, x is initialized
}
```

## Hoisting

JavaScript "hoists" declarations to the top of their scope before execution.

| What                     | Hoisted?                          | Initialized?               |
|--------------------------|-----------------------------------|----------------------------|
| `var x = 5`              | Yes                               | As `undefined`             |
| `let x = 5` / `const`   | Yes                               | No (TDZ)                   |
| `function foo() {}`      | Yes (entire function body)        | Yes                        |
| `const foo = () => {}`   | Yes                               | No (TDZ) — like `const`   |
| `class Foo {}`           | Yes                               | No (TDZ)                   |

**Function declarations** are fully hoisted — you can call them before the declaration line. **Function expressions** and
**arrow functions** assigned to `let`/`const` are in TDZ.

```javascript
greet(); // "Hello" — function declaration is fully hoisted
function greet() { console.log("Hello"); }

sayBye(); // ReferenceError — const is in TDZ
const sayBye = () => console.log("Bye");
```

## Closures

A **closure** is a function that retains access to its **lexical scope** (the variables from the scope in which it was
defined), even after that scope has finished executing.

```javascript
function makeCounter() {
    let count = 0;              // enclosed variable
    return function() {
        return ++count;         // accesses count via closure
    };
}

const counter = makeCounter();
counter(); // 1
counter(); // 2
counter(); // 3
```

`count` is not garbage collected because the returned function holds a reference to it via the closure.

### Classic Interview Problem: Loop + `var`

```javascript
for (var i = 0; i < 3; i++) {
    setTimeout(() => console.log(i), 100);
}
// Output: 3, 3, 3 — all callbacks share the same `i` (function-scoped)
```

**Fix 1 — use `let`** (block-scoped, new binding per iteration):

```javascript
for (let i = 0; i < 3; i++) {
    setTimeout(() => console.log(i), 100);
}
// Output: 0, 1, 2
```

**Fix 2 — IIFE** (creates a new scope per iteration):

```javascript
for (var i = 0; i < 3; i++) {
    (function(j) {
        setTimeout(() => console.log(j), 100);
    })(i);
}
// Output: 0, 1, 2
```

### Practical Uses of Closures

- **Data privacy / encapsulation** — module pattern, factory functions
- **Partial application / currying** — `function multiply(a) { return (b) => a * b; }`
- **Memoization** — caching computed results in a closed-over variable
- **Event handlers / callbacks** — retaining state between calls
- **React hooks** — `useState`, `useEffect` rely on closures internally

## Lexical vs Dynamic Scope

JavaScript uses **lexical (static) scoping** — a function's scope is determined by **where it is defined**, not where it
is called.

```javascript
const x = 10;

function outer() {
    const x = 20;
    inner(); // prints 10, not 20
}

function inner() {
    console.log(x); // looks up x in the scope where inner was DEFINED (global)
}

outer();
```

## IIFE (Immediately Invoked Function Expression)

```javascript
(function() {
    var private = "hidden";
    console.log(private); // "hidden"
})();

console.log(private); // ReferenceError
```

Before ES modules and `let`/`const`, IIFEs were the primary way to avoid polluting global scope. Now mostly seen in
legacy code and interview questions.

## Common Interview Questions

1. **What is a closure?** — A function bundled with its lexical environment. It can access variables from the scope
   where it was defined, even after that scope has returned.
2. **`var` vs `let` vs `const`?** — `var` is function-scoped and hoisted with `undefined`. `let`/`const` are
   block-scoped and hoisted into TDZ. `const` cannot be reassigned (but objects/arrays it points to can be mutated).
3. **Explain hoisting.** — Declarations are moved to the top of their scope during compilation. `var` is initialized
   as `undefined`, `let`/`const` are in TDZ, function declarations are fully available.
4. **What is TDZ?** — Temporal Dead Zone: the period between entering a block and the `let`/`const` declaration line,
   during which accessing the variable throws `ReferenceError`.
5. **Classic loop problem: why does `var` in a `for` loop print the same value?** — `var` is function-scoped, so all
   iterations share the same variable. By the time callbacks run, the loop has finished. Fix with `let` (new binding
   per iteration) or IIFE.
6. **What is an IIFE?** — Immediately Invoked Function Expression. Creates a new scope to avoid polluting the global
   namespace. Pattern: `(function() { ... })()`.

## Related

- [Types and Coercion](./types-and-coercion.md) — how values are compared
- [Prototypes and `this`](./prototypes-and-this.md) — `this` binding rules
- [Event Loop](./event-loop.md) — how `setTimeout` callbacks are scheduled

## Resources

- [MDN — Closures](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures)
- "You Don't Know JS" by Kyle Simpson — *Scope & Closures*
