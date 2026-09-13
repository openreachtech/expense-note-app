# hof-cp-button
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-cp-button/ -->

**Read the source above whenever this leaves a question open.**

Routes every clickable action trigger to `FuroButton`. Verified against the installed
`@openreachtech/furo-vue@1.3.2`; facts marked **[source]** were read from
`node_modules/@openreachtech/furo-vue/lib/components/atoms/FuroButton/` and are **not** in the skill.

## Import and identity

- Layer: atom. `import { FuroButton } from '@openreachtech/furo-vue'`
- **Never import the underlying headless primitive** — only the public `FuroButton` export.
- Manifest: `node_modules/@openreachtech/furo-vue/public/furo-vue/components.json` → `components[].name === 'FuroButton'`. Read it before writing markup to confirm this surface is still current.
- Not a form-control value component: **no `parcel.value`, no `v-model` contract.** Its only outward contract is the `click` event.

## Props

| Prop | Type | Default | Notes |
| --- | --- | --- | --- |
| `parcel` | `FuroButtonParcel \| null` | `null` | Furo behavior object. The only public prop; HTML attributes pass through. |

## `parcel` fields

| Field | Type | Default | Notes |
| --- | --- | --- | --- |
| `variant` | `FuroButtonVariant` | `'default'` | Visual style preset. |
| `size` | `FuroButtonSize` | `'default'` | Size preset. |
| `disabled` | `boolean` | `false` | Disables interaction; also respects a fallthrough `disabled` attribute. |
| `loading` | `boolean` | `false` | Shows the loading slot and blocks the click emit. |
| `asChild` | `boolean` | `false` | Merge props and behavior onto a single child element. |
| `type` | `'button' \| 'submit' \| 'reset'` | `'button'` | Native button type when not `asChild`. |

Vocabulary — the full closed sets **[source]**:

- `FuroButtonVariant` = `'default' | 'secondary' | 'destructive' | 'outline' | 'ghost' | 'link'`. There is **no** `primary`; the primary-looking style is `'default'`.
- `FuroButtonSize` = `'default' | 'sm' | 'lg' | 'icon'` (`'icon'` is the square size).

Pass-through **[source]:** any parcel key *not* in `variant|size|disabled|loading|asChild|type` is
forwarded onto the root element as an attribute. The root binds in the order
`rootAttributes → primitiveParcel → ariaAttributes → $attrs`, so a **plain attribute on the
component wins over everything** — that is how `aria-label`, `id` and `type` are supplied.

## Events

| Event | Payload | Fires when |
| --- | --- | --- |
| `click` | `ButtonEmitPayload` | User activates the button. **Not fired while `disabled` or `loading`.** |

`ButtonEmitPayload` **[source]** extends `BaseEmitPayload<MouseEvent \| PointerEvent, HTMLElement>`;
is built from `{ rawEvent }`; exposes `get controlElement` (`currentTarget ?? target`, else `null`)
and `isDefaultPrevented()`.

## Slots

| Slot | Description |
| --- | --- |
| `default` | Button label or icon content. |
| `loading` | Custom spinner in the loading overlay (default: `ph:circle-notch`). |

## Loading vs disabled — what the sign-in submit depends on

| | `loading: true` | `disabled: true` |
| --- | --- | --- |
| `click` emit | suppressed | suppressed |
| native `disabled` attribute | **yes** — set from `isDisabled() \|\| isLoading()` **[source]** | yes |
| `aria-disabled="true"` | **no** — only `disabled` sets it **[source]** | yes |
| `aria-busy="true"` | yes | no |
| root class | `loading` | `disabled` |
| label | **hidden** — `visibility: hidden`, spinner overlays it **[source]** | unchanged, `opacity: 0.5` |
| pointer events | `pointer-events: none`, `cursor: not-allowed` | same |

- Both states set the native `disabled` attribute, so a loading button is **not focusable and not in the tab order** **[source]**.
- The label keeps its box while loading (`visibility: hidden`, not `display: none`), so **the button width never shifts**.
- The loading overlay carries `aria-hidden="true"` **[source]**.
- A double submission is structurally impossible while `loading` is true: the emit is suppressed in `onClick` **and** the native `disabled` attribute is set. Binding `loading` to the in-flight flag is the whole mechanism — no extra guard belongs in the template.

## Submit semantics

- The root renders a real **`<button>`** (`as: 'button'`) unless `asChild` is true **[source]**. Set `type: 'submit'` (parcel) or `type="submit"` (attribute) and it participates in the enclosing `<form>` natively — the form's own `@submit` fires.
- **No logic in the template.** What a click does (submit a form, call a Submitter, navigate) is a **named method on the page Context** (e.g. `onClickSubmit`); the template only forwards `$event` to it.
- `asChild` merges button behavior onto a single child element (anchor, dropdown trigger).

## When NOT to use

| Need | Use instead |
| --- | --- |
| Menu of actions behind one trigger | `hof-cp-dropdown-menu` (its default trigger slot is already `FuroButton`-styled; don't nest another `FuroButton` inside it unless overriding the trigger content) |
| Floating tooltip/panel anchored to a trigger | `hof-cp-popover` |
| Modal/drawer panel | `hof-cp-dialog` (the element that opens it is still `FuroButton`) |
| Toggle holding boolean state | `hof-cp-checkbox-toggle` / `hof-cp-toggle-group` |
| Inline value editor, not an action | `hof-cp-editable-field` |

## Usage

The skill's example, rewritten to `D:\ORT\rules\` house style — the skill's own example violates the
chop-down and no-logic-in-template rules (see Conflicts):

```vue
<template>
  <FuroButton
    :parcel="context.submitButtonParcel"
    type="submit"
    @click="context.onClickSubmit($event)"
  >
    Sign in
  </FuroButton>
</template>
```

The parcel is built in the Context class, not in the template:

```js
/**
 * get: Parcel of the submit button.
 *
 * @returns {import('@openreachtech/furo-vue').FuroButtonParcel} Parcel of the button.
 */
get submitButtonParcel () {
  return {
    variant: 'default',
    loading: this.isSigningIn,
  }
}
```

`asChild` composition onto an anchor:

```vue
<template>
  <FuroButton :parcel="context.helpLinkParcel">
    <a href="/help">Get help</a>
  </FuroButton>
</template>
```

## Appearance and tokens — nothing here is self-contained

**[source]** The component ships `<style scoped>` wrapped in `@layer furo`, and every colour and
dimension is a `var()` reference it does not itself declare:

`--color-primary` / `--color-primary-foreground`, `--color-secondary(-foreground)`,
`--color-destructive(-foreground)`, `--color-input`, `--color-accent(-foreground)`,
`--color-foreground`, `--color-link`, `--color-ring`, `--size-space-2x-small`,
`--size-border-radius-medium`, `--size-thinnest`, `--size-input-height`, `--font-family`,
`--font-size(-small/-large)`, `--font-weight-medium`, `--font-line-height-body`, `--transition-timer`.

Those tokens live in `node_modules/@openreachtech/furo-vue/lib/assets/css/furo.css`
(`0010.variables-palette-color-scale.css` … `0040.variables-component-furo.css`).
**This project imports none of them** — `nuxt.config.js` loads only `~/assets/css/variables.css`
(which declares nothing at all) and `~/assets/css/main.css`. Every token above therefore resolves to
nothing today. See Conflicts.

Also **[source]:** because the component's rules sit in `@layer furo` and the application declares no
layer order, unlayered application CSS always beats them.

The root class is `.unit-button`, plus the variant name, plus `size-<size>` for non-default sizes,
plus `disabled` / `loading` — consistent with the `unit-` prefix rule in `05-frontend.md` 6-7.

## Accessibility (project target: WCAG 2.2 AA)

**The skill says nothing about accessibility.** Everything below is **[source]**, and it is what the
checkpoint 18 audit will look at:

- **Accessible name.** The label lives in the default slot. While `loading`, that slot is
  `visibility: hidden` — which removes it from the accessibility tree — and the spinner is
  `aria-hidden="true"`. A loading button therefore has **no accessible name from its content**;
  supply a durable one (`aria-label` as a plain attribute, which wins over everything).
- **Announced while loading.** `aria-busy="true"` is set; `aria-disabled` is **not**. But the native
  `disabled` attribute **is** set, and disabled controls are not focusable — so a user focused on the
  button loses focus when it enters the loading state. Decide where focus goes; the component does
  not handle it.
- **Focus visibility.** The only focus indicator is
  `.unit-button:focus-visible { outline: none; box-shadow: 0 0 0 0.125rem var(--color-ring) }`. It
  **removes the native outline** and replaces it with a shadow built from `--color-ring`. With that
  token undeclared — as it is here — the result is **no visible focus indicator at all**, a direct
  WCAG 2.4.7 / 2.4.11 failure.
- **Disabled is not visual-only.** The native `disabled` attribute is genuinely set, so a disabled
  `FuroButton` is correctly removed from the tab order and exposed as disabled.

## Conflicts — named, not resolved

1. **Skill example vs. no-logic-in-template / chop-down.** The skill writes
   `:parcel="{ variant: 'destructive', loading: context.isDeleting }"` and
   `@click="context.onClickDelete({ payload: $event })"` — inline single-line object literals in a
   template. `D:\ORT\rules\javascript-style.md` (object-literal arguments are always chopped, one
   property per line, trailing comma) and the project's no-JS-logic-in-template rule both bar this.
   **`D:\ORT\rules\` wins.** The Usage section shows the shape the rules imply; confirm the
   project's preferred seam for building the parcel.
2. **Design tokens vs. the `05-frontend.md` colour-literal ban.** The component's entire appearance
   comes from furo's own `--color-*` / `--size-*` tokens, which this project does not load, and
   `assets/css/variables.css` declares nothing. `05-frontend.md` 6-2 requires a two-tier
   `--palette-*` → `--color-*` scheme in `variables.css` and bans colour literals — so neither
   writing hex into the component nor silently adopting furo's palette as the application's design
   decision is available. **The rule wins; the tokens are simply missing.** Someone must decide.
3. **Focus indicator vs. WCAG 2.2 AA.** The component suppresses the native outline and depends on
   `--color-ring`, which is undeclared here, leaving zero focus indication. **WCAG wins** — this
   blocks the checkpoint 18 audit until a token or an application-level focus style exists.
4. **Loading label vs. accessible name.** `visibility: hidden` on the label plus `aria-hidden` on
   the spinner leaves a loading button nameless. **WCAG wins**; an `aria-label` is needed, but
   whether that is the button's job or the form's is not settled here.
5. **`sm` / `lg` vs. the no-abbreviation rule.** `05-frontend.md` 6-14 and `naming.md` forbid
   abbreviations, but `FuroButtonSize` is a closed library vocabulary of `'sm'` / `'lg'`. Library
   values must be passed verbatim; **the rule still governs every name this project writes**
   (`button`, never `btn`).
6. **Skill/manifest imprecision.** The manifest says disabled and loading "set aria-disabled /
   aria-busy". Source shows `loading` sets `aria-busy` **only**, plus the native `disabled`
   attribute — it never sets `aria-disabled`. Trust the source.
