# hof-css-layers
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-css-layers/ -->

**Read the source above whenever this leaves a question open.**

The source is one file, `SKILL.md`, 35 lines, with no `references/` or `scripts/`. It is short
enough that this digest carries essentially all of it verbatim; the added sections below the
rule are this repository's verified state, not skill text.

## The layer order

Company CSS is based on the following cascade layers (`@layer`), in this order.
**Later layers take precedence.**

| Layer | Role |
| --- | --- |
| `reset` | Reset styles for native HTML elements |
| `base` | Base design system for native HTML elements |
| `furo` | Furo component library (defined by the internal module set) |
| `app` | Application components extended from Furo |

- The `furo` layer is defined by the internal module set.
- The `app` layer is where an application that has `npm install`ed the modules defines or
  overrides styles.
- Declare the layer order **once at the top of the file (right after `@charset`)** to fix it.

## Required shape of the declaration

```css
@charset "UTF-8";

/**
 * Furo design for HTML tags
 *
 * all selectors are composed by the tag name followed by ".furo"
 */

@layer
  reset, /* Reset styles for native HTML elements */
  base,  /* Base design system for native HTML elements */
  furo,  /* Furo component library */
  app;   /* Application components extended from Furo */
```

## What the skill does NOT say

The skill states the order and each layer's role, and nothing else. In particular it does
**not** say:

- which file in an application should hold the order declaration, or where in the build it
  must be loaded;
- whether an application is obliged to adopt `@layer` at all, or may write unlayered CSS;
- what happens to unlayered CSS relative to these layers;
- anything about a single screen or feature introducing the system.

Those are decisions the skill leaves open. Treat the section below as this repository's facts,
and anything beyond it as unanswered — `full text: .claude/skills/hof-css-layers/SKILL.md`.

## Does this skill apply here? — verified state of this repository

Verified at digest time in `expense-note-frontend-staff`:

| Fact | Verified how |
| --- | --- |
| The application declares **no `@layer` anywhere**. | `assets/css/main.css` is 7 lines of iOS input sizing only; `assets/css/variables.css` is a commented, empty `:root {}`. No `@layer` in any app `.css` or `.vue`. |
| **`furo-vue@1.3.2` does participate.** It wraps its own styles in `@layer furo { … }`. | `lib/assets/css/furo/0050.furo-editor-content.css` and the `<style>` block of ~20 components (`FuroButton`, `FuroTextField`, `FuroCheckbox`, …) each open `@layer furo {`. |
| **`furo-vue` does NOT declare the order.** No `@layer a, b, c;` statement exists anywhere in the package. | Its entry `lib/assets/css/furo.css` is `@charset` plus six `@import`s of token files — no order statement. The order is therefore unowned in this repository. |
| furo's design tokens (`--palette-*`, `--color-*`, `--size-*`, `--font-*`) are **not** layered — they are plain `:root` declarations. | The five `00x0.variables-*.css` files declare `:root { … }` with no `@layer`. Custom properties are unaffected by layer precedence. |
| `furo.css` is loaded first in `nuxt.config.js`'s `css` array, then `variables.css`, then `main.css`. | `nuxt.config.js` lines 51–62. |

Consequences the implementer should reason from:

- **Unlayered CSS outranks every layered rule** (CSS cascade: unlayered styles win over all
  `@layer` styles regardless of order). Because furo's component styles sit in `@layer furo`
  and the application writes unlayered CSS, **app CSS already overrides furo component CSS
  today**, with no layer system in place. The effect the `app`-over-`furo` order exists to
  produce is already the status quo here.
- **Adopting the order declaration is a project-wide structural change, not a screen-local
  one.** The skill requires the declaration to come first; the first stylesheet loaded here is
  `furo.css` inside `node_modules`, which cannot be edited. Fixing the order therefore means
  adding a new app-owned stylesheet **ahead of `furo.css` in `nuxt.config.js`'s `css` array**
  — touching global build config and changing the precedence of every existing rule at once.
  The sibling repository `crm-kit-frontend` does exactly this with a `0000.crm-kit-layers.css`
  loaded before the reset and `furo.css`; **that is house practice in a different repository,
  and not evidence about this one.**
- A single screen that introduces `@layer app { … }` **without** a preceding order declaration
  would demote its own styles below the unlayered rules already in `main.css`, and would leave
  `reset` and `base` unordered. If `@layer` is adopted, the declaration comes first or not
  at all.
- A stale note in this project's tree doc once claimed a `reset, base, furo, app` declaration
  existed here. It does not; that claim was corrected at checkpoint 12.

**So: the skill assumes a layer system already exists** (it describes an order to follow and an
`app` layer to write into, and gives no adoption procedure). That assumption does not hold in
this repository. Introducing one is a project-wide decision to escalate, not a step inside a
single sign-in screen.

## Conflicts with always-on rules

- **`D:\ORT\rules\05-frontend.md` wins, and there is no conflict to resolve.** The always-on
  frontend rule says nothing about `@layer` — not the at-rule, not layer names, not ordering.
  This skill is **additive** to it, not contradictory.
- One adjacency, not a conflict: 05-frontend 6-13 requires extending a default component with a
  prefixed wrapper rather than modifying it. That sits consistently with the `app` layer being
  the place an application overrides `furo`.
- Every rule in 05-frontend that governs the CSS actually written — 2-space indent, no colour
  literals, semantic (non-Tailwind-like) class names, no shorthand properties, Outer → Inner
  property order, `rem` units, `unit-` prefixed layout roots, child combinator `>`, media
  queries nested inside the selector — applies unchanged whether or not any `@layer` exists.
