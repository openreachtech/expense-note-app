# hof-css-props-prohibits
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-css-props-prohibits/ -->

**Read the source above whenever this leaves a question open.**

What is prohibited when defining custom properties. Both rules govern the properties **we define**; see "Third-party boundary" for what we merely consume.

## Relative sizes: five steps, and no more

- A relative-size custom property is limited to the **five steps `huge` / `large` / `medium` / `small` / `tiny`**. Do not create steps finer than these.
- Do **not use `x-` style labels** such as `x-large` / `xx-large` / `xs` / `xl`. They are a means of adding steps beyond the five.
- Reason: a difference finer than the five steps cannot be told apart by the human eye; tokenizing an indistinguishable difference introduces meaningless granularity.

**There is no escape hatch, and no sixth step.** When a dimension the five steps cannot cover is needed, do not add a relative size — **name it for the element's role** (→ the custom property naming rule, "Put the more distinctive word last"). At that point it is no longer a perceptual size scale but a dimension specific to the element.

| Case | Form |
| :-- | :-- |
| a perceptual size | one of the five steps, e.g. `--size-space-large` |
| a dimension the five steps cannot cover | a role name, e.g. `--size-header-height`, `--size-nav-width` |

```css
/* NG: finer than the five steps / x- style labels */
:root {
  --size-space-xs: 0.25rem;
  --size-space-x-large: 3rem;
  --size-space-xx-large: 4rem;
}
```

```css
/* OK: relative sizes go up to five steps */
:root {
  --size-space-tiny: 0.25rem;
  --size-space-small: 0.5rem;
  --size-space-medium: 1rem;
  --size-space-large: 2rem;
  --size-space-huge: 4rem;
}
```

## No abbreviations in property names

- Abbreviations in the custom property names **we define** follow the **whitelist** of the JavaScript naming rule ("criteria for using abbreviations") — `id` / `min` / `max` / `config` / `env`, and so on. Every other abbreviation is prohibited. The whitelist is shared between JS and CSS and not maintained twice.
- Whitelist-external abbreviations — `bg` (→ `background`), `hdr` (→ `header`), `btn` (→ `button`), `img` (→ `image`), and the relative-size `xs` / `xl` (→ `tiny` / `huge`) — are **never used in our own definitions**. Spell every word out in full.
- Reason: a whitelist-external abbreviation is completed differently by each reader, so its meaning does not carry. A custom property's **name is its only index**; a name that carries no meaning cannot serve as an index.

NG `--color-bg` / `--size-hdr-height` -> OK `--color-background` / `--size-header-height`.

**Third-party boundary — the one exception.** When Vue or a third-party module uses an ugly abbreviation **in its interface**, we cannot connect without matching it, so we follow their abbreviation at the boundary where we touch that interface (e.g. a `--reka-*` custom property we consume). The exception is limited to that boundary; that they use it is not a reason to use abbreviations in our own definitions (→ the JavaScript naming rule, "do not imitate external modules' abbreviations").

## Conflicts with `D:\ORT\rules\05-frontend.md` (always-on rule WINS)

Named, not resolved.

| # | Skill shows | `05-frontend.md` says | Outcome |
| :-- | :-- | :-- | :-- |
| 1 | `--size-header-height: 4.5rem` — an absolute decimal on a block's height | 6-6: absolute decimals only below font-size scale; **never on a block's width or height** | rule wins — a role-named block dimension takes an integer `rem` |
| 2 | `--color-background: #fff` — a hex literal straight into a semantic name | 6-2: two tiers, `--palette-white: #fff` then `--color-background: var(--palette-white)` | rule wins — declare the palette tier first; the skill's one-tier example is naming guidance only |

Agreements (settled, do not re-check): `rem` not `px`; the skill's `0.25` / `0.5` values are on 6-6's intuitive-decimal list; the granularity reasoning is explicitly "the same reasoning as the rem-based unit convention"; the no-abbreviation rule matches 6-14; role naming (`--size-header-height`) is 6-2/6-3 semantic naming, not look-based.

## Repo note — extending the furo scale (thin)

`@openreachtech/furo-vue` supplies `--size-space-*`, `--size-border-radius-*` and `--font-size-*` built from `x-large` / `2x-large` … `5x-large`, plus `--size-space-compact|snug|relaxed|spacious|none` — **the very labels this skill prohibits**, and it has no `huge`. It also already defines `--size-header-height` and `--size-nav-width`.

What the skill settles: the prohibition binds **the names we define**, and its only carve-out is consuming a third party's interface. So *reading* `--size-space-x-large` is the consumption boundary; *declaring* a new `--size-space-x-large` (or any `x-`/`2x-` step) in `assets/css/variables.css` is prohibited, and a new app-side relative size uses the five steps.

What the skill does **not** say: whether an app-side scale may extend a vendor scale whose steps already violate the rule, or must start a fresh five-step scale of its own. Resolve by reading `full text: .claude/skills/hof-css-props-prohibits/SKILL.md#prohibit-excessively-granular-relative-sizes` and the custom property naming rule it points at.
