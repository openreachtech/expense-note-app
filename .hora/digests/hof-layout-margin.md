# hof-layout-margin
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-layout-margin/ -->

**Read the source above whenever this leaves a question open.**

## The core rule

For elements laid out with Flex / Grid, **spacing is the responsibility of the container
(the layout owner), not of the item's own `margin`**.

- A **layout item** = "an item laid out with Flex / Grid (a direct child of the layout)".
- **Do not put `margin` on a layout item.**

```css
/* NG: burning spacing into the item itself */
.item {
  margin-block-end: 1rem;
}
```

## Which form applies

| Situation | Form |
| :-- | :-- |
| Even spacing, Flex / Grid | container's `gap` |
| Per-item spacing `gap` cannot express | `.layout.xxx > .item` from the parent |
| Vertically stacked children in normal flow (block flow) — `gap` unavailable | owl selector `.layout.xxx > * + *`, following child owns `margin-block-start` |

## Even spacing — `gap`

Make even spacing in Flex / Grid with the container's `gap`. Because `gap` is spacing the
container owns, **no extra spacing appears on the edge items** and no adjacent-margin
collapsing occurs.

```css
/* OK: the container owns spacing via gap */
.layout {
  display: flex;
  gap: 1rem;
}
```

## Uneven / exceptional spacing — own it from the parent

When per-item spacing that `gap` cannot express is needed, **still do not write it on the
item**. Specify the child's style from the layout owner, using the child combinator.

```css
/* OK: apply exceptional per-item spacing to the child from the parent's specific variant */
.layout.toolbar > .item {
  margin-inline-start: auto;
}
```

## Stacked children in flow — the owl selector

`gap` belongs to Flex / Grid, so it is unavailable in normal flow. From the layout owner,
use `> * + *` so the **following** child owns the spacing as `margin-block-start`.

```css
/* OK: from the layout owner, the following child owns the spacing via margin-block-start (the owl selector) */
.layout.xxx > * + * {
  margin-block-start: 2rem;
}
```

- **First and last child:** `* + *` selects only children that have a preceding sibling.
  The **first child has no preceding sibling, so it gets no margin.** Spacing arises *only
  between* one child and the next — **no stray spacing at the layout's start or end.**
- The skill's literal wording scopes this to similar `<section>`s stacking vertically,
  while also stating the owl selector plays in flow the role `gap` plays in Flex / Grid.
  full text: `.claude/skills/hof-layout-margin/SKILL.md#spacing-between-stacked-sections`

## Where a `margin` IS still permitted

Only in a declaration **written from the layout owner** — `.layout.xxx > .item` or
`.layout.xxx > * + *`. Never in a rule whose selector is the item itself. The owner of the
spacing always stays on the layout side.

## Not covered by this skill

- **Spacing between a label and its control.** The skill says nothing about it. See the
  `hof-cp-control-block` digest/skill (the molecule that frames a field with its label).
- Margin on an element that is neither a Flex / Grid item nor a child of a spacing-owning
  layout. Out of the skill's stated scope.

## Conflicts — named, not resolved. `D:\ORT\rules\` wins.

1. **`gap` is a shorthand.** `gap: 1rem` is shorthand for `row-gap` / `column-gap`, and
   `rules/05-frontend.md` 6-4 bans CSS shorthands outright ("ショートハンドは原則すべて禁止",
   with `flex: …` shown as NG and split into `flex-grow` / `flex-shrink` / `flex-basis`).
   The skill's entire even-spacing rule is expressed as `gap`. **Rule wins** — but note the
   skill's *responsibility* rule (container owns spacing) is untouched by how the
   declaration is spelled.
2. **Placeholder class names `.layout` / `.item`.** `rules/naming.md` lists `item` among
   the forbidden words; `rules/05-frontend.md` 6-3 forbids class names that describe the
   CSS mechanism (Tailwind-like naming) and 6-7 requires a layout block's root element to
   carry the `unit-` prefix with a semantic, single-word remainder. The skill's `.layout`,
   `.item`, `.toolbar`, `.xxx` are illustrative selectors, not sanctioned names.
   **Rule wins.**
3. **Rules-internal tension (not the skill's).** `rules/05-frontend.md` 6-5's
   property-order example writes `margin: 0 auto;` in the "Outer" slot, while 6-4 of the
   same file bans all shorthands. The no-shorthand rule governs the actual declaration, so
   it must be written as individual `margin-*` properties. Named here because this skill is
   entirely about `margin` declarations.

## Agreements with `D:\ORT\rules\`

- **Child combinator only.** The skill uses `>` throughout and never a descendant
  combinator — exactly 6-8.
- **`rem` units** in every example — 6-6.
- **Individual logical properties** (`margin-block-start`, `margin-block-end`,
  `margin-inline-start`), never the `margin` shorthand — consistent with 6-4.
- **Property order.** Margin is the outermost property; 6-5 orders outer → main → inner
  with a blank line where meaning changes. The skill's snippets are too small to show
  ordering, so 6-5 governs any rule you write with more than one declaration.
