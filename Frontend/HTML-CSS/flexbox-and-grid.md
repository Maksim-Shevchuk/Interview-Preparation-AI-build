# Flexbox and Grid

Two core CSS layout modules. Flexbox is for one-dimensional layouts (row **or** column), Grid is for two-dimensional
layouts (rows **and** columns simultaneously). Common interview topics include when to use which, how axes work,
alignment, and auto-sizing.

---

## Flexbox

### Core Concept

A flex container distributes space among its children along the **main axis** and aligns them along the **cross axis**.

```css
.container {
    display: flex; /* or inline-flex */
}
```

### Axes and Direction

| Property         | Values                                           | Default      |
|------------------|--------------------------------------------------|--------------|
| `flex-direction` | `row`, `row-reverse`, `column`, `column-reverse` | `row`        |
| `flex-wrap`      | `nowrap`, `wrap`, `wrap-reverse`                 | `nowrap`     |
| `flex-flow`      | shorthand for `flex-direction` + `flex-wrap`     | `row nowrap` |

- `flex-direction: row` — main axis is horizontal (left-to-right in LTR).
- `flex-direction: column` — main axis is vertical (top-to-bottom).

### Container Alignment

| Property          | Axis  | What it does                                          |
|-------------------|-------|-------------------------------------------------------|
| `justify-content` | Main  | Distributes free space along the main axis            |
| `align-items`     | Cross | Aligns all items along the cross axis                 |
| `align-content`   | Cross | Distributes rows/columns when `flex-wrap: wrap`       |
| `gap`             | Both  | Gaps between flex items                               |

**`justify-content`** — `flex-start` | `flex-end` | `center` | `space-between` | `space-around` | `space-evenly`

**`align-items`** — `flex-start` | `flex-end` | `center` | `baseline` | `stretch` (default)

### Flex Item Properties

```css
.item {
    flex-grow: 0;     /* share of free space the item takes */
    flex-shrink: 1;   /* how much the item can shrink */
    flex-basis: auto; /* initial size before space distribution */
    /* shorthand: */
    flex: 0 1 auto;   /* grow shrink basis */
}
```

| Shorthand    | Equivalent  | When to use                              |
|--------------|-------------|------------------------------------------|
| `flex: 1`    | `1 1 0%`   | Item stretches to fill available space    |
| `flex: auto` | `1 1 auto` | Stretches but respects content size       |
| `flex: none` | `0 0 auto` | Fixed size, does not shrink               |
| `flex: 0`    | `0 1 0%`   | Minimum size                             |

**`align-self`** — overrides `align-items` for a specific item.

**`order`** — visual order (default `0`). Does not change DOM order — only affects visual rendering.

### Common Patterns

**Centering on both axes:**

```css
.center {
    display: flex;
    justify-content: center;
    align-items: center;
}
```

**Sticky footer:**

```css
body {
    display: flex;
    flex-direction: column;
    min-height: 100vh;
}
main {
    flex: 1; /* takes all remaining space */
}
```

**Nav with an item pushed to the right:**

```css
nav {
    display: flex;
    gap: 1rem;
}
nav .logout {
    margin-left: auto; /* auto margin pushes the item to the edge */
}
```

---

## Grid

### Core Concept

A grid container creates a **two-dimensional grid** of rows and columns. Children are placed into cells or can span
multiple cells.

```css
.container {
    display: grid; /* or inline-grid */
}
```

### Defining the Grid

```css
.grid {
    grid-template-columns: 200px 1fr 1fr; /* 3 columns */
    grid-template-rows: auto 1fr auto;     /* 3 rows */
    gap: 16px;                             /* gaps between cells */
}
```

### Units and Functions

| Unit / Function       | Description                                                       |
|-----------------------|-------------------------------------------------------------------|
| `fr`                  | Fraction of free space (`1fr 2fr` → 1/3 and 2/3)                |
| `repeat(3, 1fr)`     | Shorthand for `1fr 1fr 1fr`                                     |
| `minmax(200px, 1fr)` | Minimum 200px, maximum is a fraction of free space               |
| `auto-fill`          | Creates as many columns as fit (empty tracks are preserved)       |
| `auto-fit`           | Like `auto-fill`, but empty tracks collapse to 0                 |
| `fit-content(300px)` | Size to content, but no larger than 300px                        |

**Responsive grid without media queries:**

```css
.grid {
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
}
```

### Placing Items

```css
.item {
    grid-column: 1 / 3;     /* from line 1 to line 3 (spans 2 columns) */
    grid-row: 1 / 2;        /* spans 1 row */
    /* or shorthand: */
    grid-area: 1 / 1 / 2 / 3; /* row-start / col-start / row-end / col-end */
}
```

**`span`** — explicitly specify how many cells to occupy:

```css
.item {
    grid-column: span 2; /* spans 2 columns from auto-placed position */
}
```

### Named Areas (grid-template-areas)

```css
.layout {
    display: grid;
    grid-template-columns: 200px 1fr;
    grid-template-rows: auto 1fr auto;
    grid-template-areas:
        "header  header"
        "sidebar content"
        "footer  footer";
}

.header  { grid-area: header; }
.sidebar { grid-area: sidebar; }
.content { grid-area: content; }
.footer  { grid-area: footer; }
```

Area names must form a **rectangle**. Use `.` for empty cells.

### Grid Alignment

| Property          | Level     | Axis              | What it does                                  |
|-------------------|-----------|--------------------|-----------------------------------------------|
| `justify-items`   | Container | Inline (horizontal) | Aligns cell contents horizontally            |
| `align-items`     | Container | Block (vertical)    | Aligns cell contents vertically              |
| `place-items`     | Container | Both               | Shorthand for `align-items justify-items`     |
| `justify-content` | Container | Inline             | Distributes the entire grid within container  |
| `align-content`   | Container | Block              | Distributes the entire grid within container  |
| `justify-self`    | Item      | Inline             | Overrides `justify-items` for a single item   |
| `align-self`      | Item      | Block              | Overrides `align-items` for a single item     |

### Implicit Tracks

When items are placed beyond the explicitly defined grid, the browser creates **implicit tracks**:

```css
.grid {
    grid-template-columns: 1fr 1fr;
    grid-auto-rows: minmax(100px, auto); /* size of implicit rows */
    grid-auto-flow: row;                 /* row | column | dense */
}
```

`grid-auto-flow: dense` — the browser fills empty cells even if it breaks DOM order.

---

## Flexbox vs Grid: When to Use Which

| Criterion                      | Flexbox                              | Grid                                   |
|--------------------------------|--------------------------------------|----------------------------------------|
| Dimensionality                 | One-dimensional (row or column)      | Two-dimensional (rows and columns)     |
| Layout control                 | Content-first                        | Layout-first                           |
| Responsive without media queries | Limited                            | `auto-fit` / `auto-fill` + `minmax`    |
| Overlapping items              | No (only via `position`)            | Yes (items can overlap)                |
| Named areas                    | No                                   | `grid-template-areas`                  |
| Alignment on both axes         | Only via nested flex containers      | Native                                 |

**Rule of thumb:**
- **Flexbox** — navigation, toolbars, card rows, form elements, any **linear** flow.
- **Grid** — full page layouts, table-like data, galleries, dashboards — anything that needs control on **both axes**.
- They combine well: Grid for macro page layout, Flexbox for micro layout inside components.

---

## Common Interview Questions

### What is the difference between `auto-fill` and `auto-fit`?

- **`auto-fill`** — creates as many columns as fit. If there are fewer items, empty tracks are **preserved** (they occupy space).
- **`auto-fit`** — same, but empty tracks **collapse to 0**, and existing items stretch.

```css
/* auto-fill: 3 items but 5 columns fit — 2 empty tracks remain */
grid-template-columns: repeat(auto-fill, minmax(100px, 1fr));

/* auto-fit: 3 items stretch to fill the entire width */
grid-template-columns: repeat(auto-fit, minmax(100px, 1fr));
```

### How does `flex-grow` distribute space?

`flex-grow` works with the **remaining** free space (after accounting for `flex-basis`). If the container is 600px, two
items with `flex-basis: 100px` and `flex-grow: 1` and `flex-grow: 3`:

1. Free space: `600 - 100 - 100 = 400px`.
2. First item: `100 + 400 × (1/4) = 200px`.
3. Second item: `100 + 400 × (3/4) = 400px`.

### What are `min-content`, `max-content`, and `fit-content`?

- **`min-content`** — the smallest size without overflow (text wraps at every word break).
- **`max-content`** — the size where content renders in a single line with no wrapping.
- **`fit-content(X)`** — `min(max-content, max(min-content, X))` — sizes to content, but no larger than X.

### Why won't a flex item shrink below its content?

By default, `min-width: auto` on flex items means the item cannot be smaller than its content. Fix:

```css
.item {
    min-width: 0;     /* allows shrinking below content size */
    overflow: hidden; /* or clip the content */
}
```

### How does `z-index` work in Grid?

Grid items can overlap (unlike Flexbox). Stacking order is determined by:
1. `z-index` (if set).
2. DOM order (last element on top).

Grid items create a stacking context without needing `position: relative`.

### How to make an item span the full grid width?

```css
.full-width {
    grid-column: 1 / -1; /* from first to last line */
}
```

`-1` refers to the last explicit grid line.

---

## Subgrid (Level 2)

Allows a nested grid container to **inherit** the parent grid's tracks:

```css
.parent {
    display: grid;
    grid-template-columns: 1fr 2fr 1fr;
}

.child {
    display: grid;
    grid-column: 1 / 4;
    grid-template-columns: subgrid; /* inherits parent's 3 columns */
}
```

Solves the problem of aligning nested elements to a shared grid (e.g., cards with headers on the same line).

Browser support: all modern browsers (Chrome 117+, Firefox 71+, Safari 16+).
