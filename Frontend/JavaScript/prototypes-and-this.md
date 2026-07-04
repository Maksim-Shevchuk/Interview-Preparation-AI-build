# Prototypes and `this`

JavaScript's object model is based on **prototypal inheritance**, not classical. Understanding the prototype chain and
the rules of `this` binding is essential for interviews.

## Prototype Chain

Every JavaScript object has an internal `[[Prototype]]` link (accessible via `__proto__` or `Object.getPrototypeOf()`).
When a property is not found on the object itself, the engine walks up the prototype chain until it finds it or reaches
`null`.

```
myObj → MyClass.prototype → Object.prototype → null
```

```javascript
const animal = { eats: true };
const rabbit = Object.create(animal); // rabbit.__proto__ === animal
rabbit.jumps = true;

rabbit.jumps; // true    — own property
rabbit.eats;  // true    — found on prototype (animal)
rabbit.flies; // undefined — not found in the chain
```

### `prototype` vs `__proto__`

| Term                    | What it is                                                     |
|-------------------------|----------------------------------------------------------------|
| `Foo.prototype`         | An object that becomes `[[Prototype]]` of instances created via `new Foo()` |
| `obj.__proto__`         | Accessor to the object's `[[Prototype]]` (non-standard, use `Object.getPrototypeOf()`) |
| `Object.create(proto)`  | Creates a new object with `proto` as its `[[Prototype]]`       |

```javascript
function Person(name) {
    this.name = name;
}
Person.prototype.greet = function() {
    return `Hi, I'm ${this.name}`;
};

const p = new Person("Alice");
p.greet();                              // "Hi, I'm Alice"
p.__proto__ === Person.prototype;       // true
Person.prototype.__proto__ === Object.prototype; // true
Object.prototype.__proto__ === null;    // true (end of chain)
```

### `new` Operator — Step by Step

When `new Foo(args)` is called:

1. Create a new empty object.
2. Set its `[[Prototype]]` to `Foo.prototype`.
3. Call `Foo` with `this` bound to the new object.
4. If `Foo` returns an object — use that; otherwise return the new object.

```javascript
// Simplified polyfill:
function myNew(Constructor, ...args) {
    const obj = Object.create(Constructor.prototype); // steps 1-2
    const result = Constructor.apply(obj, args);       // step 3
    return result instanceof Object ? result : obj;    // step 4
}
```

## ES6 Classes

Classes are **syntactic sugar** over prototypal inheritance. Under the hood, `class` still uses prototypes.

```javascript
class Animal {
    constructor(name) {
        this.name = name;     // own property
    }
    speak() {                 // on Animal.prototype
        return `${this.name} makes a noise`;
    }
}

class Dog extends Animal {
    speak() {                 // on Dog.prototype (overrides Animal.prototype.speak)
        return `${this.name} barks`;
    }
}

const d = new Dog("Rex");
d.speak();                    // "Rex barks"
d instanceof Dog;             // true
d instanceof Animal;          // true
```

Key differences from plain functions:
- Classes are **not hoisted** (TDZ applies).
- Calling without `new` throws `TypeError`.
- Methods are **non-enumerable** by default.
- `super` keyword for calling parent methods.

## `this` Binding Rules

`this` in JavaScript is determined by **how a function is called**, not where it is defined (except arrow functions).
Four rules, in order of precedence:

### 1. `new` Binding (highest priority)

```javascript
function Foo() { this.x = 42; }
const obj = new Foo(); // this → new object
obj.x; // 42
```

### 2. Explicit Binding — `call`, `apply`, `bind`

```javascript
function greet() { return `Hi, ${this.name}`; }

const user = { name: "Alice" };
greet.call(user);               // "Hi, Alice"  — call with this = user
greet.apply(user);              // "Hi, Alice"  — same, but args as array
const bound = greet.bind(user); // returns new function with this permanently bound
bound();                        // "Hi, Alice"
```

| Method   | Arguments            | Returns              |
|----------|----------------------|----------------------|
| `call`   | `(thisArg, a, b, c)` | Calls immediately    |
| `apply`  | `(thisArg, [a,b,c])` | Calls immediately    |
| `bind`   | `(thisArg, a, b)`    | New bound function   |

### 3. Implicit Binding — method call

```javascript
const obj = {
    name: "Bob",
    greet() { return `Hi, ${this.name}`; }
};
obj.greet(); // "Hi, Bob" — this = obj (the object before the dot)
```

**Common pitfall — losing implicit binding:**

```javascript
const fn = obj.greet; // detaching the method
fn(); // "Hi, undefined" — this = global (or undefined in strict mode)
```

### 4. Default Binding (lowest priority)

```javascript
function show() { console.log(this); }
show(); // window (browser) or global (Node) in sloppy mode, undefined in strict mode
```

### Arrow Functions — Lexical `this`

Arrow functions do NOT have their own `this`. They capture `this` from the **enclosing lexical scope** at definition
time.

```javascript
const obj = {
    name: "Alice",
    greet: () => `Hi, ${this.name}`,      // this = enclosing scope (global), NOT obj
    delayedGreet() {
        setTimeout(() => {
            console.log(`Hi, ${this.name}`); // this = obj (captured from delayedGreet)
        }, 100);
    }
};

obj.greet();        // "Hi, undefined" — arrow function, this is NOT obj
obj.delayedGreet(); // "Hi, Alice" — arrow captures this from delayedGreet
```

**Arrow functions cannot be used as constructors** — `new (() => {})` throws `TypeError`.

## `instanceof` and Property Checks

```javascript
// instanceof — checks prototype chain
[] instanceof Array;   // true
[] instanceof Object;  // true (Array.prototype → Object.prototype)

// Property checks
"name" in obj;                    // true if "name" exists on obj OR its prototype chain
obj.hasOwnProperty("name");      // true only if it's an own property
Object.hasOwn(obj, "name");      // ES2022, preferred over hasOwnProperty
```

## Common Interview Questions

1. **Explain prototypal inheritance.** — Each object has a `[[Prototype]]` link. Property lookups walk up the chain
   until found or `null`. Constructors' `.prototype` becomes the `[[Prototype]]` of instances.
2. **What are the 4 rules of `this`?** — new > explicit (call/apply/bind) > implicit (method call) > default
   (global/undefined). Arrow functions don't have their own `this`.
3. **What do `call`, `apply`, `bind` do?** — All explicitly set `this`. `call` passes args individually, `apply`
   passes as array, `bind` returns a new function with `this` permanently bound.
4. **How does `new` work?** — Creates empty object, links prototype, calls constructor with `this = new object`,
   returns the object (or constructor's return value if it's an object).
5. **What is the difference between `class` and a constructor function?** — `class` is syntactic sugar. Differences:
   classes aren't hoisted, require `new`, methods are non-enumerable, `super` keyword available.
6. **Why doesn't `this` work in an arrow function used as a method?** — Arrow functions capture `this` from the
   enclosing scope. When defined at object literal level, the enclosing scope is the module/global, not the object.

## Related

- [Closures and Scope](./closures-and-scope.md) — lexical scoping, which also determines arrow function `this`
- [ES6+ Features](./es6-features.md) — classes, arrow functions, `Symbol`

## Resources

- [MDN — Inheritance and the prototype chain](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Inheritance_and_the_prototype_chain)
- [MDN — this](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this)
- "You Don't Know JS" by Kyle Simpson — *this & Object Prototypes*
