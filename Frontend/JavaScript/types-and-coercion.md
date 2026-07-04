# Types and Coercion

JavaScript is a dynamically typed language with **7 primitive types** and **1 structural type**. Understanding type
coercion is one of the most frequent interview topics because it underlies many subtle bugs and tricky quiz questions.

## Primitive Types

| Type        | `typeof` result | Falsy values          | Notes                                      |
|-------------|----------------|-----------------------|--------------------------------------------|
| `undefined` | `"undefined"`  | `undefined`           | Variable declared but not assigned          |
| `null`      | `"object"` ⚠️  | `null`                | Historical bug in the spec, never fixed     |
| `boolean`   | `"boolean"`    | `false`               |                                             |
| `number`    | `"number"`     | `0`, `-0`, `NaN`      | IEEE 754 double-precision (64-bit)          |
| `bigint`    | `"bigint"`     | `0n`                  | ES2020, arbitrary precision                 |
| `string`    | `"string"`     | `""` (empty string)   | Immutable, UTF-16 encoded                   |
| `symbol`    | `"symbol"`     | _(none)_              | ES2015, guaranteed unique                   |

### Structural (Reference) Type

- **Object** — includes plain objects `{}`, arrays `[]`, functions, `Date`, `RegExp`, `Map`, `Set`, etc.
- `typeof function` → `"function"` (special case, but functions are still objects).
- `typeof []` → `"object"` — use `Array.isArray()` to check for arrays.

## `typeof` Gotchas

```javascript
typeof null          // "object"   — historical bug
typeof NaN           // "number"   — NaN is "Not-a-Number" but its type is number
typeof undeclaredVar // "undefined" — no ReferenceError (only safe way to check undeclared vars)
typeof function(){}  // "function"
typeof []            // "object"
typeof Symbol()      // "symbol"
typeof 1n            // "bigint"
```

## `==` vs `===` (Abstract vs Strict Equality)

`===` (strict equality) — **no type coercion**. Returns `true` only if type AND value match.

`==` (abstract equality) — performs **type coercion** according to the Abstract Equality Comparison Algorithm:

```javascript
// Rules of == coercion (simplified):
// 1. null == undefined → true (and vice versa). Nothing else == null or undefined.
// 2. Number vs String → String is converted to Number.
// 3. Boolean vs anything → Boolean is converted to Number first (true→1, false→0).
// 4. Object vs primitive → Object is converted via ToPrimitive (valueOf, then toString).

null == undefined    // true
null == 0            // false (null only == undefined)
null == ""           // false

"0" == false         // true  — false→0, "0"→0
"" == false          // true  — false→0, ""→0
[] == false          // true  — []→""→0, false→0
[] == ![]            // true  — ![]→false, then same as [] == false

NaN == NaN           // false — NaN is not equal to anything, including itself
NaN === NaN          // false
```

**Interview rule of thumb:** Always use `===`. The only reasonable use of `==` is `x == null` to check for both `null`
and `undefined` at once.

## Type Coercion

### To Number

```javascript
Number("")           // 0
Number(" ")          // 0
Number(null)         // 0
Number(undefined)    // NaN
Number(false)        // 0
Number(true)         // 1
Number("123")        // 123
Number("123abc")     // NaN
Number([])           // 0   — [].toString() → "" → 0
Number([1])          // 1   — [1].toString() → "1" → 1
Number([1,2])        // NaN — [1,2].toString() → "1,2" → NaN

+"5"                 // 5   — unary + is a shorthand for Number()
```

### To String

```javascript
String(null)         // "null"
String(undefined)    // "undefined"
String(true)         // "true"
String(123)          // "123"
String([1, 2, 3])    // "1,2,3"  — calls Array.prototype.toString() → join(",")
String({})           // "[object Object]"

// + operator with a string triggers string coercion:
"5" + 3              // "53"   — number is coerced to string
5 + "3"              // "53"
5 + 3 + "px"         // "8px"  — left to right: 5+3=8, then 8+"px"
"px" + 5 + 3         // "px53" — left to right: "px"+5="px5", then "px5"+3
```

### To Boolean

The **7 falsy values** (everything else is truthy):

```javascript
Boolean(false)       // false
Boolean(0)           // false
Boolean(-0)          // false
Boolean(0n)          // false
Boolean("")          // false
Boolean(null)        // false
Boolean(undefined)   // false
Boolean(NaN)         // false

// Truthy surprises:
Boolean([])          // true  — empty array is truthy!
Boolean({})          // true  — empty object is truthy!
Boolean("0")         // true  — non-empty string is truthy!
Boolean("false")     // true
```

## `Object.is()` — SameValue Comparison

Behaves like `===` except for two edge cases:

```javascript
Object.is(NaN, NaN)   // true  (=== gives false)
Object.is(0, -0)       // false (=== gives true)
```

## Common Interview Questions

1. **What are the primitive types in JavaScript?** — 7: undefined, null, boolean, number, bigint, string, symbol.
2. **Why does `typeof null` return `"object"`?** — Legacy bug from the first JS implementation (type tag for objects
   was 0, and null was represented as the null pointer 0x00).
3. **Explain `==` vs `===`.** — `===` does no coercion; `==` coerces types following the spec's Abstract Equality
   algorithm. Always prefer `===`.
4. **What is `NaN`? How do you check for it?** — "Not-a-Number", result of invalid math operations. `typeof NaN` is
   `"number"`. Check with `Number.isNaN(x)` (not the global `isNaN()` which coerces its argument).
5. **What are falsy values?** — `false`, `0`, `-0`, `0n`, `""`, `null`, `undefined`, `NaN`.
6. **What does `[] == ![]` evaluate to and why?** — `true`. `![]` → `false` (array is truthy). Then `[] == false` →
   `[] → "" → 0`, `false → 0`. `0 == 0` → `true`.

## Related

- [Closures and Scope](./closures-and-scope.md) — `var`/`let`/`const` and hoisting
- [ES6+ Features](./es6-features.md) — `Symbol`, `BigInt`, template literals

## Resources

- [MDN — JavaScript data types and data structures](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures)
- [ECMAScript Spec — Abstract Equality Comparison](https://tc39.es/ecma262/#sec-abstract-equality-comparison)
- "You Don't Know JS" by Kyle Simpson — *Types & Grammar*
