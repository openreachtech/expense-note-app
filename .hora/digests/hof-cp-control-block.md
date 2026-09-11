# hof-cp-control-block
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-cp-control-block/ -->

**Read the source above whenever this leaves a question open.**

`FuroControlBlock` — molecule that frames ONE form-field atom with a label, a required
mark and error messages. Domain-agnostic: the control goes in the default slot, so the
block never knows which atom it wraps.

- Layer: molecule
- Import: `import { FuroControlBlock } from '@openreachtech/furo-vue'`
  (verified against `@openreachtech/furo-vue@1.3.2` — `lib/index.components.js:28`
  re-exports it from the package root)
- Manifest: `node_modules/@openreachtech/furo-vue/public/furo-vue/components.json`
  → `components[].name === 'FuroControlBlock'`. Read the manifest before writing markup
  if you need to confirm this is still current.

## When NOT to use

| Need | Go to |
| --- | --- |
| the actual input control, not the label/error frame | the atom's own skill: `hof-cp-text-field`, `hof-cp-textarea`, `hof-cp-checkbox-toggle`, `hof-cp-select`, `hof-cp-date-time` |
| a floating tooltip / hint bubble, not an inline label+error block | `hof-cp-popover` |
| an inline "click to edit" affordance, not a standing labeled field | `hof-cp-editable-field` |

## Props

| Prop | Type | Default | Notes |
| --- | --- | --- | --- |
| `parcel` | `ControlBlockParcel \| null` | `null` | Reactive data object holding the label, control id, error messages, required flag, and orientation. |

## `parcel` fields

| Field | Type | Default | Notes |
| --- | --- | --- | --- |
| `label` | `string \| null` | `null` | Label text. When null, no label is rendered. |
| `controlId` | `string \| null` | `null` | Id of the slotted control, rendered as label `for`. When null, the label has no `for`. |
| `errorMessages` | `Array<string>` | `[]` | Error messages. A non-empty array marks the block invalid and renders one line per message. |
| `required` | `boolean` | `false` | When true, renders a required mark (*) after the label. |
| `orientation` | `'vertical' \| 'horizontal'` | `'vertical'` | Layout of label vs control. `vertical` stacks; `horizontal` places the label beside the control. |

## Events

None. `FuroControlBlock` emits nothing — the slotted control owns its own
`change-value` / `commit-value` / `update:value` events.

## Slots

- `default` — the control to frame. Pass the same id as the parcel `controlId` so label
  `for` resolves.

**There is no `hint` prop and no `hint` slot.** The manifest `summary` string reads
"Label, control, hint, and error wrapper", but no `hint` field exists in the parcel and
no hint slot exists in the template (verified: a `hint` grep over the component directory
returns nothing). A hint / description line has no home in this component.

## Label association (WCAG)

The block renders a real `<label :for="context.controlId ?? undefined">`. The association
is programmatic **only when the consumer passes the same id twice** — once as parcel
`controlId`, once as the `id` attribute on the slotted atom. The block cannot set the id
on the slot child itself. A `controlId` with no matching `id` on the control leaves the
label visually adjacent but unassociated.

The component carries a ponytail comment stating the known gap: the error region carries
`role="alert"` only; **there is no `aria-describedby` wiring from the error text into the
slotted control**, because the block cannot set attributes on a slot child. Its stated
upgrade path is "expose an errorId getter for consumers to bind". Neither the skill nor
the component supplies one today.

The required mark is `<span class="required-mark" aria-hidden="true">*</span>` — a visual
cue only, carrying no programmatic required semantics.

## Who owns validation display

| Concern | Owner | How |
| --- | --- | --- |
| error message **text** | `FuroControlBlock` | parcel `errorMessages` → `<ul class="error" role="alert">`, one `<li>` per message |
| invalid **state on the input** (`aria-invalid`) | the field atom | atom parcel `invalid: boolean` → `:aria-invalid` on the `<input>` |

Not a conflict — they are different surfaces, and the skill's own usage example sets both.
The atom parcel carries no message text (`FuroTextField` / `FuroEmailField` /
`FuroPasswordField` parcels are `{ value, invalid }` only). The consumer is the single
source of truth and must feed both: `errorMessages` to the block and `invalid` to the atom.

**Shown and cleared** purely by the value of `errorMessages`: non-empty renders the block
`invalid` and the message list; empty (`[]`) removes both. There is no imperative
show/clear API and no internal error state — clearing is the context getter returning `[]`.

## Form-level message (not attached to a field)

**The skill does not cover it.** It scopes the component to "any single form-field atom",
lists no form-level or summary role, and names no alternative component for one.

Mechanically the block *can* render a message with no field — `label: null`,
`controlId: null`, `errorMessages: ['…']` and an empty default slot produce the
`role="alert"` list alone — but that use is unsanctioned by the skill and leaves an empty
`<div class="control">` in the markup. Treat the home for a form-level refusal message as
an open question; do not infer it from this digest.

## Usage

```vue
<template>
  <FuroControlBlock
    :parcel="{
      label: 'Full name',
      controlId: 'full-name',
      errorMessages: context.fullNameErrorMessages,
      required: true,
    }"
  >
    <FuroTextField
      id="full-name"
      v-model:value="form.fullName"
      :parcel="{ invalid: context.hasFullNameError() }"
    />
  </FuroControlBlock>
</template>
```

(Shown as the skill writes it. See "Conflicts to flag" — the inline method call and the
un-chopped attribute both lose to `D:\ORT\rules\`.) Horizontal orientation is the same
shape with `orientation: 'horizontal'` added to the parcel.

A consumer-supplied `class` and `style` compose onto the block root
(`inheritAttrs: false` merges `$attrs.class` / `$attrs.style`), so a layout selector like
`.control.name { flex: 1 }` reaches the wrapper.

## Rules (per project conventions)

- `FuroControlBlock` is not a form-control itself, so it has no `value` and no
  `v-model:value` contract — pass `parcel` for label/error/orientation only, and put
  `v-model:value` on the slotted control atom instead.
- Never import the underlying headless primitive — only the public `FuroControlBlock`
  export.
- Compute `errorMessages` and `required` in the page/component Context (e.g. a
  `{field}ErrorMessages` getter), not inline in the template.

## Styling and colour source

The component ships `<style>` inside `@layer furo` and reads these custom properties:
`--color-foreground`, `--color-destructive` (required mark + error text),
`--font-label-small`, `--font-body-small`, `--size-space-tiny`, `--size-space-small`.

**None of them are defined in this application.** `@openreachtech/furo-vue` declares them
in `lib/assets/css/furo/0030.variables-semantic-color.css` and
`0035.variables-semantic-dimension.css`, but `nuxt.config.js` loads only
`~/assets/css/variables.css` and `~/assets/css/main.css` — no furo stylesheet is imported
anywhere in the app. `assets/css/variables.css` declares nothing (`:root {}` plus a
comment). So `color: var(--color-destructive)` resolves to an invalid value and the error
text inherits the ambient colour instead of rendering red; the `font` and `gap`
declarations collapse likewise.

The error styling's colour must therefore come from a semantic custom property the
application declares in `assets/css/variables.css`, two-tier (`--palette-*` →
`--color-*`), per `D:\ORT\rules\05-frontend.md` §6-2. The app declares no `@layer` order,
so unlayered application CSS wins over the component's `@layer furo` rules without a
specificity fight.

## Conflicts to flag (named, not resolved)

1. **Inline object literal in `:parcel`, and a method call inside it.** The skill's usage
   example writes the whole parcel as an inline object literal in the template, including
   `:parcel="{ invalid: context.hasFullNameError() }"` — a method call in an object-literal
   property value. `D:\ORT\rules\javascript-style.md` ("No logic in object-literal property
   values"; never a method call in a property value) and the standing constraint "no JS
   logic in a template — validation state comes from a getter on the page's context class"
   both bar this. **The rule wins.** The skill's own closing rule agrees in spirit
   ("Compute `errorMessages` and `required` in the page/component Context … not inline in
   the template") while its example contradicts it.
2. **Un-chopped attributes in the example.** `:parcel="{ invalid: context.hasFullNameError() }"`
   sits on one line, and the horizontal example's `FuroCheckbox` carries two attributes
   without chop-down. `D:\ORT\rules\05-frontend.md` §6-9 requires one attribute per line at
   two or more attributes. **The rule wins.**
3. **Manifest says "hint", component has none.** The manifest `summary` advertises a hint;
   the parcel and template have no hint field or slot. **The component's real surface
   wins** — there is no hint to supply.
4. **`role="alert"` without `aria-describedby`.** The component's own comment records that
   the error text is never wired to the control it describes. Against a WCAG 2.2 AA target
   this is a known gap in the library, not something the skill resolves — the skill offers
   no workaround and neither does this digest.
