# hof-css-props-naming
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-css-props-naming/ -->

**Read the source above whenever this leaves a question open.**

## 1. The top prefix denotes the kind

> "A custom property denotes what kind of value it holds through its **top prefix**."
> The first word right after `--` is the **top prefix**. From the prefix you can tell which kind of
> value the variable holds.

| Top prefix | Kind | Example |
| --- | --- | --- |
| `--palette-*` | Raw color values. Colors with no meaning | `--palette-sky-500` |
| `--color-*` | Semantic colors. Named by their use | `--color-primary`, `--color-text` |
| `--size-*` | Dimensions. Length / width / height | `--size-header-height` |
| `--value-*` | Unitless scalars. Ratios and base values | `--value-golden-ratio` |
| `--motion-*` | The time axis of transition / animation. duration, easing, delay | `--motion-duration-base`, `--motion-easing-standard` |

**That table is the whole prefix list the skill gives — and it is deliberately not closed:**

> Top prefixes are **added as needed**. The four above are not a fixed set. When a new kind is
> needed, do not force it into an existing prefix — stand up a new prefix. The set of prefixes stays
> open; the convention itself is the framework of "stand up a prefix per kind".

(The source says "four" while listing five rows; the table is the authority.)

Which to use, as a decision rule:

| The value is | Prefix |
| --- | --- |
| a raw color, meaningless on its own | `--palette-` |
| a color named for what it is used for | `--color-` |
| a length / width / height | `--size-` |
| a unitless scalar — a ratio, a base number (z-index layer bases are cited as an example) | `--value-` |
| a duration, easing or delay | `--motion-` |
| a kind none of the above fits | a **new** top prefix — never stretch an existing one |

## 2. Put the more distinctive word last

The top prefix (the broadest kind) comes first; the more distinctive — more specific, more
qualifying — a word is, the later it comes. Read left to right, "broad kind → individual qualifier":

```
--color-background-header
   └kind  └role       └which (distinctive)
```

- **A component-specific or state-specific property therefore appends the distinctive word at the
  end**, never the front: `--color-background-header`, not `--header-background-color`.
- Distinctive-first (`--header-background-color`) is **forbidden**: it scatters siblings across
  lexical sorting and `--color-` autocomplete, and CSS — flat, module-less, all of `:root` — has no
  namespace outside the name to restore the index.
- This is the **reverse of JavaScript class naming** (`AlphaSample` keeps the distinctive word in
  front, the kind behind). Do not carry class-naming habits into custom properties.

Casing and separators: every name in the source is **lowercase, hyphen-separated (kebab-case)**,
prefix first. The skill states this only by example, never as a sentence.
full text: .claude/skills/hof-css-props-naming/SKILL.md#put-the-more-distinctive-word-last

## 3. palette and color — two layers, one-way flow

> Colors alone are handled in two layers, `--palette-*` and `--color-*`. **No other kind has this
> two-layer structure.**

- **palette layer** (`--palette-*`): raw color values. No meaning. The **internal** layer.
- **color layer** (`--color-*`): colors given meaning by use. The **only public interface** the
  application touches.

```
--palette-*  →  --color-*  →  application CSS
(raw, internal)  (meaning, public)  (references color only)
```

### (1) The application may call only `--color-*`

```css
.button {
  background: var(--color-primary);      /* ✅ call the color layer */
}
```

```css
.button {
  background: var(--palette-blue-500);   /* ❌ do not call palette directly */
}
```

> Calling palette directly scatters raw color values through the application and skips the layer of
> meaning. palette is referenced only by the color layer.

### (2) A `--color-*` value is either `var(--palette-*)` or a literal

```css
:root {
  --color-primary: var(--palette-sky-500);   /* ✅ reference a palette */
  --color-border:  #ccc;                      /* ✅ a directly defined color */
}
```

> A literal color (`#ccc` and the like) may be written **only here**, at the color definition.
> Because the application can call only `--color-*`, raw color values are sealed in here and never
> leak into the application CSS.

(See conflict 1 — the always-on rule is stricter about that literal.)

## 4. Palette definitions — the template

`templates/property-palette.css` is the concrete example of `--palette-*`. Shape, not reproduced here:

- `:root { … }`, one `/* Hue */` comment block per hue.
- **Each hue is aligned in 11 steps from `050` to `950`**: `050 100 200 300 400 500 600 700 800 900
  950` — the first step is **zero-padded to three digits** (`--palette-slate-050`, never `-50`).
- Value is a bare hex literal.
- Hues present: slate, gray, zinc, neutral, stone, red, orange, amber, yellow, lime, green, emerald,
  teal, cyan, sky, rose. **No `blue`** — the hue the consumer repository's library already ships.

full text (all 209 lines, every hex): .claude/skills/hof-css-props-naming/templates/property-palette.css

## 5. What this does NOT say

Each of these is absent from the source, not compressed out of it. Do not infer an answer from this
digest:

- **Whether an app may extend or add to a library's tokens, and whether an app-level `--color-*` may
  reference a library-supplied `--palette-*`.** The skill draws **no library/application
  distinction** at all. What it does say applies by prefix regardless of who declared the property:
  a `--color-*` definition may take `var(--palette-*)` (§3-2), and application CSS may never call
  any `--palette-*` (§3-1). Consequence for this repository: referencing furo-vue's
  `--palette-blue-500` from inside an app `--color-*` definition is within the written rule;
  reaching for it from a component's CSS is not. Confirm the intent before relying on it.
- **Redefining / overriding a library `--color-*`** (furo-vue already declares `--color-ring`,
  `--color-destructive`, `--color-foreground`, `--font-*`, `--size-space-*`): not addressed.
- **State naming** (`hover`, `focus`, `disabled`, `active`) has no stated form. Only the ordering
  rule of §2 constrains it — the state word is distinctive, so it goes **last**.
- **Theming / dark mode, `@media` or `[data-theme]` scoping, fallbacks in `var(…, …)`**: not
  addressed.
- "the z-index convention" is named as a related topic (layer base values + `calc()` notation) but is
  **not** part of this skill; no such file exists in this directory.

## 6. Conflicts with the always-on rules

Named, not resolved. `D:\ORT\rules\` is always-on and **wins** in each.

1. **A color literal at the `--color-*` layer.** The skill explicitly permits `--color-border: #ccc`
   as one of the two legal `--color-*` values. `D:\ORT\rules\05-frontend.md` 6-2 says a color literal
   (hex / rgb) is never written directly as a property value — colors must be defined as CSS
   variables in `variables.css`, and its worked example is strictly two-tier, every `--color-*`
   resolving to a `var(--palette-*)`. **The rule wins:** declare the `--palette-*` first and point
   the `--color-*` at it. Reserve the skill's literal escape hatch for a case the rule's author
   sanctions.
2. **Appearance in a name — where the ban does and does not bite.** Both sources agree that
   `--color-*` is named by **use** (`--color-article-title` ✅, `--color-dark-gray` ❌ — the name
   becomes a lie the moment the theme changes). They also agree that the **palette layer is named by
   appearance on purpose** (`--palette-sky-500`; the rule's own example is `--palette-white` /
   `--palette-gray-300`). No conflict — but the use-not-appearance ban applies to the `--color-*`
   layer and **not** to `--palette-*`, so do not "fix" a hue name. Where they could pull apart is a
   non-color prefix: nothing in either source blesses an appearance-flavoured `--size-*` or
   `--motion-*`. Name those by use too.
3. **Abbreviations inside a property name.** `D:\ORT\rules\naming.md` and `05-frontend.md` 6-14 ban
   abbreviations outright. The skill abbreviates nothing, so nothing here excuses one:
   `--size-button-height`, never `--size-btn-height`; `--color-background-navigation`, never
   `--color-background-nav`. **The rule wins.**
4. **`--size-*` values are constrained by a rule the skill never mentions.** `05-frontend.md` 6-6
   requires `rem` (never `px`) and permits only the decimals `.1 .2 .25 .3 .4 .5 .6 .7 .75 .8 .9`,
   with decimals barred from dimensions larger than a font-size. The skill says nothing about units.
   **The rule wins** for every `--size-*` this project declares — including when mirroring a
   library's `--size-space-*` value.
