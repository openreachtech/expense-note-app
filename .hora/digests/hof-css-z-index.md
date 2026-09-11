# hof-css-z-index
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-css-z-index/ -->

**Read the source above whenever this leaves a question open.**

## Does the sign-in screen need this at all?

Probably not. The skill governs **how to write a `z-index` when one is needed**; it never
requires writing one. A heading, two fields, a button and an error message do not overlap,
are not a Header / Footer / Nav / Sidebar, and are not a Modal — they are ordinary content,
which is the `content` layer's base, i.e. the default stacking. **Writing no `z-index` is the
correct outcome for a flat form.** Apply the rules below only if something must actually be
raised above a sibling.

## The three layers

Base values are defined as variables in `:root` (max-value: 2147483647):

| Layer | Variable | Base value | For |
| --- | --- | --- | --- |
| ground | `--value-z-index-layer-content` | `0000000` | Ordinary content |
| cloud | `--value-z-index-layer-staying` | `1000000` | Elements that stay, e.g. Header / Footer / Nav / Sidebar |
| space | `--value-z-index-layer-overlay` | `2000000` | Overlays such as Modals |

**In this repository the scale already exists — do not re-declare it.**
`@openreachtech/furo-vue@1.3.2` ships `lib/assets/css/furo/0020.variables-z-index.css`,
imported by `furo.css`, declaring those three **plus a fourth the skill does not document**:

```css
--value-z-index-layer-popover: 3000000;
```

So all four `--value-z-index-layer-*` are globally readable already. `assets/css/variables.css`
is an empty `:root {}`; adding these there would fork a parallel scale. Extend the shipped one.

## Always `calc()` — even when the right operand is 0

```css
.modal {
  z-index: calc(var(--value-z-index-layer-overlay) + 0); /* ✅ */
}
```

```css
z-index: calc(var(--value-z-index-layer-staying) + 0); /* ✅ */
z-index: var(--value-z-index-layer-staying);           /* ❌ */
```

"Chapter and verse": the **chapter** is the layer variable, the **verse** is the right operand
(ordering within that layer).

**Bare numeric `z-index`** (`z-index: 10`) — the skill states the rule as "Always write z-index
in the form `calc(layer-variable + number)`", so no. Its one explicit ❌ is the bare `var()`
without `calc()`; a bare integer is excluded by "always" rather than shown as an example.

## Within the same layer, stack in steps of 1000

Increase the verse in thousands — `+ 0`, `+ 1000`, `+ 2000`. Read even `+ 0` as "a thousands
place still left open", not as the ones place.

```css
.header {
  z-index: calc(var(--value-z-index-layer-staying) + 0);
}

.sidebar {
  z-index: calc(var(--value-z-index-layer-staying) + 1000);
}

.more-bar {
  z-index: calc(var(--value-z-index-layer-staying) + 2000);
}
```

**Something that must sit between two existing levels:** that is exactly what the thousands
step buys — insert between `+ 1000` and `+ 2000` (e.g. `+ 1500`). The skill answers this only
*within* one layer. It gives **no** rule for something needing to sit between two **layer
bases**; the million-wide gap means the verse absorbs it, but that inference is mine, not the
skill's. full text: .claude/skills/hof-css-z-index/SKILL.md#within-the-same-layer-stack-in-steps-of-1000

Note one inconsistency in the source: the "Verse" illustration lists `+ 0 / + 1 / + 2`, while
the operative section mandates steps of 1000. Follow the thousands.

## Conflicts with the always-on rules

**None.** `D:\ORT\rules\05-frontend.md` wins by standing order, and it says nothing about
`z-index` — this skill is purely additive to it. The `--value-*` prefix also matches that
rule's own `Values` block convention (`--value-golden-ratio`).

Adjacent, not a conflict: 05-frontend 6-2 collects variables in `variables.css`, whereas this
scale arrives from the furo-vue package. Consume it from there; do not copy it into
`assets/css/variables.css`.
