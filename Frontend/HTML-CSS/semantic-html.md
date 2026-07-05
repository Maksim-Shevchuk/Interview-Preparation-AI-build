# Semantic HTML

Semantic HTML means using elements that convey **meaning** about the content they contain, rather than just visual
presentation. Interviews focus on knowing the right elements, understanding accessibility implications, and explaining
why semantics matter for SEO and screen readers.

---

## Why Semantics Matter

1. **Accessibility** — screen readers use semantic tags to build an outline and navigate the page (e.g., jump between
   headings, skip to `<main>`, list all `<nav>` landmarks).
2. **SEO** — search engines weight content inside `<article>`, `<h1>`–`<h6>`, `<main>` higher than generic `<div>` soup.
3. **Readability** — developers understand the document structure at a glance.
4. **Maintainability** — semantic markup is self-documenting; less need for class-name conventions to convey purpose.

---

## Document Structure Elements

### `<header>`

Introductory content for its nearest sectioning ancestor or the page itself. Can contain navigation, logos, search bars.

```html
<header>
    <h1>My App</h1>
    <nav>...</nav>
</header>
```

A page can have **multiple** `<header>` elements — e.g., one for the page and one inside an `<article>`.

### `<nav>`

A section with **navigation links** — primary menu, breadcrumbs, table of contents, pagination.

```html
<nav aria-label="Main navigation">
    <ul>
        <li><a href="/">Home</a></li>
        <li><a href="/about">About</a></li>
    </ul>
</nav>
```

Not every group of links needs `<nav>` — only major navigation blocks. A footer link list is usually fine as a plain
`<ul>`.

### `<main>`

The **dominant content** of the page. There must be only **one visible `<main>`** per page (unless others are hidden
with `hidden` attribute). Skip-navigation links typically target `<main>`.

```html
<body>
    <header>...</header>
    <main id="content">
        <!-- primary page content -->
    </main>
    <footer>...</footer>
</body>
```

### `<footer>`

Footer for its nearest sectioning ancestor or the page. Typically contains copyright, contact info, related links.

### `<aside>`

Content **tangentially related** to the surrounding content — sidebars, pull quotes, ads, related links. Screen readers
can skip it as non-essential.

```html
<article>
    <p>Main article text...</p>
    <aside>
        <p>Fun fact related to the article.</p>
    </aside>
</article>
```

---

## Sectioning Elements

### `<section>`

A **thematic grouping** of content, typically with a heading. Use when the content forms a logical section but is not
standalone enough for `<article>`.

```html
<section>
    <h2>Features</h2>
    <p>...</p>
</section>
```

### `<article>`

A **self-contained, independently distributable** piece of content — blog post, news article, comment, widget. Should
make sense on its own if extracted from the page.

```html
<article>
    <h2>Understanding Flexbox</h2>
    <p>Flexbox is a one-dimensional layout...</p>
    <footer>Published on 2025-01-15</footer>
</article>
```

Articles can be **nested** — e.g., blog post (`<article>`) with comments (each also `<article>`).

### `<section>` vs `<article>` vs `<div>`

| Element     | When to use                                                   |
|-------------|---------------------------------------------------------------|
| `<article>` | Self-contained content that makes sense on its own            |
| `<section>` | Thematic grouping within a page, usually with a heading       |
| `<div>`     | No semantic meaning — use only for styling/layout wrappers    |

**Rule of thumb:** if you would put it in an RSS feed, it's an `<article>`. If it groups related content under a
heading, it's a `<section>`. If it's just a styling hook, it's a `<div>`.

---

## Text-Level Semantics

| Element        | Meaning                                              | Not the same as          |
|----------------|------------------------------------------------------|--------------------------|
| `<strong>`     | Strong importance / urgency                          | `<b>` (stylistic bold)   |
| `<em>`         | Stress emphasis (changes sentence meaning)           | `<i>` (stylistic italic) |
| `<mark>`       | Highlighted / relevant in current context            | —                        |
| `<small>`      | Side comments, legal text, fine print                | —                        |
| `<time>`       | Machine-readable date/time                           | —                        |
| `<abbr>`       | Abbreviation with optional `title` expansion         | —                        |
| `<cite>`       | Title of a referenced work                           | —                        |
| `<code>`       | Inline code fragment                                 | —                        |
| `<kbd>`        | User keyboard input                                  | —                        |
| `<samp>`       | Sample program output                                | —                        |
| `<blockquote>` | Block-level quotation (with optional `cite` attr)    | `<q>` (inline quote)     |
| `<del>` / `<ins>` | Deleted / inserted text (edit tracking)           | `<s>` (no longer relevant) |

### `<b>` / `<i>` vs `<strong>` / `<em>`

- `<b>` — draw attention without conveying extra importance (keywords, product names).
- `<i>` — alternative voice (foreign words, technical terms, thoughts).
- `<strong>` — content is **important** (warnings, key phrases).
- `<em>` — content has **stress emphasis** that changes meaning ("I *love* cats" vs "I love *cats*").

---

## Forms and Interactive Elements

### `<label>`

Associates text with a form control. Critical for accessibility — clicking the label focuses/toggles the input.

```html
<!-- explicit association -->
<label for="email">Email</label>
<input id="email" type="email" />

<!-- implicit association -->
<label>
    Email
    <input type="email" />
</label>
```

### `<fieldset>` and `<legend>`

Groups related form controls with a descriptive caption:

```html
<fieldset>
    <legend>Shipping Address</legend>
    <label>Street <input type="text" /></label>
    <label>City <input type="text" /></label>
</fieldset>
```

### `<output>`

Result of a calculation or user action:

```html
<form oninput="result.value = parseInt(a.value) + parseInt(b.value)">
    <input type="number" id="a" /> +
    <input type="number" id="b" /> =
    <output name="result" for="a b">0</output>
</form>
```

### `<details>` and `<summary>`

Native disclosure widget — no JavaScript needed:

```html
<details>
    <summary>Show more info</summary>
    <p>Extra content revealed on click.</p>
</details>
```

### `<dialog>`

Native modal/non-modal dialog with built-in focus trapping and backdrop:

```html
<dialog id="confirm">
    <p>Are you sure?</p>
    <button>Yes</button>
    <button>No</button>
</dialog>
```

```javascript
document.getElementById('confirm').showModal(); // modal with backdrop
```

---

## Media and Figures

### `<figure>` and `<figcaption>`

Self-contained content (image, diagram, code listing) with an optional caption:

```html
<figure>
    <img src="chart.png" alt="Sales growth Q1-Q4" />
    <figcaption>Figure 1: Quarterly sales growth in 2024.</figcaption>
</figure>
```

### `<picture>`

Art-direction and format selection for images:

```html
<picture>
    <source media="(min-width: 800px)" srcset="hero-wide.webp" type="image/webp" />
    <source media="(min-width: 800px)" srcset="hero-wide.jpg" />
    <source srcset="hero-narrow.webp" type="image/webp" />
    <img src="hero-narrow.jpg" alt="Hero banner" />
</picture>
```

### `<video>` / `<audio>`

Always provide fallback content and accessible alternatives:

```html
<video controls>
    <source src="demo.mp4" type="video/mp4" />
    <track kind="captions" src="captions.vtt" srclang="en" label="English" />
    Your browser does not support video.
</video>
```

---

## Tables

Tables are semantic when used for **tabular data** (not layout).

```html
<table>
    <caption>Monthly Revenue</caption>
    <thead>
        <tr>
            <th scope="col">Month</th>
            <th scope="col">Revenue</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>January</td>
            <td>$10,000</td>
        </tr>
    </tbody>
    <tfoot>
        <tr>
            <td>Total</td>
            <td>$120,000</td>
        </tr>
    </tfoot>
</table>
```

Key attributes: `scope="col"` / `scope="row"` on `<th>` — tells screen readers the header's direction.

---

## Headings Hierarchy

- Use **one `<h1>`** per page (the page title).
- Headings must not skip levels — `<h1>` → `<h2>` → `<h3>`, not `<h1>` → `<h3>`.
- Each sectioning element (`<section>`, `<article>`) can restart at `<h2>` (or even `<h1>` per the outline algorithm,
  but in practice browsers never implemented the outline algorithm, so **keep a flat heading hierarchy**).

---

## ARIA: When Semantics Are Not Enough

ARIA (Accessible Rich Internet Applications) attributes supplement HTML semantics for custom widgets:

| ARIA Attribute      | Purpose                                                     |
|---------------------|-------------------------------------------------------------|
| `role`              | Overrides the element's implicit role (`role="tablist"`)    |
| `aria-label`        | Provides an accessible name when no visible text exists     |
| `aria-labelledby`   | Points to the ID of another element that labels this one    |
| `aria-describedby`  | Points to additional descriptive text                       |
| `aria-hidden`       | Hides an element from the accessibility tree                |
| `aria-live`         | Announces dynamic content changes (`polite` / `assertive`) |
| `aria-expanded`     | Whether a collapsible section is open or closed             |
| `aria-required`     | Marks a form field as required                              |

**First rule of ARIA:** don't use ARIA if a native HTML element can do the job. `<button>` is always better than
`<div role="button" tabindex="0">`.

---

## Common Interview Questions

### Why not just use `<div>` and `<span>` for everything?

- Screen readers cannot derive meaning from `<div>` — users lose navigation landmarks, heading outlines, and form
  associations.
- SEO crawlers weight semantic elements higher.
- `<div>` soup requires more ARIA attributes to achieve the same accessibility as native elements, and is more fragile.

### What is the document outline algorithm?

The HTML5 spec defined an outline algorithm where each sectioning element (`<section>`, `<article>`, `<nav>`, `<aside>`)
could start its own heading hierarchy. **No browser ever implemented it.** In practice, always use a flat heading
hierarchy (`<h1>` → `<h2>` → `<h3>`).

### When should you use `<section>` vs `<div>`?

Use `<section>` when the content is a **thematic grouping** and you would give it a heading. If it's purely a layout
wrapper with no semantic meaning, use `<div>`.

### What is a landmark?

Landmarks are page regions that assistive technology can jump to directly:

| HTML Element | ARIA Role       |
|--------------|-----------------|
| `<header>`   | `banner`        |
| `<nav>`      | `navigation`    |
| `<main>`     | `main`          |
| `<aside>`    | `complementary` |
| `<footer>`   | `contentinfo`   |
| `<section>`  | `region` (if labeled) |
| `<form>`     | `form` (if labeled)   |

### What is the difference between `alt=""` and no `alt` attribute?

- `alt="description"` — describes the image for screen readers.
- `alt=""` — explicitly marks the image as **decorative** (screen readers skip it).
- No `alt` at all — **accessibility violation**; screen readers may read the file name instead.
