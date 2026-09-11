# hof-css-coding-styles
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-css-coding-styles/ -->

**Read the source above whenever this leaves a question open.**

Conventions for CSS coding style (formatting and notation). The whole skill is two rules, both about
line breaks *between* and *around* selectors. It says nothing about what goes *inside* a selector.

## Selector lists — chop down right after each comma

In a selector list (comma-separated), break the line right after each `,`, putting **one selector per
line**. **No exceptions — always chop down, even with just two selectors.**

```css
/* NG: multiple selectors on one line */
.a, .b, .c {
  color: #cc3300;
}

/* OK: chop down right after each comma */
.a,
.b,
.c {
  color: #cc3300;
}
```

## One blank line between selectors (rules)

Between one selector (rule) and the next, insert **exactly one blank line**. When the next rule is
preceded by a comment, **keep the blank line and put it before the comment** — the comment stays
attached to the rule below it.

```css
/* OK: one blank line between rules; keep it even when a comment precedes the next rule */
.a {
  color: #cc3300;
}

/* styles for b */
.b {
  color: #cc3300;
}
```

## What this skill does NOT cover

Do not read silence here as permission. The skill states no rule on any of the following, so the
always-on `rules/05-frontend.md` and the sibling skills below govern them:

| Question | Where it is actually answered |
|---|---|
| **Property order within a selector** | **NOT here.** Sibling skill `hof-selector-props-sort` ("Outer-to-Inner Order": categorize by what the property applies to, sort outer → inner; **within a property group, sort alphabetically**), plus `rules/05-frontend.md` §6-5. Read that skill — checkpoint 15 must not take property order from this digest. |
| Blank lines *inside* a selector | `rules/05-frontend.md` §6-5 — blank line where the meaning changes (Outer / Main / Inner). |
| Indentation width | `rules/05-frontend.md` §6-1 — always 2 spaces. |
| One declaration per line, spacing around `:` and `{}`, quoting, long-value wrapping | Unstated in this skill. The skill's own code samples show `property: value` with a single space after the colon, one declaration per line, and `{` on the selector's last line — follow the samples. |
| Comment style beyond blank-line placement | Unstated; samples use `/* ... */` only. |
| Shorthand properties, units, combinators, class naming, media-query placement | `rules/05-frontend.md` §6-2/6-3/6-4/6-6/6-7/6-8/6-11, and siblings `hof-css-props-prohibits`, `hof-css-units`, `hof-css-prohibits`, `hof-css-props-naming`. |

full text: .claude/skills/hof-css-coding-styles/SKILL.md (the source is 54 lines; the two rules above are its entirety)

## Conflicts and agreements with `rules/05-frontend.md`

`D:\ORT\rules\05-frontend.md` is always-on and **WINS** wherever it speaks.

**Conflicts: none.** This skill and the rule do not overlap on a single point. The skill governs
line breaks between selectors; the rule governs indentation, shorthand, property order, units,
combinators, naming, and media-query placement. There is nothing to resolve.

**Agreements — settled, do not re-check:**

- **2-space indentation** — the rule mandates it (§6-1); every code sample in this skill uses 2 spaces. Consistent.
- **Comment syntax** `/* ... */` — used by both, uncontradicted.
- **Property order outer → inner with a blank line where meaning changes** (rule §6-5) stands unopposed; this skill's "blank line between rules" is about the gap *between* rules and does not touch the blank lines *inside* one.

**Rule-only mandates this skill is silent on — they still bind** (listed so the implementer does not
mistake this skill's silence for an exemption): no shorthand properties at all (§6-4); `rem` never
`px` (§6-6); media queries written **inside** the selector they modify (§6-11) — note that a nested
media query is a block inside a rule, so the "one blank line between rules" rule above applies to it;
child combinator `>` never the descendant combinator (§6-8); semantic class names, never Tailwind-like
(§6-3); `unit-` prefix on a layout block's root (§6-7).
