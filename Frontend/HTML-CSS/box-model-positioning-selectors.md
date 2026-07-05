# CSS Box Model, Positioning, and Selectors

Three foundational CSS topics that come up in almost every frontend interview. Understanding how elements are sized,
how they are placed on the page, and how they are targeted by CSS rules is essential.

---

## Box Model

Every element generates a rectangular box consisting of four areas (inside out):

```
┌─────────────────────────── margin ───────────────────────────┐
│  ┌─────────────────────── border ───────────────────────┐    │
│  │  ┌─────────────────── padding ─────────────────┐     │    │
│  │  │  ┌───────────── content ──────────────┐     │     │    │
│  │  │  │                                    │     │     │    │
│  │  │  └────────────────────────────────────┘     │     │    │
│  │  └─────────────────────────────────────────────┘     │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

### `box-sizing`

| Value         | `width` / `height` includes     | Default for    |
|---------------|---------------------------------|----------------|
| `content-box` | Content only                    | All elements   |
| `border-box`  | Content + padding + border      | —              |

With `content-box` (default), adding padding or border **increases** the element's total size. With `border-box`,
padding and border are subtracted from the declared width/height — the total size stays the same.

```css
/* universal reset — almost every project uses this */
*, *::before, *::after {
    box-sizing: border-box;
}
```

### Margin

- Margins are **outside** the border — they create space between elements.
- Margins can be **negative** (pull elements closer or overlap).
- Margins are **transparent** — they don't have a background.

### Margin Collapsing

Vertical margins of adjacent block-level elements **collapse** — the larger margin wins, they don't add up.

```html
<div style="margin-bottom: 20px;">A</div>
<div style="margin-top: 30px;">B</div>
<!-- Gap between A and B is 30px, NOT 50px -->
```

**When margins collapse:**
- Adjacent siblings (vertical only).
- Parent and first/last child (if no border, padding, or BFC boundary between them).
- Empty blocks (top and bottom margins of the same element collapse).

**When margins do NOT collapse:**
- Horizontal margins (never collapse).
- Floated or absolutely positioned elements.
- Elements in a flex or grid container (flex/grid items).
- Elements with `overflow` other than `visible` (they create a BFC).
- Inline-block elements.

### Inline vs Block vs Inline-Block

| Behavior                       | `block`  | `inline`    | `inline-block`   |
|--------------------------------|----------|-------------|------------------|
| Starts on new line             | Yes      | No          | No               |
| Takes full width               | Yes      | No          | No               |
| `width` / `height` respected   | Yes      | No          | Yes              |
| Vertical `margin` / `padding`  | Yes      | No effect on layout | Yes        |
| Horizontal `margin` / `padding`| Yes      | Yes         | Yes              |

---

## Positioning

The `position` property determines **how** an element is placed in the document flow.

### `static` (default)

Element is in the normal flow. `top`, `right`, `bottom`, `left`, and `z-index` have **no effect**.

### `relative`

Element stays in the normal flow but is offset **from its original position**. The space it originally occupied is
preserved — other elements are not affected.

```css
.box {
    position: relative;
    top: 10px;   /* moves 10px DOWN from original position */
    left: 20px;  /* moves 20px RIGHT from original position */
}
```

Commonly used as a **containing block** for absolutely positioned children.

### `absolute`

Element is **removed from the normal flow** — it no longer occupies space. Positioned relative to the nearest
**positioned ancestor** (any ancestor with `position` other than `static`). If none exists, it's relative to the
initial containing block (usually `<html>`).

```css
.parent {
    position: relative; /* establishes containing block */
}
.child {
    position: absolute;
    top: 0;
    right: 0; /* top-right corner of .parent */
}
```

### `fixed`

Removed from the normal flow. Positioned relative to the **viewport** — stays in place during scrolling.

```css
.navbar {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    z-index: 1000;
}
```

**Gotcha:** `fixed` positioning breaks when an ancestor has `transform`, `filter`, or `perspective` set — the element
becomes positioned relative to that ancestor instead of the viewport.

### `sticky`

A hybrid of `relative` and `fixed`. The element is relative until a scroll threshold is reached, then it "sticks" to
the specified offset.

```css
.section-header {
    position: sticky;
    top: 0; /* sticks to the top when scrolled past */
}
```

**Requirements:**
- Must have at least one of `top`, `right`, `bottom`, `left` set.
- Won't work if any ancestor has `overflow: hidden`, `overflow: auto`, or `overflow: scroll` (the sticky element
  can only stick within its scrolling container).

### Positioning Summary

| `position`  | In flow? | Offset relative to            | Creates stacking context? |
|-------------|----------|-------------------------------|---------------------------|
| `static`    | Yes      | — (offsets ignored)           | No                        |
| `relative`  | Yes      | Its own original position     | Yes (if `z-index` set)    |
| `absolute`  | No       | Nearest positioned ancestor   | Yes (if `z-index` set)    |
| `fixed`     | No       | Viewport                      | Always                    |
| `sticky`    | Yes      | Scroll container + threshold  | Always                    |

### Stacking Context

A stacking context is a three-dimensional conceptualization of elements along the z-axis. Elements within a stacking
context are painted together, and `z-index` only competes **within the same stacking context**.

**What creates a new stacking context:**
- `position: relative` / `absolute` with `z-index` other than `auto`.
- `position: fixed` / `sticky` (always).
- `opacity` less than `1`.
- `transform`, `filter`, `perspective`, `clip-path` (any value other than `none`).
- `isolation: isolate`.
- Flex/grid items with `z-index` other than `auto`.
- `will-change` with a property that creates stacking context.

**Common pitfall:** a `z-index: 9999` element won't appear above a sibling stacking context that has a lower `z-index`
at the parent level. `z-index` is not global — it's scoped to the parent stacking context.

### Containing Block

The containing block determines how percentage values (`width: 50%`, `top: 10%`) are resolved:

| Element's `position` | Containing block                                          |
|-----------------------|-----------------------------------------------------------|
| `static` / `relative` | Content box of the nearest block-level ancestor          |
| `absolute`            | Padding box of the nearest positioned ancestor           |
| `fixed`               | Viewport (or ancestor with `transform`/`filter`)         |

---

## Selectors

### Basic Selectors

| Selector       | Example          | Matches                                 |
|----------------|------------------|-----------------------------------------|
| Universal      | `*`              | All elements                            |
| Type           | `div`            | All `<div>` elements                    |
| Class          | `.card`          | Elements with `class="card"`            |
| ID             | `#header`        | Element with `id="header"`              |
| Attribute      | `[type="email"]` | Elements with matching attribute        |

### Attribute Selectors

| Selector           | Matches                                           |
|--------------------|---------------------------------------------------|
| `[attr]`           | Has the attribute (any value)                     |
| `[attr="val"]`     | Exact match                                       |
| `[attr~="val"]`    | Space-separated list contains `val`               |
| `[attr|="val"]`    | Equals `val` or starts with `val-`                |
| `[attr^="val"]`    | Starts with `val`                                 |
| `[attr$="val"]`    | Ends with `val`                                   |
| `[attr*="val"]`    | Contains `val` anywhere                           |
| `[attr="val" i]`   | Case-insensitive match (add `i` flag)             |

### Combinators

| Combinator    | Syntax  | Meaning                                           |
|---------------|---------|---------------------------------------------------|
| Descendant    | `A B`   | B anywhere inside A                               |
| Child         | `A > B` | B is a direct child of A                          |
| Adjacent      | `A + B` | B immediately follows A (same parent)             |
| General       | `A ~ B` | B follows A anywhere (same parent)                |

```css
/* all <p> inside .card */
.card p { }

/* only direct <p> children of .card */
.card > p { }

/* <p> immediately after <h2> */
h2 + p { }

/* any <p> after <h2> (same parent) */
h2 ~ p { }
```

### Pseudo-classes

**Structural:**

| Pseudo-class            | Matches                                      |
|-------------------------|----------------------------------------------|
| `:first-child`          | First child of its parent                    |
| `:last-child`           | Last child of its parent                     |
| `:nth-child(n)`         | nth child (`2n` = even, `2n+1` = odd, `3`)  |
| `:nth-last-child(n)`    | nth child from the end                       |
| `:only-child`           | Element with no siblings                     |
| `:first-of-type`        | First of its type among siblings             |
| `:nth-of-type(n)`       | nth of its type among siblings               |
| `:empty`                | Element with no children (including text)    |
| `:not(selector)`        | Negation                                     |
| `:is(selector)`         | Matches any selector in the list             |
| `:where(selector)`      | Like `:is()` but with zero specificity       |
| `:has(selector)`        | Parent selector — matches if contains match  |

**State:**

| Pseudo-class    | Matches                                          |
|-----------------|--------------------------------------------------|
| `:hover`        | Mouse is over the element                        |
| `:focus`        | Element has focus                                |
| `:focus-visible`| Element has focus AND it should be visually shown |
| `:focus-within` | Element or any descendant has focus              |
| `:active`       | Element is being activated (mousedown)           |
| `:visited`      | Already-visited link                             |
| `:checked`      | Checked checkbox/radio/option                    |
| `:disabled`     | Disabled form element                            |
| `:required`     | Form element with `required` attribute           |
| `:valid` / `:invalid` | Form validation state                     |
| `:placeholder-shown`  | Input currently showing placeholder text   |

### `:is()`, `:where()`, and `:has()`

```css
/* :is() — matches any selector in the list, takes highest specificity of the list */
:is(h1, h2, h3) { color: navy; }

/* :where() — same matching but ZERO specificity (easy to override) */
:where(h1, h2, h3) { color: navy; }

/* :has() — the "parent selector" (no equivalent before this) */
.card:has(img) { padding: 0; }           /* cards that contain an image */
.card:has(> .badge) { border: 2px solid; } /* cards with a direct .badge child */
```

### Pseudo-elements

| Pseudo-element   | Creates                                          |
|------------------|--------------------------------------------------|
| `::before`       | Inserted content before the element's children   |
| `::after`        | Inserted content after the element's children    |
| `::first-line`   | First rendered line of a block                   |
| `::first-letter` | First letter of a block                          |
| `::placeholder`  | Placeholder text in an input                     |
| `::selection`    | Text selected by the user                        |
| `::marker`       | Bullet/number of a list item                     |

`::before` and `::after` require the `content` property to be set (even if `content: ""`).

---

## Specificity

Specificity determines which CSS rule wins when multiple rules target the same element.

### Calculation

Specificity is a tuple `(A, B, C)`:

| Component | What counts                                      | Example               |
|-----------|--------------------------------------------------|-----------------------|
| A         | ID selectors                                     | `#header` → (1,0,0)  |
| B         | Class, attribute, pseudo-class selectors         | `.card` → (0,1,0)    |
| C         | Type, pseudo-element selectors                   | `div` → (0,0,1)      |

```css
/* (0,0,1) */
p { }

/* (0,1,0) */
.card { }

/* (0,1,1) */
p.card { }

/* (1,0,0) */
#header { }

/* (1,1,1) */
#header .nav a { }

/* (0,2,1) — :hover is a pseudo-class */
.card:hover p { }
```

### Special Cases

- **`*` (universal)** — specificity (0,0,0).
- **`:is()` / `:not()` / `:has()`** — take the specificity of their **most specific argument**.
- **`:where()`** — always (0,0,0), regardless of contents.
- **Inline styles** (`style="..."`) — beat any selector (conceptually (1,0,0,0)).
- **`!important`** — beats inline styles. `!important` rules compete among themselves by normal specificity.

### Cascade Order (simplified)

When specificity is equal, the rule that appears **later** in source order wins. Full cascade:

1. User-agent styles (browser defaults).
2. User styles.
3. Author styles (your CSS).
4. Author `!important`.
5. User `!important`.
6. User-agent `!important`.
7. CSS transitions.
8. CSS animations.

Within the same origin, specificity breaks ties, then source order.

### Cascade Layers (`@layer`)

Layers let you control cascade priority explicitly, independent of specificity:

```css
@layer reset, base, components, utilities;

@layer reset {
    * { margin: 0; }
}
@layer utilities {
    .mt-4 { margin-top: 1rem; } /* wins over reset even with lower specificity */
}
```

Unlayered styles always beat layered styles (unless `!important`).

---

## Common Interview Questions

### What is the difference between `content-box` and `border-box`?

With `content-box`, `width: 200px` + `padding: 20px` + `border: 2px` = 244px total. With `border-box`, the total stays
200px — content area shrinks to 156px to accommodate padding and border.

### How do you center an element horizontally and vertically?

```css
/* Flexbox */
.parent { display: flex; justify-content: center; align-items: center; }

/* Grid */
.parent { display: grid; place-items: center; }

/* Absolute positioning */
.child {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
}

/* Absolute + inset + auto margin */
.child {
    position: absolute;
    inset: 0;
    margin: auto;
    width: fit-content;
    height: fit-content;
}
```

### Why does `z-index` not work?

1. The element has `position: static` — `z-index` requires a positioned element (or flex/grid child).
2. The element's parent creates a stacking context with a lower `z-index` — child cannot escape it.
3. The competing element is in a different stacking context.

### What is BFC (Block Formatting Context)?

A BFC is an isolated layout region where:
- Floats are contained (no parent collapse).
- Margins don't collapse across the BFC boundary.
- The BFC doesn't overlap adjacent floats.

**Created by:** `overflow` other than `visible`, `display: flow-root`, `display: flex/grid`, `position: absolute/fixed`,
floats, inline-blocks, table cells.

The modern way to create a BFC: `display: flow-root`.

### How does `position: sticky` differ from `position: fixed`?

- `sticky` stays in the flow and only "sticks" when a scroll threshold is reached; stops sticking at the end of its
  containing block. `fixed` is always removed from the flow and positioned relative to the viewport.
- `sticky` respects its scrolling container; `fixed` respects the viewport (unless an ancestor has `transform`).

### What is the difference between `:nth-child()` and `:nth-of-type()`?

```html
<div>
    <h2>Title</h2>
    <p>First paragraph</p>
    <p>Second paragraph</p>
</div>
```

- `p:nth-child(2)` — matches the **second child** of the parent if it's a `<p>` (matches "First paragraph").
- `p:nth-of-type(2)` — matches the **second `<p>`** among siblings regardless of other element types (matches "Second
  paragraph").

### What does the `:has()` selector enable?

`:has()` is the first true "parent selector" in CSS. It lets you style an element based on its descendants:

```css
/* style a form that contains an invalid input */
form:has(:invalid) { border-color: red; }

/* style a card differently if it has no image */
.card:not(:has(img)) { padding: 2rem; }
```

Browser support: all modern browsers (Chrome 105+, Firefox 121+, Safari 15.4+).
