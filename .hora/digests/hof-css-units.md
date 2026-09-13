# hof-css-units
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-css-units/ -->

**Read the source above whenever this leaves a question open.**

`SKILL.md` is a 13-line table of contents only; the rules live in `rem-base.md` (base unit, 12 steps, scope, fixed px, px→rem conversion) and `zero.md` (units on zero). `references/objections.md` is rebuttal material for px advocates — no rule, not digested.

## The base unit is rem, never px

- Decide **all** CSS dimensions in rem. Do not build layout, spacing, or font sizes on a px basis.
- `1rem` = the root element's font size (the height of a character) treated as 1. **"`1rem = 16px`" is not an identity** — 16px is only the baseline default; the px a rem resolves to moves with the user's font-size setting (phone and PC alike).
- Why: px nullifies the user's font-size setting — an accessibility violation. With rem the whole design scales with the font size the user chose.
- `padding` / `margin` on a `<button>` etc. must be rem too, or the ratio to the font size breaks when text is enlarged or reduced.

## Value granularity — the 12 steps

Limit rem values to these steps. The table is 0–1; **values of 1 and above use the same granularity** (`1.25`, `1.75`, …).

| Kind | Values |
| --- | --- |
| One decimal place (10 steps) | `0.0` `0.1` `0.2` `0.3` `0.4` `0.5` `0.6` `0.7` `0.8` `0.9` |
| One quarter / three quarters (2 steps) | `0.25` `0.75` |

- In effect: "round to one decimal place, but `0.25` / `0.75` are allowed as exceptions."
- OK: `0.4rem`, `0.75rem`, `1.5rem`, `2rem`, `1.25rem`. **Not OK: `1.35rem`.**
- **Never four-decimal values such as `0.0625rem`** — it shows off the mistaken "1px = 1/16rem" premise, and a 0.0005rem difference is below both the eye and the dip resolution.
- Thirds (1/3, 2/3) are **not** allowed as steps — humans cannot estimate them at a glance the way they can halves and quarters.

### When the 12 steps cannot express the value

Express it by **combining 12-step-legal numbers with `calc()`**. Never write a raw decimal such as `0.3333333rem` or `1.5625rem`.

```css
.col { inline-size: calc(1rem / 3); }

:root {
  --ratio: 1.25;
}

h3 { font-size: calc(1rem * var(--ratio)); }                /* = 1.25rem */
h2 { font-size: calc(1rem * var(--ratio) * var(--ratio)); } /* don't write 1.5625rem raw */
```

Keep a type-scale ratio itself a graspable number (`1.25`, `1.2`; a perfect fourth is `calc(1rem * 4 / 3)`), composed through a Custom Property.

## Scope — which unit for which property

rem is relative to font size, so **use rem only where there is a reason to align to the font size, and whenever you use rem, always use the 12-step rem.**

| Specification | Unit |
| --- | --- |
| Font-size-derived: `padding`, `margin`, `gap`, `font-size`, text-coupled dimensions (and `box-shadow` if you judge a font-size basis looks better) | 12-step rem |
| `line-height`, `z-index`, `opacity`, `flex-grow` | **unitless** — out of scope for the 12 steps (`line-height: 1.5`, never `1.5rem`) |
| Layout proportion, fluid sizing, viewport- or container-based | `%`, `fr`, `ch`, `vw`, `vh`, `dvh`, `clamp()`, `min()`, `max()`, `cqi` — **free to use**. The rule is "do not base things on px," not "use only rem." |
| **Responsive breakpoints** | rem — they affect the number of characters shown across the width |

```css
/* NG: wrong unit for the job */
.card {
  line-height: 1.5rem;    /* should be unitless */
  gap: 6px;               /* font-based spacing fixed in px → use 0.4rem */
  inline-size: 480px;     /* proportion belongs in % / fr, not fixed px */
}
```

## Zero values

- A **`<length>` zero is written `0`, with no unit.** Never `0px` / `0rem` / `0em` — this covers `margin-block`, `inset-block-start`, `border-width`.
- A type where a unit is **syntactically required** keeps it at zero: `<time>` → `0s` / `0ms`, `<angle>` → `0deg`. A bare `0` is *invalid* for `<time>`, so `transition-delay: 0` and `animation-duration: 0` are broken CSS. When zeroing a value derived from `--motion-*`, write `0s`.

## Handling fixed px values

- As a rule, **do not write raw px directly in the CSS of the actual app.**
- If a fixed px is unavoidable, **name it as a CSS Custom Property first, without exception.** If you cannot give it a meaningful name, that proves px was not necessary.
- The representative example is the hairline (thinnest displayable width, e.g. a one-physical-pixel rule): define `--hairline-width` in one place and reference it everywhere. (→ conflicts below.)

## Refactoring existing px styles

Convert the px value to exact rem using `1rem = 16px` — the premise held by whoever wrote it — then round to the **nearest of the 12 steps**. The rounding is uniquely determined (a px step of 0.0625rem exceeds the 0.05rem half-width, so two px values never land on the same side of a target).

| px | Exact rem | Adopted |
| --- | --- | --- |
| 0 | 0 | `0.0rem` |
| 1 | 0.0625 | `0.1rem` |
| 2 | 0.125 | `0.1rem` |
| 3 | 0.1875 | `0.2rem` |
| 4 | 0.25 | `0.25rem` |
| 5 | 0.3125 | `0.3rem` |
| 6 | 0.375 | `0.4rem` |
| 7 | 0.4375 | `0.4rem` |
| 8 | 0.5 | `0.5rem` |
| 9 | 0.5625 | `0.6rem` |
| 10 | 0.625 | `0.6rem` |
| 11 | 0.6875 | `0.7rem` |
| 12 | 0.75 | `0.75rem` |
| 13 | 0.8125 | `0.8rem` |
| 14 | 0.875 | `0.9rem` |
| 15 | 0.9375 | `0.9rem` |

For 16px and above, divide by 16: quotient = integer part, look the remainder (0–15px) up above, add them (20px = 16 + 4 → `1.25rem`; 22px = 16 + 6 → `1.4rem`). `16px` is absent from the table because `16px = 1rem` — it carries into the integer part.

## Not covered by this skill

There is no per-topic section for **border radius**, **border width**, or **icon sizing** beyond the general Scope table (font-size-derived → 12-step rem) and the `border-width: 0` zero example. No breakpoint *values* are prescribed — `48rem` appears only inside an example. Nothing on colors, `line-height` design policy, class naming, or property order (those live in `D:\ORT\rules\05-frontend.md`).
full text: .claude/skills/hof-css-units/rem-base.md#scope

## Where `D:\ORT\rules\05-frontend.md` and this skill disagree — the rule WINS

The always-on rule carries its own unit section (6-6) and is authoritative. Do not average these; take the rule's value.

| Point | Skill says | Rule says — **follow this** |
| --- | --- | --- |
| Decimals on a **block / inline-block `width`/`height`**, and on `<video>` / `<iframe>` / `<img>` | No such restriction; its own example writes `.col { inline-size: calc(1rem / 3); }`, and `max-inline-size` is "an appropriate rem per case" | **Absolute decimals only below font-size scale.** No decimals on those dimensions; decimals stay fine on border-radius, padding, margin, gap, font-size, icons. Prefer `calc((100% / 3) - 0.9rem)` over a decimal width |
| A fixed `1px` hairline | Permitted when named as a Custom Property (`--hairline-width: 1px`) | **All dimensions in `rem`, not `px`** (6-6) — no hairline carve-out is stated |
| Media-query form | `@media (min-width: 48rem)`, nested inside the selector | Range syntax, nested inside the selector: `@media (30rem < width)` (6-11). The unit (rem) agrees |
| Shorthand in the skill's examples (`padding: 0.75rem`, `border-block-end: var(--hairline-width) solid #ccc`) | Used freely | **Shorthands are banned** (6-4) — write `padding-block` / `padding-inline`, and `border-*-width` / `-style` / `-color` separately |
| Color literal in the skill's examples (`#ccc`) | Written inline | **Color literals are banned** (6-2) — define in `variables.css` as `--palette-*` → `--color-*` |

**Where they agree** (nothing to decide): rem is the base unit and px is not; the permitted decimal set is identical — the rule's `.1 .2 .25 .3 .4 .5 .6 .7 .75 .8 .9` equals the skill's 12 steps minus `0.0`; 16px-derived values such as `0.625rem` and `0.0625rem` are banned by both (the skill rounds 10px to `0.6rem`); thirds are written `100% / 3` or `calc(1rem / 3)`, never `33.333%` or `0.3333333rem`; breakpoints are in rem.
