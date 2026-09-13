# hof-prohibits
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-prohibits/ -->

**Read the source above whenever this leaves a question open.**

The whole skill is one 57-line `SKILL.md` (no `references/`). It states **two** prohibitions and one preference. It says nothing about props, emits, slots, component boundaries, or CSS — see "Not covered" below.

## Prohibition 1 — no JavaScript logic in a `.vue` `<template>`

- Do not write JavaScript **logic** inside a `.vue` `<template>`. Put logic on the members of the `<script>`'s `context` (the object returned by `setup`), and **expose it as a method / getter that returns the final value**. The `<template>` only calls it bare.
- "Logic" means **anything that makes a decision or a computation, and is therefore a target of unit testing**.
- The criterion is "**the test target is a member of `context`**". A decision or computation can be unit-tested only once it lives on `context`. Written in the `<template>`, untestable logic gets buried in the view. So even a small negation like `!` moves to a `context` method that returns the final state.

### Where the line falls

| In a `<template>` expression | Verdict |
| :-- | :-- |
| operators — `!`, `&&`, `\|\|`, comparison, arithmetic | **NG — is logic** |
| the ternary operator | **NG — is logic** |
| method chains that transform data, such as `.filter().map()` | **NG — is logic** |
| inline conditionals or arrow-function bodies | **NG — is logic** |
| bare access to a `context` property / method | OK — not logic |
| constructing an object / array literal | OK — not logic |
| spreading | OK — not logic |
| passing `<template>` magic variables such as `$event` / `$attrs` as arguments | OK — not logic |

The OK row's rationale, verbatim: "These carry no decision or computation and are not a test target."

```vue
<!-- NG: the <template> contains logic, the ! -->
<button :disabled="!context.canScrollPrevious()">
```

```vue
<!-- OK: call a context member named for the state in the positive -->
<button :disabled="context.isDisabledScrollPreviousButton()">
```

```vue
<!-- OK: these carry no decision or computation, so they are not a test target -->
<TabsRoot
  v-bind="{
    ...context.extractRootAttributes(),
    ...$attrs,
  }"
  @update:model-value="context.onTabChange({
    value: $event,
  })"
>
```

## Prohibition 2 — no negative words in an extracted predicate's name

Do **not** use negative words in the name of an extracted predicate. Do not name it `cannot~` / `not~` / `no~`; name it for the state it represents, in the positive (for a value passed to `:disabled`, `isDisabledSubmitButton()`). A negative name produces double negatives such as `!context.isDisabledSubmitButton()`, which read poorly.

## Preferred, not required — hide a literal merge behind a `build`-style member

Literal construction such as the `...$attrs` merge can be hidden further from the `<template>` by passing the magic variable to a `context` member and letting it assemble the object. **This is not required, but it is preferred**, since it keeps the `<template>` purely declarative. The method name will be a `build`-style one such as `buildXxxxx({ attrs: $attrs })`.

```vue
<!-- Preferred: hide the merge in context (a build-style method returns the finished object) -->
<TabsRoot
  v-bind="context.buildRootAttributes({
    attrs: $attrs,
  })"
  @update:model-value="context.onTabChange({
    value: $event,
  })"
>
```

## Against the always-on rules in `D:\ORT\rules\` — settled vs. behaviour-changing

| Point | Status |
| :-- | :-- |
| `else if` / `switch`, nested `if`, imperative loops, `forEach` | settled by `javascript-style.md`; this skill adds nothing — do not re-check |
| Tailwind-like class names, CSS shorthand | settled by `05-frontend.md`; this skill is silent on CSS |
| object-literal argument chopped one-property-per-line | agree — every skill example obeys `javascript-style.md` |
| boolean member named `is~` / `has~` / `can~` | agree with `naming.md` |
| **ternary** | **stricter in template position.** `javascript-style.md` permits a chopped-down ternary in JS; this skill forbids any ternary inside `<template>` |
| **`.map()` / `.filter()`** | **stricter in template position.** `javascript-style.md` *requires* these over loops in JS; this skill forbids a transforming chain inside `<template>` |
| **negative predicate names (`cannot~`/`not~`/`no~`)** | **stricter.** `naming.md` requires a third-person-singular/auxiliary prefix but does not ban negatives; this skill does |

Nothing in this skill is **looser** than the always-on rules.

## Not covered by this skill

**Props, emits, and component boundaries: the skill states no prohibition on any of them.** It never mentions `defineProps`, `defineEmits`, slots, or component granularity. Treat prop/emit conventions as owned by another skill (`hof-nuxt`) or by the always-on rules — do not infer a prohibition here.

Also uncovered: the skill shows a *called* context member (`context.onTabChange({ … })`) but never a bare handler **reference** (`@click="context.onSubmit"`). That form falls under "bare access to a `context` property / method" = OK, but the skill gives no example.
full text: .claude/skills/hof-prohibits/SKILL.md

## Conflicts — flagged, not resolved

1. **`attrs` is a banned abbreviation.** The skill's preferred shape is `buildRootAttributes({ attrs: $attrs })`, but `naming.md` forbids abbreviations outside its whitelist, and `attrs` (= `attributes`) is not on it. **The always-on rule wins** — the *parameter* should be `attributes`. (`$attrs` itself is a Vue magic variable and cannot be renamed.) Do not resolve this yourself; raise it.
2. **Template `v-bind` object literal vs. chop-down.** The skill permits an inline `{ ...a, ...$attrs }` literal in a `<template>`; `javascript-style.md` requires object-literal arguments chopped one property per line. The skill's own example is already chopped, so the two agree in practice — **the rule wins** if a one-line literal is ever tempting.
