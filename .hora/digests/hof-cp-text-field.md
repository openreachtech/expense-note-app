# hof-cp-text-field
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-cp-text-field/ -->

**Read the source above whenever this leaves a question open.**

Single-line text input atoms from `@openreachtech/furo-vue` (installed: 1.3.2).
Lines marked **(source-verified)** were confirmed against
`node_modules/@openreachtech/furo-vue/` and are not stated in `SKILL.md`.

## Which component

| Collecting | Use |
| --- | --- |
| email address | `FuroEmailField` |
| password | `FuroPasswordField` |
| plain single-line text | `FuroTextField` |
| number | `FuroNumberField` |
| file upload | `FuroFileField` |

**Never `FuroTextField type="email"` / `type="password"`.** The skill's reason,
verbatim: the typed variants "hard-code the underlying input semantics
(`type="email"`, `type="password"`, …) rather than relying on a parent passing a
native `type` attribute onto `FuroTextField`"; they "bake in the right payload
helpers, ARIA/ validation semantics, and (for number/file) a materially different
internal structure that `FuroTextField` lacks."

(source-verified) It is not merely discouraged but impossible: every atom renders
a hard-coded `type=` and `generateInputAttributes()` destructures `type` out of
the fallthrough attrs before spreading them, so a passed `type` is discarded.

### When NOT to use

- Multi-line text → `hof-cp-textarea` (`FuroTextarea`).
- Needs a label / hint / error message around it → wrap in `FuroControlBlock`
  (`hof-cp-control-block`), **don't hand-roll a `<label>`**.
- Value picked from a fixed or searchable list → `hof-cp-select`
  (`FuroSelect` / `FuroAutocompleteField`).
- In-place edit of an already-displayed value → `hof-cp-editable-field`.
- Date/time entry → `hof-cp-date-time`, not `FuroTextField` with manual formatting.

## Import

```js
import { FuroEmailField } from '@openreachtech/furo-vue'
import { FuroPasswordField } from '@openreachtech/furo-vue'
import { FuroTextField } from '@openreachtech/furo-vue'
import { FuroNumberField } from '@openreachtech/furo-vue'
import { FuroFileField } from '@openreachtech/furo-vue'
```

- Layer: atom.
- **Never import the underlying headless primitive** — only the public exports above.
- Manifest: `node_modules/@openreachtech/furo-vue/public/furo-vue/components.json`
  → `components[].name === 'FuroTextField' | 'FuroEmailField' | 'FuroPasswordField'
  | 'FuroNumberField' | 'FuroFileField'`. Read it if you need to confirm the surface
  is still current.

## Prop surface — email & password (the sign-in pair)

All five components declare **only `parcel`** as a public prop; native HTML
attributes pass through as fallthrough attrs.

| Component | Prop | Type | Default |
| --- | --- | --- | --- |
| FuroEmailField | `parcel` | `FuroEmailFieldParcel \| null` | `null` |
| FuroPasswordField | `parcel` | `FuroPasswordFieldParcel \| null` | `null` |
| FuroTextField | `parcel` | `FuroTextFieldParcel \| null` | `null` |

### `parcel` fields

| Component | Field | Type | Default | Notes |
| --- | --- | --- | --- | --- |
| FuroEmailField | `value` | `string \| null` | `null` | Controlled value, synced to the internal ref. |
| FuroEmailField | `invalid` | `boolean` | `false` | Applies the invalid / error visual state. |
| FuroPasswordField | `value` | `string \| null` | `null` | Controlled value, synced to the internal ref. |
| FuroPasswordField | `invalid` | `boolean` | `false` | Applies the invalid / error visual state. |
| FuroTextField | `value` | `string \| number \| null` | `null` | Controlled value, synced to the internal ref. |
| FuroTextField | `invalid` | `boolean` | `false` | Applies the invalid / error visual state. |

`parcel.value` is string-only on email and password; numeric values are not supported.

### Events (identical on all three)

| Event | Payload | Fires when |
| --- | --- | --- |
| `change-value` | `EmailFieldEmitPayload` / `PasswordFieldEmitPayload` / `TextFieldEmitPayload` | Emitted on every input event (each keystroke). |
| `commit-value` | same payload class | Emitted on blur or Enter key press. |
| `update:value` | `string \| null` | Raw string value emitted alongside `change-value` and `commit-value` for `v-model:value`. |

### Slots

`FuroTextField`, `FuroEmailField`, `FuroPasswordField`, `FuroFileField`: **no slots.**

### Emit payload API (source-verified)

`EmailFieldEmitPayload` / `PasswordFieldEmitPayload` extend `BaseEmitPayload`
and expose exactly:

| Member | Returns |
| --- | --- |
| `get value` | `string \| null` — current field value |
| `extractTrimmedValue()` | `string \| null` |
| `hasEmptyText()` | `boolean` (missing control counts as empty) |
| `hasEmptyTrimmedText()` | `boolean` |
| `hasEnteredText()` | `boolean` (whitespace counts) |
| `hasEnteredTrimmedText()` | `boolean` |
| `get rawEvent` | `InputEvent \| FocusEvent \| KeyboardEvent` |
| `get controlElement` | `HTMLInputElement \| null` |
| `getElementAttribute({ key })` | `string \| null` |

`TextFieldEmitPayload` additionally has a `get trimmedValue` getter.

## Binding the value, and the page Context

- **Use `v-model:value`** on `FuroTextField` / `FuroEmailField` / `FuroPasswordField`
  / `FuroNumberField`, alongside `parcel`. This is the **one documented exception to
  the project's general no-`v-model` rule**, which "applies only to custom in-project
  components, not to `furo-vue` library components."
- `FuroFileField` has no `value` / `update:value` — read the selected files from the
  `change-value` payload; do not try to bind `v-model:value`.
- **Put commit/validation logic in the page/component Context** (`on{Field}Commit`
  style method), **not inline in the template**. For this project that Context
  extends `BaseAppContext` (`app/vue/contexts/BaseAppContext.js`).
- (source-verified) The component watches both `parcel.value` and the `value`
  fallthrough attribute into its own internal ref, `immediate: true` — **no parent
  watcher is needed**, and that ref starts at `null`.
- (source-verified) `commit-value` fires on **blur and on Enter**; Enter inside a
  form therefore both commits the field and may submit — order accordingly.

## Validation state & error message

- The field component carries **only the `invalid` boolean** in its `parcel`; it
  renders the invalid visual state and (source-verified) binds `:aria-invalid` and
  an `.invalid` root class.
- **The message belongs to the wrapping `FuroControlBlock`, not the field.** A field
  that "needs a label / hint / error message around it" is wrapped — see
  `hof-cp-control-block`. Do not hand-roll a `<label>` or an error paragraph.
- (source-verified) `FuroControlBlock` renders `<label :for="controlId">` and a
  `<ul class="error" role="alert">` from `parcel.errorMessages`; its parcel is
  `{ label?, controlId?, errorMessages?, required?, orientation? }`.

## Disabled / readonly / loading, and autocomplete

`SKILL.md` says nothing about any of these. All of the following is
**(source-verified)** on the installed 1.3.2:

- There is **no `disabled`, `readonly`, or `loading` parcel field, and no loading
  state at all.** These are plain native attributes passed as fallthrough attrs:
  `disabled`, `readonly`, `required`, `name`, `id`, `placeholder`, `maxlength`,
  `autocomplete`, `inputmode` all reach the underlying `<input>` untouched. Only
  `type` is stripped.
- `:disabled` is styled (`cursor: not-allowed`, `opacity: 0.5`). `readonly` has **no**
  styling — a readonly field looks identical to an editable one.
- **`autocomplete` is entirely the caller's responsibility** — the library never sets
  it. For sign-in, pass it explicitly on each field:
  `autocomplete="username"` on the email field and
  `autocomplete="current-password"` on the password field, so password managers fill
  the pair.
- `FuroPasswordField` renders a bare `<input type="password">` — **there is no
  reveal/show-password toggle and no slot to add one.** See Conflicts.

## Accessibility (WCAG 2.2 AA — checkpoint 18)

- The field renders a **bare `<input>` with no label of its own.** A programmatic
  label comes only from wrapping in `FuroControlBlock` with `parcel.label` **and**
  `parcel.controlId`, and passing the **matching `id`** to the field as a fallthrough
  attribute. `controlId` without a matching `id` on the input silently produces a
  `<label for>` pointing at nothing.
- (source-verified) `FuroControlBlock` **does not wire `aria-describedby`** from its
  error region into the slotted control — the library's own source comment records
  this as a known gap ("the block cannot set attrs on a slot child"). The error text
  is announced via `role="alert"` only; if the audit requires the message be
  programmatically associated with the input, bind `aria-describedby` yourself as a
  fallthrough attr.
- (source-verified) `:focus-visible` sets `outline: none` and changes only
  `border-color` to `var(--color-ring)` — verify this against the focus-appearance
  criterion rather than assuming the library passes.
- (source-verified) The invalid state on the field is conveyed by border colour
  (`--color-destructive`) plus `aria-invalid` alone; the non-colour information is
  the control block's message.

## Number & file variants (thin — full text: SKILL.md "`parcel` fields" / "Events")

- `FuroNumberField` parcel: `value: number|null` (`null`), `invalid: boolean`
  (`false`, also sets `aria-invalid`), `min`, `max`, `step`, `stepSnapping`,
  `formatOptions: Intl.NumberFormatOptions`, `locale`, `inputParcel`,
  `decrementParcel`, `incrementParcel`. Slots `decrement` / `increment`
  (scoped `{ value }`; defaults `ph:minus` / `ph:plus`). Events `change-value` /
  `commit-value` (`NumberFieldEmitPayload`), `update:value` (`number|null`).
- `FuroFileField` parcel: `invalid` only — **no `value` field**, because browsers
  forbid scripting a file input's value; selection is read from the event payload.
  Only `change-value` (`FileFieldEmitPayload`); no `commit-value`, no `update:value`.

## Usage shape (skill's example, verbatim — see Conflicts before copying)

```vue
<template>
  <FuroEmailField
    v-model:value="form.email"
    :parcel="{ invalid: context.hasEmailError() }"
    placeholder="you@example.com"
    @commit-value="context.onCommitEmail({ payload: $event })"
  />
</template>
```

## Conflicts — named, not resolved. `D:\ORT\rules\` wins.

1. **Logic inside a template object literal.** The skill's example writes
   `:parcel="{ invalid: context.hasEmailError() }"` — a method call as an
   object-literal property value, inline in the template. `rules/javascript-style.md`
   ("No logic in object-literal property values": never a method call in a property
   value) and the project's no-JS-logic-in-a-template constraint both forbid it.
   **Rule wins.**
2. **Un-chopped object-literal argument.** The skill writes
   `@commit-value="context.onCommitEmail({ payload: $event })"` on one line.
   `rules/javascript-style.md` requires an object literal passed as an argument to be
   multi-line, one property per line, trailing comma, even for a single property.
   **Rule wins.**
3. **`v-model` exception.** The skill sanctions `v-model:value` on furo-vue
   components as an explicit, scoped exception to the project's no-`v-model` rule.
   `D:\ORT\rules\` does not mention `v-model`, so nothing in the rules contradicts it
   — recorded so the exception is not mistaken for a violation.
4. **Method-name inconsistency inside the skill.** Its Rules section prescribes
   `on{Field}Commit` while its example uses `onCommitEmail`. `rules/naming.md`
   requires a method name to begin with a verb, which favours `onCommitEmail`.
   **Rule wins.**
5. **Manifest overstates `FuroPasswordField`.** `components.json` summarises it as
   "Password input atom with **reveal toggle**", but the installed component renders a
   bare `<input type="password">` with no toggle and `"slots": []`. The source, not
   the summary, is accurate — do not plan a reveal button on this component.
6. **`aria-invalid` under-documented.** `SKILL.md` attributes `aria-invalid` only to
   `FuroNumberField`'s `invalid` field; the installed source binds `:aria-invalid` on
   the text, email, and password atoms too.
