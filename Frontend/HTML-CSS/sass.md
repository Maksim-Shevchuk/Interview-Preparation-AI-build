# SASS (Syntactically Awesome Style Sheets)

Sass is a **CSS preprocessor** — a scripting language that compiles down to plain CSS. It adds variables, nesting,
mixins, functions, control flow, and modular file splitting on top of CSS, so stylesheets become DRY, maintainable,
and programmable. Interview questions usually focus on what Sass solves, the **SCSS vs indented** syntax,
mixin-vs-function trade-offs, the modern `@use` module system (vs the old `@import`), and how much of Sass is still
relevant now that native CSS has variables, nesting, and `@layer`.

---

## Why Sass Still Matters

Native CSS has caught up on several fronts — `:root` variables ("custom properties"), native nesting,
`@layer`, `@scope`, and `color-mix()`. But Sass still adds value for:

- **Compile-time computation** — Sass runs during build; CSS custom properties are evaluated at runtime in the browser
  (which means they can't drive `@media` breakpoints, can't be interpolated into selectors, can't be reused inside
  static asset URLs).
- **Loops and conditionals** — generate families of utility classes (`mt-1 … mt-12`) without copy-paste.
- **Mixins with arguments and `@content` blocks** — encapsulate patterns that produce *multiple* declarations
  (clearfix, visually-hidden, BEM modifiers).
- **Functions** — compute a single value (`strip-unit`, `lighten`/`darken`, color contrast).
- **Modular code organization** — split stylesheets into partials and assemble them through `@use`/`@forward`.

Rule of thumb in modern codebases: use **native CSS custom properties** for runtime theming and things that need to
change in the browser (dark mode, user preference), and use **Sass** for static, build-time generation, mixins, and
large-scale file organization.

---

## Syntaxes: SCSS vs Sass (Indented)

Sass has **two** interchangeable syntaxes — both compile to identical CSS.

| Aspect              | SCSS (`.scss`)                       | Sass (`.sass`, indented)          |
|---------------------|--------------------------------------|-----------------------------------|
| Braces / semicolons | Required (it's a superset of CSS)    | None — indentation defines blocks |
| Learning curve      | Familiar to CSS authors              | Python/Haml-like                  |
| Existing CSS reuse  | Paste valid CSS in → it's valid SCSS | Must convert syntax               |
| Adoption            | Default in most projects             | Minority, used by Ruby ecosystem  |

```scss
// SCSS
.button {
  background: $primary;

  &:hover {
    background: darken($primary, 10%);
  }
}
```

```sass
// Sass (indented)
.button
  background: $primary

  &:hover
    background: darken($primary, 10%)
```

Everything below uses **SCSS** — it's the default and what 99% of jobs mean by "Sass".

---

## Core Features

### 1. Variables

Sass variables are **compile-time only** and exist at any scope. They don't appear in devtools and can't be tweaked
in the browser.

```scss
$brand-primary: #2d6cdf;
$grid-gap: clamp(1rem, 2vw, 2rem);
$breakpoint-md: 768px;

.hero {
  color: $brand-primary;
  margin-inline: $grid-gap;
}
```

Compare with native CSS custom properties — runtime-mutable, theme-aware:

```css
:root {
    --brand-primary: #2d6cdf;
}

.dark {
    --brand-primary: #6fa0ff;
}

/* overrides in scope */

.hero {
    color: var(--brand-primary);
}

/* reactively picks up `.dark` */
```

**Use Sass variables for compile-time constants** (spacing scales, breakpoints), CSS custom properties for
**runtime theming** (dark mode, user preferences).

### 2. Nesting

```scss
.card {
  background: white;

  &__title {
    font-size: 1.25rem;
  }

  &:hover {
    box-shadow: 0 8px 24px rgba(0, 0, 0, .12);
  }

  &--featured {
    border-color: gold;
  }
}
```

Compiles to:

```css
.card {
    background: white;
}

.card__title {
    font-size: 1.25rem;
}

.card:hover {
    box-shadow: 0 8px 24px rgba(0, 0, 0, .12);
}

.card--featured {
    border-color: gold;
}
```

**Pitfall:** deep nesting produces over-specific selectors. Cap at 2–3 levels; flat BEM beats `nav > ul > li > a`.

### 3. Partials and the `@use` module system

Split into `_*` partials and import as modules. `@use` namespaces imports and runs each file only once — the
modern, recommended replacement for legacy `@import`.

```scss
// _variables.scss
$primary: #2d6cdf;
```

```scss
// _mixins.scss
@mixin center($gap: 1rem) {
  display: flex;
  place-items: center;
  gap: $gap;
}
```

```scss
// main.scss
@use 'variables' as v;
@use 'mixins'    as m;

.header {
  color: v.$primary;
  @include m.center(2rem);
}
```

`@forward` re-exports a partial through an index so consumers see one entry point:

```scss
// _index.scss
@forward 'variables';
@forward 'mixins';
@forward 'buttons';
```

```scss
@use 'styles' as *; // picks up everything _index forwards
```

### 4. Mixins

Reusable blocks of **declarations**. May take arguments, default values, arbitrary keyword args, and `@content`
blocks (for wrapping selectors or media queries).

```scss
@mixin visually-hidden {
  position: absolute !important;
  width: 1px;
  height: 1px;
  overflow: hidden;
  clip: rect(0 0 0 0);
  white-space: nowrap;
}

.icon-label {
  @include visually-hidden;
}
```

```scss
@mixin mq($from, $to: null) {
  @if $to == null {
    @media (min-width: $from) {
      @content;
    }
  } @else {
    @media (min-width: $from) and (max-width: $to) {
      @content;
    }
  }
}

.sidebar {
  display: none;
  @include mq(768px) {
    display: block;
    width: 240px;
  }
}
```

### 5. Functions

A mixin **emits CSS declarations**; a function **returns a value** you interpolate elsewhere.

```scss
@function strip-unit($n) {
  @return $n / ($n * 0 + 1); // 16px / 1 == 16 (unitless)
}

@function rem($px, $base: 16px) {
  @return strip-unit($px) / strip-unit($base) * 1rem;
}

.h1 {
  font-size: rem(32px);
}

// 2rem
.h2 {
  font-size: rem(24px);
}

// 1.5rem
```

### 6. Control flow

```scss
@for $i from 1 through 12 {
  .mt-#{$i} {
    margin-top: $i * 0.25rem;
  }
}

$weights: low, medium, high;
@each $w in $weights {
  .priority-#{$w} {
    border-left: 3px solid map-get($priority-colors, $w);
  }
}

@function contrast-text($bg) {
  @if luminance($bg) > 50% {
    @return #000;
  } @else {
    @return #fff;
  }
}
```

### 7. Maps and lists

```scss
$theme: (
        primary: #2d6cdf,
        accent: #f59e0b,
        danger: #dc2626,
);

.button--primary {
  background: map-get($theme, primary);
}

.button--accent {
  background: map-get($theme, accent);
}

@each $name, $color in $theme {
  .text-#{$name} {
    color: $color;
  }
}
```

### 8. `@extend` and placeholder selectors

`@extend` shares declarations across selectors — produces grouped selectors, smaller output. **Use sparingly**; it
can over-couple unrelated rules and break in scoped environments (CSS modules, Vue).

```scss
%card-base { // placeholder — not emitted by itself
  padding: 1.25rem;
  border-radius: 12px;
  background: white;
}

.profile-card {
  @extend %card-base;
  box-shadow: 0 2px 6px black;
}

.pricing-card {
  @extend %card-base;
  border-top: 4px solid gold;
}
```

```css
/* Output — both rules grouped under the shared base */
.profile-card,
.pricing-card {
    padding: 1.25rem;
    border-radius: 12px;
    background: white;
}

.profile-card {
    box-shadow: 0 2px 6px black;
}

.pricing-card {
    border-top: 4px solid gold;
}
```

Prefer **mixins over `@extend`** when the shared block is small or when output clarity matters more than byte size.

---

## Code Generation Patterns

The headline reason to reach for Sass over hand-written CSS is **generating families of tokens, utilities, or themed
components programmatically**.

### 1. Spacing / typography scale (`rem`-based loop)

```scss
// _scale.scss
$step: 0.25rem; // 4px

@for $i from 0 through 16 {
  .m-#{$i} {
    margin: $i * $step;
  }
  .mt-#{$i} {
    margin-top: $i * $step;
  }
  .p-#{$i} {
    padding: $i * $step;
  }
  .pt-#{$i} {
    padding-top: $i * $step;
  }
}
```

Generated output (excerpt):

```css
.m-0 {
    margin: 0;
}

.m-1 {
    margin: 0.25rem;
}

.mt-2 {
    margin-top: 0.5rem;
}

/* …16 variants for each property */
```

Hand-writing 64 utility classes is error-prone; the loop guarantees a single source of truth.

### 2. Theme map → BEM component variants

```scss
$component-themes: (
        primary: #2d6cdf,
        secondary: #6b7280,
        success: #16a34a,
        warn: #f59e0b,
        danger: #dc2626,
);

@mixin button-variant($color) {
  --btn-bg: #{$color};
  --btn-fg: white;
  --btn-hover: #{color-mix(in srgb, $color 85%, black)};

  background: var(--btn-bg);
  color: var(--btn-fg);
  &:hover {
    background: var(--btn-hover);
  }
}

@each $name, $color in $component-themes {
  .button--#{$name} {
    @include button-variant($color);
  }
}
```

Output:

```css
.button--primary {
    --btn-bg: #2d6cdf;
    --btn-fg: #fff;
    --btn-hover: #265bb8;
    background: var(--btn-bg);

    ...
}

.button--secondary {
    ...
}

.button--danger {
    ...
}
```

Notice the hybrid: Sass generates the **variants** at build time; CSS custom properties inside each variant enable
**runtime hover state** and `prefers-color-scheme` overrides without regenerating CSS.

### 3. Responsive breakpoints generated from a config map

```scss
$breakpoints: (
        sm: 640px,
        md: 768px,
        lg: 1024px,
        xl: 1280px,
);

@mixin from($key) {
  $bp: map-get($breakpoints, $key);
  @media (min-width: $bp) {
    @content;
  }
}

.grid {
  display: grid;
  grid-template-columns: 1fr;
  @include from(md) {
    grid-template-columns: repeat(2, 1fr);
  }
  @include from(lg) {
    grid-template-columns: repeat(3, 1fr);
  }
}
```

Adding a new breakpoint is a one-line change to `$breakpoints`.

### 4. Icon system from a folder scan

Sass can't read the filesystem, but a build script can. Drop a Node script next to the stylesheets and let Sass
consume its output:

```scss
// _icons.scss (generated by scripts/gen-icons.js → writes this file)
$icons: (
        arrow: "<svg ...>...</svg>",
        check: "<svg ...>...</svg>",
        close: "<svg ...>...</svg>",
);
```

```js
// scripts/gen-icons.js
const fs = require('fs');
const path = require('path');

const dir = 'src/icons';
const files = fs.readdirSync(dir).filter(f => f.endsWith('.svg'));

const body = files.map(f => {
    const name = path.basename(f, '.svg');
    const svg = fs.readFileSync(path.join(dir, f), 'utf8')
        .replace(/\s+/g, ' ')
        .replace(/"/g, '\\"');
    return `  ${name}: "${svg}",`;
}).join('\n');

fs.writeFileSync('src/styles/_icons.scss',
    `$icons: (\n${body}\n);\n`);
```

```scss
@use 'icons' as i;

@each $name, $svg in i.$icons {
  .icon-#{$name} {
    width: 1.25em;
    height: 1.25em;
    background: url('data:image/svg+xml;utf8,#{i.$svg}') no-repeat center / contain;
    /* encode for URL safety */
    @if str-index($svg, '#') {
      background-image: url("data:image/svg+xml;utf8,#{url-encode($svg)}");
    }
  }
}
```

This turns **"add an icon → drop a file"** into the entire class system without touching the stylesheet by hand —
a common interview talking point about how build steps extend Sass beyond its own syntax.

### 5. Generating a CSS-in-JS-style API contract

For a design system shared between Sass and TypeScript (e.g., Storybook + React), let **Sass be the source of truth**
and emit the same tokens to JSON:

```scss
// tokens.scss — the design system entry
$tokens: (
        color: (primary: #2d6cdf, danger: #dc2626),
        spacing: (sm: 0.5rem, md: 1rem, lg: 1.5rem),
);

@each $category, $pairs in $tokens {
  @each $name, $value in $pairs {
    :root {
      --#{$category}-#{$name}: #{$value};
    }
  }
}
```

```js
// scripts/extract-tokens.js — reads tokens.scss via sass API
const sass = require('sass');
const result = sass.renderSync({file: 'src/styles/tokens.scss'});
// parse @use exports or simply maintain a parallel JSON exported from Sass doc-blocks
fs.writeFileSync('src/tokens.json', JSON.stringify(parsed, null, 2));
```

Now `<Button color="primary">` in React and `.button--primary` in Sass reference the **same token**, sourced from the
same file. Drift is impossible.

---

## Build & Tooling

| Tool                   | Use                                                                                 |
|------------------------|-------------------------------------------------------------------------------------|
| `sass` (Dart Sass)     | Reference implementation, recommended; `npm i -g sass`                              |
| `sass-loader` / `vite` | Bundler integration — auto-recompile on save                                        |
| `stylelint`            | Lint rules: `no-duplicate-variables`, `no-vendor-prefix`                            |
| `postcss-preset-env`   | Pipe compiled CSS through autoprefixer / nesting polyfill                           |
| `@use`/`@forward`      | Module system (legacy `@import` is deprecated and will be removed in Dart Sass 3.0) |

```bash
sass src/styles/main.scss dist/styles.css --no-source-map --style=compressed
# Watch mode
sass --watch src/styles/main.scss:dist/styles.css
```

---

## Modern Sass vs Modern CSS — When to Use What

| Need                                      | Use Sass         | Use native CSS                             |
|-------------------------------------------|------------------|--------------------------------------------|
| Runtime-switchable theme color            | ✗ (compile-time) | ✓ custom properties                        |
| Compute value from another (lighten, mix) | ✓ functions      | partial — `color-mix()`                    |
| Loop to generate N utility classes        | ✓ `@for`         | ✗ (no way yet)                             |
| Conditional styles by breakpoint          | ✓ `@if` / mixins | partial — `@media`                         |
| Dark mode                                 | ✓ or ✗           | ✓ `prefers-color-scheme`                   |
| Split stylesheets across components       | ✓ `@use`         | partial — `@import` is gone in newer specs |
| Hash-based scoping / CSS modules          | ✗                | ✓ (bundler feature)                        |

**Pragmatic answer for an interview:** "Sass for build-time code generation, native CSS custom properties for runtime
theming — combine both; **don't** use Sass variables where you need darkness-/user-driven switches at runtime."

---

## Anti-Patterns to Avoid

- **Deep `@import` chains** — replaced by `@use`; legacy `@import` is global-only, runs files multiple times, will be
  removed in Dart Sass 3.0.
- **`@extend` chains across files** — produces surprising groupings and breakage under CSS modules; prefer mixins.
- **Sass variables for theming** — they can't react to `prefers-color-scheme`; use custom properties.
- **Nesting deeper than 3 levels** — selector specificity explodes; use BEM or scoped CSS modules instead.
- **Hand-writing utility classes** — if you have more than 5 variants of a property, generate them with a loop.
- **Browser-only logic in Sass** — anything depending on user interaction, JS state, or DOM attributes can't live
  in a preprocessor.

---

## Common Interview Questions

- What is Sass and what problems does it solve over plain CSS?
- SCSS vs indented Sass — differences, when to choose each.
- Compare Sass variables with CSS custom properties. When would you use which?
- Explain `@use` vs `@import` vs `@forward`. Why is `@import` deprecated?
- Difference between a mixin and a function? Provide a use case for each.
- When would you avoid `@extend`?
- How would you implement a responsive spacing scale that matches the design system?
- Show how to generate an icon system from a folder of SVG files.
- How do you keep a design token in sync between Sass and TypeScript?
- What native CSS features would you reach for now instead of Sass?

---

## Related

- [Semantic HTML](./semantic-html.md) — Sass styles semantic structure; classes are behavior-free.
- [Flexbox and Grid](./flexbox-and-grid.md) — the layout primitives you compose with Sass tokens.
- [Box Model, Positioning, Selectors](./box-model-positioning-selectors.md) — selector specificity that nesting affects.
- [Web Performance](../Web-Performance/web-performance.md) — generated CSS size and runtime cost.

---

## Resources

- *Sass Documentation* — https://sass-lang.com/documentation
- *Dart Sass migration guides* — `@import` → `@use`, division operator deprecation.
- *MDN: CSS Custom Properties* — https://developer.mozilla.org/en-US/docs/Web/CSS/--*
- *CSS Tricks: An Introduction to Sass* — long-form intro.
- *Sass Guidelines* by Kitty Giraudel — opinionated, widely cited style guide.