# Property Descriptors

Every property in a JavaScript object has a hidden **descriptor** — a set of meta-attributes that control how the
property behaves: whether it can be changed, deleted, or shows up in loops. Interviewers use this topic to test deep
understanding of the object model.

## Two Kinds of Descriptors

### Data Descriptor

Describes a property that holds a **value**.

| Attribute      | Default (via `=`) | Description                                              |
|----------------|-------------------|----------------------------------------------------------|
| `value`        | `undefined`       | The property's value                                     |
| `writable`     | `true`            | If `false`, the value cannot be changed                  |
| `enumerable`   | `true`            | If `false`, hidden from `for...in`, `Object.keys()`      |
| `configurable` | `true`            | If `false`, property cannot be deleted or descriptor changed (except `writable` true→false) |

### Accessor Descriptor

Describes a property defined by **getter/setter** functions. Cannot have `value` or `writable`.

| Attribute      | Default           | Description                                              |
|----------------|-------------------|----------------------------------------------------------|
| `get`          | `undefined`       | Function called when the property is read                |
| `set`          | `undefined`       | Function called when the property is assigned            |
| `enumerable`   | `true`            | Same as above                                            |
| `configurable` | `true`            | Same as above                                            |

**A descriptor is either data OR accessor — never both.** Setting `value`/`writable` together with `get`/`set` throws
`TypeError`.

## Reading Descriptors

```javascript
const obj = { name: "Alice", age: 30 };

Object.getOwnPropertyDescriptor(obj, "name");
// { value: "Alice", writable: true, enumerable: true, configurable: true }

Object.getOwnPropertyDescriptors(obj);
// {
//   name: { value: "Alice", writable: true, enumerable: true, configurable: true },
//   age:  { value: 30,      writable: true, enumerable: true, configurable: true }
// }
```

## Defining / Modifying Properties

### `Object.defineProperty`

```javascript
const user = {};

// Data descriptor
Object.defineProperty(user, "id", {
    value: 1,
    writable: false,       // read-only
    enumerable: true,
    configurable: false,   // cannot delete or reconfigure
});

user.id = 2;          // silently ignored (or TypeError in strict mode)
delete user.id;       // false — cannot delete
user.id;              // 1
```

**Important:** when creating a property via `defineProperty`, all omitted attributes default to `false`/`undefined` —
the opposite of regular assignment (`obj.x = 1`), where `writable`, `enumerable`, `configurable` all default to `true`.

```javascript
// Via assignment — all flags are true
obj.x = 1;
// Equivalent to:
Object.defineProperty(obj, "x", {
    value: 1, writable: true, enumerable: true, configurable: true
});

// Via defineProperty with omitted flags — all default to false
Object.defineProperty(obj, "y", { value: 2 });
// Equivalent to:
// { value: 2, writable: false, enumerable: false, configurable: false }
```

### `Object.defineProperties`

```javascript
Object.defineProperties(user, {
    firstName: { value: "Alice", writable: true, enumerable: true, configurable: true },
    lastName:  { value: "Smith", writable: true, enumerable: true, configurable: true },
    fullName:  {
        get() { return `${this.firstName} ${this.lastName}`; },
        set(val) {
            const [first, last] = val.split(" ");
            this.firstName = first;
            this.lastName = last;
        },
        enumerable: true,
        configurable: true,
    },
});

user.fullName;             // "Alice Smith"
user.fullName = "Bob Lee";
user.firstName;            // "Bob"
```

## Accessor Descriptors (Getters / Setters)

```javascript
const account = {
    _balance: 0, // convention: underscore = "private"

    get balance() {
        return this._balance;
    },

    set balance(amount) {
        if (amount < 0) throw new Error("Balance cannot be negative");
        this._balance = amount;
    },
};

account.balance = 100;   // calls setter
account.balance;         // 100 — calls getter
account.balance = -50;   // Error: Balance cannot be negative

Object.getOwnPropertyDescriptor(account, "balance");
// { get: [Function], set: [Function], enumerable: true, configurable: true }
// Note: no `value` or `writable` — it's an accessor descriptor
```

## Effect of `enumerable`

```javascript
const obj = {};
Object.defineProperty(obj, "hidden", { value: 42, enumerable: false });
obj.visible = 100;

Object.keys(obj);          // ["visible"]     — only enumerable
Object.values(obj);        // [100]
JSON.stringify(obj);       // '{"visible":100}' — hidden is excluded

for (const key in obj) {
    console.log(key);      // "visible" only
}

// But the property still exists and is accessible:
obj.hidden;                // 42
Object.getOwnPropertyNames(obj); // ["hidden", "visible"] — ALL own properties
```

## Effect of `configurable`

When `configurable: false`:
- Cannot `delete` the property.
- Cannot change `enumerable`.
- Cannot change descriptor type (data ↔ accessor).
- Cannot change `configurable` back to `true`.
- **Exception:** CAN change `writable` from `true` → `false` (one-way).

```javascript
const obj = {};
Object.defineProperty(obj, "x", { value: 1, writable: true, configurable: false });

obj.x = 2;          // OK — writable is true
Object.defineProperty(obj, "x", { writable: false }); // OK — one-way switch allowed
obj.x = 3;          // silently fails (or TypeError in strict)
Object.defineProperty(obj, "x", { writable: true });  // TypeError — cannot reverse
```

## Object-Level Protection

Three levels of "locking down" an entire object:

| Method                    | Prevents adding | Prevents deleting | Prevents modifying | `extensible` | `configurable` | `writable` |
|---------------------------|:-:|:-:|:-:|:-:|:-:|:-:|
| `Object.preventExtensions(obj)` | ✅ | — | — | `false` | unchanged | unchanged |
| `Object.seal(obj)`               | ✅ | ✅ | — | `false` | `false` | unchanged |
| `Object.freeze(obj)`             | ✅ | ✅ | ✅ | `false` | `false` | `false` |

```javascript
const frozen = Object.freeze({ a: 1, nested: { b: 2 } });
frozen.a = 99;          // silently ignored
frozen.c = 3;           // silently ignored
delete frozen.a;        // false

// ⚠️ freeze is SHALLOW — nested objects are NOT frozen
frozen.nested.b = 99;   // works!
frozen.nested.b;        // 99
```

Check status:

```javascript
Object.isExtensible(obj); // true/false
Object.isSealed(obj);     // true/false
Object.isFrozen(obj);     // true/false
```

### Deep Freeze (Recursive)

```javascript
function deepFreeze(obj) {
    Object.freeze(obj);
    Object.getOwnPropertyNames(obj).forEach(prop => {
        const val = obj[prop];
        if (val !== null && typeof val === "object" && !Object.isFrozen(val)) {
            deepFreeze(val);
        }
    });
    return obj;
}
```

## Practical Use Cases

1. **Immutable constants** — `Object.freeze` for config objects that must not change.
2. **Computed properties** — getters that derive values (like `fullName` from `firstName` + `lastName`).
3. **Validation** — setters that validate before assignment.
4. **Hiding internal properties** — `enumerable: false` to exclude from serialization and iteration.
5. **Library / framework internals** — non-configurable, non-writable properties to prevent accidental override (e.g.,
   `Math.PI` is non-writable, non-configurable).
6. **Proper object cloning** — `Object.defineProperties(target, Object.getOwnPropertyDescriptors(source))` preserves
   getters/setters (unlike `Object.assign` which calls them).

```javascript
// Object.assign calls getters — copies the VALUE, not the accessor
const clone1 = Object.assign({}, account);
Object.getOwnPropertyDescriptor(clone1, "balance");
// { value: 100, writable: true, ... } — getter/setter lost!

// Proper clone that preserves descriptors
const clone2 = Object.defineProperties({}, Object.getOwnPropertyDescriptors(account));
Object.getOwnPropertyDescriptor(clone2, "balance");
// { get: [Function], set: [Function], ... } — preserved!
```

## Common Interview Questions

1. **What is a property descriptor?** — A metadata object that defines how a property behaves: its value (or
   getter/setter), and whether it is writable, enumerable, and configurable.
2. **Data descriptor vs accessor descriptor?** — Data has `value` + `writable`. Accessor has `get` + `set`. They are
   mutually exclusive.
3. **Difference between `Object.freeze`, `Object.seal`, `Object.preventExtensions`?** — `preventExtensions` blocks
   adding new properties. `seal` also makes existing properties non-configurable (no delete). `freeze` also makes
   them non-writable. All are **shallow**.
4. **Why does `defineProperty` behave differently from `obj.x = 1`?** — Regular assignment sets all flags to `true`.
   `defineProperty` defaults omitted flags to `false`.
5. **How to properly clone an object with getters/setters?** — Use
   `Object.defineProperties({}, Object.getOwnPropertyDescriptors(source))`. `Object.assign` and spread call getters
   and copy the resulting value, losing the accessor.
6. **How to create a truly immutable object?** — `Object.freeze` for shallow, recursive `deepFreeze` for nested
   structures. Note: `const` only prevents reassignment of the variable, not mutation of the object.

## Related

- [Prototypes and `this`](./prototypes-and-this.md) — prototype chain, `Object.create`
- [ES6+ Features](./es6-features.md) — `Symbol`, `class` private fields as an alternative to closure-based privacy

## Resources

- [MDN — Object.defineProperty()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)
- [MDN — Property descriptors](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/getOwnPropertyDescriptor)
- [javascript.info — Property flags and descriptors](https://javascript.info/property-descriptors)
