# hof-css-prohibits
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-css-prohibits/ -->

**Read the source above whenever this leaves a question open.**

Collects CSS notations that are prohibited (anti-patterns). Five prohibitions, all of them
**new** relative to the always-on `D:\ORT\rules\05-frontend.md` — that rule forbids shorthands,
`px`, 16px-derived decimals, Tailwind-like class names, colour literals, the descendant
combinator, layout `line-height` and div soup, and says nothing about any of the five below.

| # | Prohibition | vs `05-frontend.md` |
| :-- | :-- | :-- |
| 1 | physical-direction properties and values | **new** |
| 2 | CSS nesting | **new** (its media-query exception matches rule 6-11) |
| 3 | one-line declaration blocks | **new** (aligned with charter (7)) |
| 4 | `!important` | **new** |
| 5 | bare tag selectors outside `reset` / `base` | **new** — stricter than the rule's `>`-combinator requirement; free to obey both |

Nothing in this skill is *looser* than the rule. Where its own examples drift, see the final section.

## 1. Prohibit physical-direction properties and values

**Forbidden without exception:** physical direction — `top` / `right` / `bottom` / `left`, physical
axes like `width` / `height`, and physical values like `float: left`. **Why:** in right-to-left (RTL)
languages left / right do not work and break the layout.

**Write instead:** only the **logical properties and values**, which follow the writing direction.
To set a single side use `*-inline-start` / `*-inline-end` / `*-block-start` / `*-block-end` —
start / end are the only way to set one side, so they are allowed.

**The one exception:** `border-image-outset` takes four values but has no logical equivalent and
cannot be replaced, so it alone may stay in physical form. `outline-*` is **out of scope** (uniform,
no per-side declarations, no logical variant — nothing to prohibit).

Mapping is shown for LTR; in RTL the inline direction's left/right is reversed.

### A. Properties with a top / right / bottom / left set

| Physical | Logical |
| --- | --- |
| `margin-top` | `margin-block-start` |
| `margin-bottom` | `margin-block-end` |
| `margin-left` | `margin-inline-start` |
| `margin-right` | `margin-inline-end` |
| `margin` (top+bottom) | `margin-block` |
| `margin` (left+right) | `margin-inline` |
| `padding-top` | `padding-block-start` |
| `padding-bottom` | `padding-block-end` |
| `padding-left` | `padding-inline-start` |
| `padding-right` | `padding-inline-end` |
| `padding` (top+bottom) | `padding-block` |
| `padding` (left+right) | `padding-inline` |
| `border-top` | `border-block-start` |
| `border-bottom` | `border-block-end` |
| `border-left` | `border-inline-start` |
| `border-right` | `border-inline-end` |
| `border-top-left-radius` | `border-start-start-radius` |
| `border-top-right-radius` | `border-start-end-radius` |
| `border-bottom-right-radius` | `border-end-end-radius` |
| `border-bottom-left-radius` | `border-end-start-radius` |
| `top` | `inset-block-start` |
| `bottom` | `inset-block-end` |
| `left` | `inset-inline-start` |
| `right` | `inset-inline-end` |
| (top+bottom) | `inset-block` |
| (left+right) | `inset-inline` |
| (all four) | `inset` |
| `scroll-margin-top` | `scroll-margin-block-start` |
| `scroll-margin-bottom` | `scroll-margin-block-end` |
| `scroll-margin-left` | `scroll-margin-inline-start` |
| `scroll-margin-right` | `scroll-margin-inline-end` |

- border `-width` / `-style` / `-color` follow the same shape (`border-left-width` →
  `border-inline-start-width`). Both edges: `border-block` / `border-inline` (and `border-block-width`, etc.).
- scroll both edges: `scroll-margin-block` / `scroll-margin-inline`. `scroll-padding-*` follows exactly the same shape.

### B. Physical dimensions

| Physical | Logical |
| --- | --- |
| `width` | `inline-size` |
| `height` | `block-size` |
| `min-width` | `min-inline-size` |
| `min-height` | `min-block-size` |
| `max-width` | `max-inline-size` |
| `max-height` | `max-block-size` |
| `overflow-x` | `overflow-inline` |
| `overflow-y` | `overflow-block` |

### C. Physical-direction values

| Property | Physical value → logical value |
| --- | --- |
| `float` / `clear` | `left` → `inline-start`, `right` → `inline-end` |
| `text-align` | `left` → `start`, `right` → `end` |
| `caption-side` | `top` → `block-start`, `bottom` → `block-end`, `left` → `inline-start`, `right` → `inline-end` |

## 2. Prohibit CSS nesting

**Forbidden:** nesting selectors — neither a selector inside a rule nor an `&` combination.
**Why:** diffs are unreadable in GitHub pull requests; deep nesting forces scrolling to confirm an
addition sits at the correct level; the editor assistance that supplies nesting context does not
work on GitHub.

**Write instead:** keep selectors flat.

```css
/* NG: nested selectors */
.card {
  color: #cc3300;

  .card-title {
    font-weight: bold;
  }

  &:hover {
    opacity: 0.8;
  }
}

/* OK: keep selectors flat */
.card {
  color: #cc3300;
}

.card-title {
  font-weight: bold;
}

.card:hover {
  opacity: 0.8;
}
```

**The one exception:** a media-query override **for the same selector** may be written as `@media { }`
inside that selector. (This is what rule 6-11 already requires.)

```css
/* OK (the one exception): only a media-query override for the same selector goes inside */
.card {
  padding-inline: 0.75rem;

  @media (min-width: 48rem) {
    padding-inline: 1rem;
  }
}
```

## 3. Prohibit one-line declaration blocks

**Forbidden:** writing a selector's `{ }` (declaration block) on one line. **No exceptions, even for a
single declaration** — this keeps things consistent.

**Write instead:** opening `{` on the same line as the selector, one declaration per line, closing `}`
on its own line — always chopped down.

```css
/* NG: { } written on one line */
.card { color: #cc3300; }

/* OK: always chopped down */
.card {
  color: #cc3300;
}
```

## 4. Prohibit `!important`

**Forbidden without exception, whatever the reason.** **Why:** it hoists a declaration outside the
specificity system and breaks control of the cascade itself, leading to an "importance war" of
`!important` overriding `!important`; its effect is non-local, silently nullifying legitimate styles
elsewhere with a cause that is hard to trace.

**Write instead:** do not raise specificity either — resolve the override through the order of
**cascade layers**: `reset` → `base` → `furo` → `app`, **later layers win**
(→ the cascade layers convention).

```css
/* NG: striking with !important */
.button {
  color: #fff !important;
}

/* OK: let layer order decide (app beats furo — no !important) */
@layer furo {
  .button {
    color: #333;
  }
}

@layer app {
  .button {
    color: #fff;
  }
}
```

## 5. Prohibit bare tag selectors outside the `reset` / `base` layers

**Allowed only** in `@layer reset` and `@layer base`, the two layers that define the look of the
native HTML elements themselves (`reset` = reset, `base` = base design system).
**Forbidden** in the `furo` / `app` layers. **Why:** a bare tag selector hits every element of that
kind across the whole document — correct for "the default look of the raw element", but in
component layers it becomes a source of styles leaking onto unrelated elements.

**Write instead:** in `furo` / `app`, always qualify the tag with a class — the `tagname.furo` form
(per the Furo convention).

```css
/* OK: reset / base target the native elements themselves */
@layer base {
  a {
    color: var(--color-primary);
  }
}

/* NG: a bare tag selector in furo / app */
@layer furo {
  a {
    color: var(--color-primary);
  }
}

/* OK: in furo / app, qualify with a class (tagname.furo) */
@layer furo {
  a.furo {
    color: var(--color-primary);
  }
}
```

## Conflicts with `D:\ORT\rules\05-frontend.md` — the rule WINS, unresolved

Named, not resolved. The rule is always-on and takes precedence in every row below.

| # | Conflict | The rule's side (wins) |
| :-- | :-- | :-- |
| C1 | This skill's sanctioned logical forms `padding-block` / `padding-inline` / `margin-block` / `margin-inline` / `border-block` / `border-inline` / `inset-block` / `inset-inline` / `inset` / `scroll-margin-block` / `scroll-margin-inline` are **two-or-more-value shorthands**, which rule 6-4 forbids outright. | Rule 6-4 wins: write the single-side logical longhands (`padding-block-start` + `padding-block-end`), never the `-block` / `-inline` pair form. |
| C2 | The skill's OK example writes `border-block-end: var(--hairline-width) solid #ccc` — a shorthand packing width + style + colour. | Rule 6-4 wins: split into `border-block-end-width` / `-style` / `-color`. |
| C3 | Skill examples use colour literals in property values (`#cc3300`, `#fff`, `#333`, `#ccc`). | Rule 6-2 wins: no hex/rgb in a property value; use a purpose-named `var(--color-xxx)` defined in `variables.css`. |
| C4 | The nesting exception's example writes `@media (min-width: 48rem)`; rule 6-11's example writes the range form `@media (30rem < width)`. Also `min-width` is itself a Part-B physical dimension, though here it is a media feature, not a property. | Rule 6-11's range form is the house example; prefer `@media (48rem < width)`. Low-grade — the rule states no explicit mandate. |

**Stricter, and free to obey:** prohibition 5 (class-qualified tag selectors in `furo` / `app`) sits on
top of rule 6-8's child-combinator requirement without contradicting it — satisfy both.
The skill's own NG/OK examples are not style exemplars; take the mapping and the rules, not the formatting.
