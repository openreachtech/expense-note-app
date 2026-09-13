# hof-css
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-css/ -->

**Read the source above whenever this leaves a question open.**

Every component owns a `<style scoped>` block whose selectors descend from a single `.unit-*` root. Colors, sizes, and typography come from CSS custom properties (design tokens), never hard-coded values.

## Unit selectors

- A unit is a root styling boundary and MUST use the `.unit-` prefix: `.unit-header {}` `.unit-form {}` `.unit-modal {}` `.unit-card {}`.
- Bad: `.header {}` (no unit root), `.page-header {}` (context-duplicating prefix instead of `unit-`), `.app-main {}`.
- **Every selector chain starts from a unit.** `.unit-card > .title {}`, never a bare `.title {}`. "Never create global descendant selectors without a unit root. Because Vue `scoped` styles already isolate the component, the unit root documents intent rather than providing isolation."
- **Short descendant names** — the unit supplies the context, so do not repeat it. Prefer: `.heading .body .content .text .title .label .value .icon .image .input .list .item .row .col .group .actions .header .footer`. Bad: `.unit-modal > .modal-header {}`.
- **Child combinator `>`** — `.unit-card > .body > .content {}`, not `.unit-card .body .content {}`.
- **Chain depth: aim 2–4 levels, never exceed ~5.** When a section grows deep, promote it into its own unit instead of extending the chain. "Excessive nesting signals poor component decomposition or a missing unit boundary."

```html
<div class="unit-header">
  <div class="heading">
    <span class="text">Title</span>
  </div>
</div>
```

```css
.unit-header {}
.unit-header > .heading {}
.unit-header > .heading > .text {}
```

## Design tokens

- App tokens are declared in `assets/css/variables.css`. **"When a component needs a new shared value, add a token to `assets/css/variables.css` rather than inlining it."**
- **"Never hard-code a color, font size, or z-index in a component."** Reference a token. Bad: `background-color: #fff`, `font-size: 22px`.
- **Palette vs semantic:** palette tokens are raw hues (`--palette-brand-blue-500`, `--palette-gray-100`). **"Do not reference palette tokens directly in components; use the semantic tokens below, which alias the palette."**
- Token families the library supplies, and which this app therefore only overrides or extends:

| Family | Tokens |
| --- | --- |
| Palette | `--palette-*` raw hues |
| Semantic color | `--color-primary`, `--color-background`, `--color-background-panel`, `--color-surface-*` (`success`/`error`/`ongoing`/`warning`, each with `-subtle`/`-lighter`/`-darker`), `--color-border-*`, `--color-text-*` (`title`/`subtitle`/`caption`/`light`/`disabled`/`input`…) |
| Sizes | `--size-thinnest`, `--size-radius-rounded`, `--size-header-height`, `--size-nav-width`, `--size-screen-height` |
| Typography | `--font-family-body`, `--font-size-*` (`title-large` → `tiny`) paired with matching `--size-line-height-*` |
| Gradients | `--gradient-membership-silver\|gold\|platinum\|diamond` |
| Transitions | `--transition-timing-ease-in\|out\|in-out` |

full text: references/design-tokens.md

## Global stylesheet layering

"Global CSS load order is fixed in `nuxt.config.js` `css: [...]`." **Layering here means load order in that array — the skill declares no `@layer`, and this repository has none.**

- **"Furo layers load first; app overrides come after the corresponding Furo file."**
- **"Global, cross-page styles → `assets/css/main.css`. Component-scoped styles stay in the component's `<style scoped>`."**
- "New global CSS files must be registered in `nuxt.config.js` with a numeric prefix that places them correctly in the cascade."

The order in this repository, which already satisfies "library first, app after":

```
@openreachtech/furo-vue/lib/assets/css/furo.css   ← library tokens: --palette-*, --color-*, z-index, dimensions
~/assets/css/variables.css                        ← app design tokens (currently an empty :root {})
~/assets/css/main.css                             ← app global styles
```

**File-map divergence (not a rule conflict):** the skill's reference names a `@openreachtech/furo-nuxt` chain of numerically-prefixed files (`0000.furo.css`, `0010.variables-palette-color-scale.css`, `0020.variables-z-index.css`, `0100.reset.css`, `0200.base.css`, `0300.gimmick.css`), an app `assets/css/0110.reset.css`, and an `assets/css/variables-component-default.css` for component defaults. **None of those exist here** — this repository uses furo-vue's single public entry `furo.css` and has only `variables.css` + `main.css`. Apply the skill's *ordering rule*, not its file list. full text: references/global-stylesheets.md

## Transitions

Add `will-change` for a transformed property **only** when a `transform` transition causes a visible 1px sub-pixel jump. "Overusing it wastes memory by promoting unnecessary layers."

## Where the skill agrees with `D:\ORT\rules\05-frontend.md` — settled, do not re-check

| Point | Both say |
| --- | --- |
| `unit-` prefix | A layout block's root carries `unit-`; a bare `.header` is wrong |
| Child combinator | Use `>`; do not rely on descendant matching |
| Semantic class names | Names describe structure/meaning, never a CSS property (no Tailwind-like names) |
| No colour literals | Every colour is a custom property; never a hex in a component |
| Two-tier tokens | Raw palette (`--palette-*`) aliased by use-named semantic tokens (`--color-*`); components read only the semantic tier |
| Token home | Shared values are declared in `assets/css/variables.css`, not inlined |

## Conflicts with `D:\ORT\rules\05-frontend.md` — the rule WINS, unresolved

1. **Shorthand properties.** The skill's own examples use shorthands: `border: var(--size-thinnest) solid var(--color-border)` (references/design-tokens.md) and `transition: transform 0.2s var(--transition-timing-ease-out)` (references/transitions.md). Rule 6-4 bans shorthand outright. **Rule wins** — write `border-width` / `border-style` / `border-color` and `transition-property` / `transition-duration` / `transition-timing-function` separately.
2. **`line-height`.** The skill pairs every `--font-size-*` with a matching `--size-line-height-*` and its example sets `line-height: var(--size-line-height-large)`. Rule 6-12 forbids `line-height` unless asked (`p` gets the golden ratio, everything else defaults to 1). **Rule wins.**
3. **`px`-valued tokens.** The skill documents `--size-thinnest` as `1px`. Rule 6-6 mandates `rem` and never `px`, with only intuitive decimal fractions. **Rule wins for tokens this app declares**; a library token consumed by name is not something the app writes.
4. **Descendant-combinator escape hatch.** The skill permits a descendant selector "only when a direct child is impossible". Rule 6-8 permits none. **Rule wins** — always `>`.
5. **Abbreviated descendant name `.col`.** It appears in the skill's preferred-names list. Rule 6-14 (and `naming.md`) ban abbreviations. **Rule wins** — write `.column`. The same list's `.list` and `.item` are forbidden words under `naming.md`; treat that as unresolved rather than settled.
6. **Rule requirements the skill is silent on, which still bind:** properties ordered outer → inner with blank lines where meaning changes (6-5), media queries written inside the selector they modify (6-11), `unit-` names kept to one word after the prefix with finer structure as split classes such as `.unit-table.users` (6-7), 2-space indent (6-1), semantic HTML tags over layout `div`s (6-10), attributes chopped down at two or more (6-9).
