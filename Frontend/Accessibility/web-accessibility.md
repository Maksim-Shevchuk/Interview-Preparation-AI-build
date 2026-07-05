# Web Accessibility (a11y)

Accessibility ensures that websites and applications are usable by everyone, including people with visual, auditory,
motor, or cognitive disabilities. Interview questions typically cover WCAG principles, ARIA usage, keyboard navigation,
and practical techniques.

---

## WCAG Principles (POUR)

The Web Content Accessibility Guidelines are organized around four principles:

| Principle       | Meaning                                                                  |
|-----------------|--------------------------------------------------------------------------|
| **Perceivable** | Content must be presentable in ways users can perceive (alt text, captions, contrast) |
| **Operable**    | UI must be operable via keyboard, touch, voice — not just mouse          |
| **Understandable** | Content and UI behavior must be predictable and readable              |
| **Robust**      | Content must work across assistive technologies and browsers             |

### Conformance Levels

| Level | Description                                            | Example requirements                      |
|-------|--------------------------------------------------------|-------------------------------------------|
| A     | Minimum — removes the biggest barriers                 | Alt text, keyboard access, no seizure risk |
| AA    | Industry standard — most legal requirements target this| Color contrast 4.5:1, resize to 200%, focus visible |
| AAA   | Highest — not always achievable for all content        | Contrast 7:1, sign language for video     |

Most companies and regulations (ADA, EAA, Section 508) require **WCAG 2.1 Level AA**.

---

## Assistive Technologies

| Technology         | Users                          | How it works                              |
|--------------------|--------------------------------|-------------------------------------------|
| Screen readers     | Blind / low vision             | Read the accessibility tree aloud         |
| Screen magnifiers  | Low vision                     | Zoom portions of the screen               |
| Voice control      | Motor disabilities             | Navigate and interact via voice commands  |
| Switch devices     | Severe motor disabilities      | Navigate with one or two physical switches|
| Braille displays   | Blind                          | Output text as Braille characters         |

Popular screen readers: NVDA (Windows, free), JAWS (Windows), VoiceOver (macOS/iOS), TalkBack (Android).

---

## The Accessibility Tree

The browser builds an **accessibility tree** from the DOM — a parallel structure that assistive technologies consume.
Each node has:

- **Role** — what the element is (`button`, `link`, `heading`, `navigation`).
- **Name** — the accessible label ("Submit", "Main navigation").
- **State** — dynamic properties (`checked`, `expanded`, `disabled`).
- **Value** — current value (slider position, input text).

Semantic HTML produces a rich accessibility tree automatically. Generic elements (`<div>`, `<span>`) appear as
unlabeled, unstructured nodes.

DevTools → Accessibility tab lets you inspect the tree.

---

## Keyboard Accessibility

### Focus Order

- Interactive elements (`<a>`, `<button>`, `<input>`, `<select>`, `<textarea>`) are focusable by default.
- Focus order should follow **visual/logical reading order** — don't rearrange with `tabindex` values > 0.
- Non-interactive elements should generally not be focusable.

### `tabindex`

| Value         | Behavior                                                     |
|---------------|--------------------------------------------------------------|
| `tabindex="0"`  | Adds element to natural tab order (use for custom widgets) |
| `tabindex="-1"` | Focusable via JS (`element.focus()`) but not via Tab key   |
| `tabindex="1+"` | **Avoid** — forces element ahead in tab order, creates chaos |

### Focus Management

- **Focus trap** — keep focus within a modal/dialog (Tab cycles inside, not behind it).
- **Focus restore** — when a modal closes, return focus to the element that opened it.
- **Skip links** — hidden link at the top of the page that jumps to `<main>` on focus.

```html
<a href="#main-content" class="skip-link">Skip to main content</a>
```

```css
.skip-link {
    position: absolute;
    top: -40px;
    left: 0;
}
.skip-link:focus {
    top: 0; /* visible only when focused */
}
```

### Key Handlers for Custom Widgets

If you build a custom widget (tabs, dropdown, combobox), you must handle keyboard interactions:

| Widget    | Expected keys                                              |
|-----------|------------------------------------------------------------|
| Tabs      | Arrow keys to switch, Tab to move out                      |
| Menu      | Arrow keys to navigate, Enter/Space to select, Escape to close |
| Dialog    | Escape to close, Tab cycles inside (focus trap)            |
| Accordion | Enter/Space to toggle, Arrow keys between headers          |

Reference: [WAI-ARIA Authoring Practices](https://www.w3.org/WAI/ARIA/apg/patterns/).

---

## ARIA (Accessible Rich Internet Applications)

### The Five Rules of ARIA

1. **Don't use ARIA if native HTML works** — `<button>` over `<div role="button">`.
2. **Don't change native semantics** unless necessary — don't add `role="heading"` to a `<button>`.
3. **All interactive ARIA controls must be keyboard accessible.**
4. **Don't use `role="presentation"` or `aria-hidden="true"` on focusable elements.**
5. **All interactive elements must have an accessible name.**

### Key ARIA Attributes

**Roles:**

| Category  | Examples                                                         |
|-----------|------------------------------------------------------------------|
| Landmark  | `banner`, `navigation`, `main`, `complementary`, `contentinfo`  |
| Widget    | `button`, `tab`, `tabpanel`, `dialog`, `alert`, `slider`        |
| Structure | `list`, `listitem`, `table`, `row`, `cell`, `heading`           |
| Live      | `alert`, `log`, `status`, `timer`, `marquee`                    |

**States and Properties:**

| Attribute            | Purpose                                              |
|----------------------|------------------------------------------------------|
| `aria-label`         | Provides accessible name (no visible text)           |
| `aria-labelledby`    | Points to element(s) that provide the name           |
| `aria-describedby`   | Points to element(s) with additional description     |
| `aria-hidden="true"` | Removes from accessibility tree (still visible)      |
| `aria-expanded`      | Whether a collapsible section is open/closed         |
| `aria-selected`      | Whether an item is selected (tabs, listbox)          |
| `aria-pressed`       | Toggle button state                                  |
| `aria-controls`      | ID of the element this control affects               |
| `aria-live`          | Announces dynamic changes: `polite` or `assertive`   |
| `aria-atomic`        | Whether the entire region is announced on change     |
| `aria-relevant`      | What types of changes to announce (additions, removals, text) |
| `aria-current`       | Current item in a set (`page`, `step`, `date`, etc.) |
| `aria-invalid`       | Whether the input has a validation error             |
| `aria-errormessage`  | ID of the element containing the error message       |
| `aria-required`      | Whether a form field is required                     |
| `aria-busy`          | Region is being updated, wait before announcing      |
| `role="presentation"` / `role="none"` | Strips semantic meaning            |

### Live Regions

For dynamic content updates that screen readers should announce:

```html
<!-- polite: announced after current speech finishes -->
<div aria-live="polite" aria-atomic="true">
    3 items in cart
</div>

<!-- assertive: interrupts current speech immediately -->
<div role="alert">
    Form submission failed. Please check your input.
</div>
```

`role="alert"` is implicitly `aria-live="assertive"`.

---

## Color and Contrast

### Contrast Ratios (WCAG AA)

| Content Type           | Minimum ratio |
|------------------------|---------------|
| Normal text (< 18px)   | 4.5:1        |
| Large text (≥ 18px bold or ≥ 24px) | 3:1 |
| UI components / graphical objects | 3:1 |

### Best Practices

- Never use color as the **only** way to convey information — add icons, patterns, or text labels.
- Ensure focus indicators have sufficient contrast against the background.
- Test with simulated color blindness (DevTools → Rendering → Emulate vision deficiencies).
- Provide a visible focus style — don't use `outline: none` without a replacement.

```css
/* good focus style */
:focus-visible {
    outline: 2px solid #0066cc;
    outline-offset: 2px;
}
```

---

## Images and Media

### Alt Text Guidelines

| Image Type    | `alt` value                                                      |
|---------------|------------------------------------------------------------------|
| Informative   | Concise description of the image's content or purpose            |
| Functional    | Describes the action (e.g., `alt="Search"` for a search icon button) |
| Decorative    | `alt=""` — empty alt, screen readers skip it                     |
| Complex       | Brief alt + longer description via `aria-describedby` or `<figcaption>` |

### Video and Audio

- **Captions** (`<track kind="captions">`) — for deaf/hard of hearing users. Include speaker identification and sound effects.
- **Subtitles** — translation of dialogue only.
- **Audio descriptions** — narration of visual content for blind users.
- **Transcripts** — full text version of audio/video content.

---

## Forms

### Labeling

Every input **must** have an accessible name. In order of preference:

1. `<label for="id">` — best (visible and associated).
2. `aria-labelledby` — points to existing visible text.
3. `aria-label` — invisible label (use when no visible text is possible).
4. `title` — least preferred (not consistently exposed).

### Error Handling

```html
<label for="email">Email</label>
<input id="email" type="email"
       aria-invalid="true"
       aria-describedby="email-error"
       aria-errormessage="email-error" />
<span id="email-error" role="alert">
    Please enter a valid email address.
</span>
```

- Associate errors with inputs via `aria-describedby` or `aria-errormessage`.
- Use `aria-invalid="true"` to indicate validation failure.
- Don't rely solely on color to indicate errors.
- Announce errors with `role="alert"` or `aria-live`.

### Grouping

- Use `<fieldset>` + `<legend>` for radio groups, checkbox groups, and related fields.
- Use `role="group"` with `aria-labelledby` for custom groupings.

---

## Responsive and Mobile Accessibility

- **Touch targets** — minimum 44×44 CSS pixels (WCAG 2.5.8 Level AA in WCAG 2.2).
- **Zoom** — page must be usable at 200% zoom without loss of content or functionality.
- **Orientation** — don't lock to portrait or landscape unless essential.
- **Motion** — respect `prefers-reduced-motion` for animations.

```css
@media (prefers-reduced-motion: reduce) {
    *, *::before, *::after {
        animation-duration: 0.01ms !important;
        transition-duration: 0.01ms !important;
    }
}
```

---

## Testing

### Automated Tools

| Tool                  | Type                | What it catches                    |
|-----------------------|---------------------|------------------------------------|
| axe / axe-core        | Library / extension | ~30-40% of WCAG issues            |
| Lighthouse            | Chrome DevTools     | Accessibility audit score          |
| eslint-plugin-jsx-a11y| Linter              | React-specific a11y issues in JSX  |
| pa11y                 | CLI / CI            | Automated page-level testing       |

Automated tools catch **at most 30–40%** of accessibility issues. Manual testing is essential.

### Manual Testing Checklist

1. **Keyboard only** — navigate the entire page with Tab, Shift+Tab, Enter, Space, Escape, Arrow keys.
2. **Screen reader** — test critical flows with NVDA/VoiceOver.
3. **Zoom** — 200% browser zoom, check for content overlap or hidden content.
4. **Color** — simulate color blindness, verify contrast ratios.
5. **Motion** — enable `prefers-reduced-motion` and verify animations are suppressed.
6. **Heading structure** — verify headings are sequential and make sense out of context.

---

## Common Interview Questions

### What is the difference between `aria-label`, `aria-labelledby`, and `aria-describedby`?

- **`aria-label`** — provides the accessible name directly as a string. Used when no visible text exists.
- **`aria-labelledby`** — points to the ID(s) of elements whose text content forms the accessible name. Overrides `aria-label` and native labels.
- **`aria-describedby`** — adds supplementary description (read after the name). Does not replace the name.

### What is the difference between `aria-hidden="true"` and `display: none`?

- `aria-hidden="true"` — removes from the accessibility tree but **still visible** on screen. Use for decorative icons or duplicated content.
- `display: none` — removes from both the visual display **and** the accessibility tree.
- `visibility: hidden` — invisible but still takes up space, removed from accessibility tree.

### How do you make a custom dropdown accessible?

1. Use `role="combobox"` on the input, `role="listbox"` on the dropdown, `role="option"` on items.
2. Manage `aria-expanded`, `aria-activedescendant`, `aria-selected`.
3. Handle keyboard: Arrow keys navigate options, Enter selects, Escape closes.
4. Focus trap within the dropdown when open.
5. Announce selection changes with `aria-live` or by updating `aria-activedescendant`.

Or use `<select>` / `<datalist>` when possible — native elements are accessible by default.

### What are skip links and why are they important?

Skip links let keyboard users bypass repetitive content (navigation) and jump directly to the main content. Without
them, keyboard users must Tab through every nav link on every page load.

### How do you handle focus in a Single Page Application?

- After route changes, move focus to the new page's main heading or a dedicated `<h1>`.
- Announce the page change via `aria-live` or by updating `document.title`.
- Don't trap focus in a component that is no longer visible.
- Libraries like `react-router` don't manage focus by default — you must implement it.
