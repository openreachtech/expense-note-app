# hof-css-line-height
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-css-line-height/ -->

**Read the source above whenever this leaves a question open.**

`SKILL.md` is the whole skill — 37 lines, no `references/`, no `scripts/`. It states one rule (default = golden ratio) and one constraint (hold it unitless). It is already terse, so this digest is close to its size.

## The default value

**The default value of `line-height` is `--value-golden-ratio` (the golden ratio, 1.618).** Define it once in the base layer and let all text inherit it. Override only when an individual case needs a different value.

```css
:root {
  --value-golden-ratio: 1.618;
}

@layer base {
  :root {
    line-height: var(--value-golden-ratio);
  }
}
```

- Fix the document-wide default line height at this single point, give it to `:root` in the `base` layer, and let it inherit (→ `hof-css-layers`).
- `--value-golden-ratio` is a `--value-*` (unitless scalar) token (→ `hof-css-props-naming`). The same token can be reused for ratios other than line-height.

## Hold it unitless

- Hold `line-height` **unitless**. `--value-golden-ratio` is a unitless scalar (`--value-*`).
- Reason: a unitless `line-height` makes each element compute its line height as "its own `font-size` × 1.618". When a child's `font-size` changes, the line height scales correctly with it. Fixing it with `px` or `em` makes the value computed at the parent inherit as-is, failing to follow the child's `font-size` and breaking the layout.

**Consequence for overrides:** a numeric override is safe because it is unitless and re-resolves per element. An override carrying `px` or `em` is not — it freezes at the declaring element and inherits as a length.

## When to override

| Case | Value |
|---|---|
| Default, all text | `var(--value-golden-ratio)` (1.618) |
| An individual case needing a different value | a bare unitless number — the skill's only example is `line-height: 1.2` on `.heading` |

```css
/* Override only when an individual case needs a different value */
.heading {
  line-height: 1.2;
}
```

The skill enumerates **no** permitted-value list beyond that example, and gives no threshold for "needs a different value". `full text: .claude/skills/hof-css-line-height/SKILL.md`

## When line-height must NOT be set at all

**The skill does not answer this.** It has no "do not set" case — its position is that every element carries the inherited golden ratio, and the only alternative it names is an override. The prohibition the implementer is subject to comes from the always-on rule below, not from this skill.

## This repository's environment (verified, not from the skill)

| Fact | State |
|---|---|
| `--value-golden-ratio: 1.618` | **Already supplied.** `@openreachtech/furo-vue/lib/assets/css/furo/0035.variables-semantic-dimension.css:98`, in `:root` under `/* Constants */`. `furo.css` is loaded first in `nuxt.config` `css`. **This checkpoint does not need to declare it.** |
| `line-height: var(--value-golden-ratio)` applied to `:root` | **Not supplied by furo.** No stylesheet in `@openreachtech/furo-vue/lib/assets/css/` sets `line-height` on any selector. The skill's `@layer base` block is the application's to write — and rule 6-12 says not to. |
| `assets/css/variables.css` | An empty `:root {}` with a comment. Declares nothing. |
| `@layer` declaration | None anywhere in the app; `nuxt.config` states layers would order by first appearance. So the skill's `@layer base` block has no declared layer order to sit in. |
| `--font-line-height-body: 1.4` | A separate furo token at the same file, line 68. **Not this skill's token**; the skill never mentions it. Do not substitute it for `--value-golden-ratio`. |

## Conflicts with the always-on rule (the rule WINS)

`D:\ORT\rules\05-frontend.md` is always-on and takes precedence. Two conflicts, named and left unresolved:

**1. The default value for non-prose elements.**

- **This skill:** the document-wide default is `--value-golden-ratio` = **1.618**, inherited by *all* text.
- **Rule 6-12:** `p` takes the golden ratio; **everything else defaults to `line-height: 1`**.
- These are different values — 1.618 vs 1 — for every non-`p` element. **The rule wins.**

**2. Whether to set `line-height` at all.**

- **This skill:** set it once on `:root` in the base layer, then override per case.
- **Rule 6-12:** *「`line-height` は指示がない限り使わない」* — do not use `line-height` unless instructed; it is not for building layout, only for making the line height of prose.
- **The rule wins.** For the checkpoint-15 screen (a heading, two field labels, a button label, a one-sentence error message — short strings, no prose `<p>`), the rule's reading is that no `line-height` is written at all, absent an instruction.
