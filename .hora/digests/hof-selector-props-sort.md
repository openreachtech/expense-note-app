# hof-selector-props-sort
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-selector-props-sort/ -->

**Read the source above whenever this leaves a question open.**

The convention for sorting the properties declared inside **one CSS selector**. Properties are
**categorized by 'what the property applies to' and sorted from the outer toward the inner**. The
convention is called **Outer-to-Inner Order**. This skill is the authority for property order inside a
selector (`hof-css-coding-styles` covers only line breaks between and around selectors).

## Categories (8 blocks) — the full ordered list

Order runs outer → inner along the target. The own element and the child elements each split into
"layout (first)" and "design (later)", giving eight blocks in total.

| # | Category (skill's own name) | Properties it holds |
|---|---|---|
| 1 | **Creating content** | `content`, `quotes` — in pseudo-elements, the content-generating properties go first |
| 2 | **Display & positioning** | `clear`, `float`, `position` (+ `inset-block-start`, `inset-block-end`, `inset-inline-start`, `inset-inline-end`), `z-index`, flex **item-side** group (`flex` / `order` / `align-self` / `justify-self`), grid **item-side** group (`grid-row` / `grid-column` / `grid-area`) |
| 3 | **Layout to Outer** | `margin-block`, `margin-inline`, `outline`, `border` (border group) — ordered from the outside in |
| 4 | **Own Element (sizing)** | `block-size` (+ `min-block-size`, `max-block-size`), `inline-size` (+ `min-inline-size`, `max-inline-size`), `resize` — order **block → inline** |
| 5 | **Child Elements concerning Layout** | `display` (flex & grid), flex **container-side** group (`flex-direction` / `flex-wrap` / `justify-content` / `align-items` / `align-content`), grid **container-side** group (`grid-template-*` / `grid-auto-*` / `justify-items` / `place-items`), `gap`, `row-gap`, `column-gap`, `overflow`, `overflow-block`, `overflow-inline`, `padding-block`, `padding-inline`, `vertical-align` |
| 6 | **Design of Own Element** | `backface-visibility`, `background` (background group), `box-shadow`, `filter`, `perspective` (perspective group), `transform` (transform group), `opacity`, `visibility` |
| 7 | **Child Elements concerning Design** | `color`, `font` (font group), `text` (text group), `letter-spacing`, `line-height`, `white-space`, `word-break`, `word-spacing`, `word-wrap`, `writing-mode` |
| 8 | **Animations** | `animation` (animation group), `transition` (transition group) |

Note the split of flex/grid across two categories: **item side** (how this element behaves inside its
parent) is category 2; **layout / container side** (how this element arranges its children) is
category 5.

## Within a property group, order alphabetically

**Scope: a "property group" = a family of longhand properties sharing a prefix** (`border-*`,
`font-*`, …). When listing several such longhands, order them alphabetically.

```css
border
border-collapse
border-image
border-radius
border-spacing
```

```css
font-family
font-size
font-weight
```

**This is NOT an alphabetical sort across a whole category block.** The category lists above carry
their own fixed order and several are not alphabetical (category 6 runs `… perspective, transform,
opacity, visibility`; category 4 is explicitly **block → inline**). Alphabetical ordering applies
only *inside* one prefix family; between families, the category's stated order wins.

**Vendor-prefixed and custom properties (`--*`): the skill says nothing.** No rule to follow here.
full text: `.claude/skills/hof-selector-props-sort/SKILL.md#within-a-property-group-order-alphabetically`

## Blank lines between blocks

**Separate the blocks with a blank line.**

The skill's sample (below) additionally inserts a blank line **inside** category 3, between
`margin-block` and `border` — so a blank line between sub-groups of one category is at least
permitted by the sample. The skill states no rule for that case.

```css
.unit-sample {
  z-index: calc(var(--value-z-index-layer-staying) + 0);

  margin-block: 1rem;

  border: var(--size-thinnest); /* ⚠ shorthand — see conflicts, do not copy */

  block-size: 10rem;
  inline-size: 100%;
  min-inline-size: 20rem;

  padding-block: 0.25rem;
  padding-inline: 0.5rem;

  background-color: var(--color-primary);

  color: var(--color-text-primary);
  font-family: 'Hiragino Sans';
  font-size: 1.5rem;
  font-weight: bold;
  line-height: var(--value-golden-ratio);

  transition: opacity 0.3s;
}
```

## Not covered by this skill

Two questions the implementer will hit that the source does **not** answer. The skill is 167 lines and
contains no text on either — reading it will not help; decide from `D:\ORT\rules\05-frontend.md`
and existing files in the tree.

| Question | What the skill says |
|---|---|
| **A property that fits no category** — where does it go? | Nothing. No catch-all block, no fallback rule. |
| **Nested blocks — where a nested `@media` sits relative to the properties** | Nothing. The skill never mentions `@media`, nesting, or nested selectors; its sample has a flat block only. (`05-frontend.md` §6-11 mandates the media query go *inside* the target selector, but neither document says whether it precedes, follows, or interleaves with the declarations.) |

---

## Conflicts and agreements with the always-on rules

`D:\ORT\rules\05-frontend.md` is always-on and **WINS** on every point below. The conflicts are named,
not resolved.

### Agreements — settled, do not re-check

- **Outer → inner direction.** §6-5 ("Outer: placement/margin → Main: the element's own core
  properties → Inner: internal padding") and this skill's Outer-to-Inner Order run the same way.
- **Blank line where the meaning changes.** §6-5 mandates it; the skill's "separate the blocks with a
  blank line" is the same instruction, with the skill's 8 categories naming exactly where the meaning
  changes.
- **The alphabetical-within-group clause is purely additive and compatible.** §6-5 says nothing about
  ordering *within* one of its three parts, and neither `hof-css-coding-styles` nor §6-5 states any
  alphabetical rule. It adds a finer sort inside a prefix family without moving anything across §6-5's
  three parts. Apply it.
- **Units / variables.** The skill's sample uses `rem` and `var(--…)` throughout, consistent with §6-2
  and §6-6.

### Conflict 1 — where `background` sits relative to `padding`

- **§6-5 (wins):** three parts — Outer (`margin`) → **Main, the element's own core properties
  (`width` / `height` / `background`)** → Inner (`padding`). Its worked example is
  `.unit-articles-container`: `margin` → blank → `max-width` → blank → `padding`. So `background`
  belongs with sizing, **before** `padding`.
- **This skill:** `background` is category **6** (Design of Own Element), which comes **after**
  `padding-block` / `padding-inline` in category **5** (Child Elements concerning Layout). Its sample
  orders `padding-block`, `padding-inline` → blank → `background-color`.

Both orders are shown above rather than averaged. The rule wins.

### Conflict 2 — shorthand properties in the skill's examples

§6-4 bans **all** CSS shorthand properties; write individual longhands.

- The skill's sample line `border: var(--size-thinnest);` is a **shorthand and is not copyable**.
- The category lists name `border`, `background`, `outline`, `font`, `text`, `animation`, `transition`,
  `perspective`, `transform` as "groups" and also list the bare shorthand (e.g. `border`,
  `background`) as a member. Read those as the *family name* marking the slot in the order — write the
  longhands (`border-block-width`, `background-color`, `transition-property`, …), never the shorthand.
- The sample's `transition: opacity 0.3s;` is likewise shorthand.

### Conflict 3 — logical shorthands (`margin-block`, `padding-inline`, …)

The skill's canonical names are the **logical** two-value shorthands: `margin-block`, `margin-inline`,
`padding-block`, `padding-inline`, `block-size`, `inline-size`. Each of `margin-block` /
`margin-inline` / `padding-block` / `padding-inline` sets two sides at once, which §6-4's ban on
shorthand and its worked example (`margin-top` / `margin-right` written separately) point away from.
Named, not resolved; §6-4 wins. The **ordering slot** the skill assigns them is unaffected — whichever
spelling you write, margins sit in category 3 and paddings in category 5.
