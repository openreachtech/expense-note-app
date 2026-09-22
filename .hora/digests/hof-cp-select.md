# hof-cp-select
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-cp-select/ -->

**Read the source above whenever this leaves a question open.**

Two molecules for choosing from a list. `FuroSelect` is a plain dropdown over a fixed `options`
array — no text filtering. `FuroAutocompleteField` is a combobox: the user types to filter
(client-side via `localCompare`, or against a parent-owned list when `localCompare: false`), with a
debounced `search-keyword`, optional select-all and optional "create new option". Short known list
(status, role, **expense category**) → `FuroSelect`. Long, searched or fetched → `FuroAutocompleteField`.
Verified against the installed `@openreachtech/furo-vue@1.3.2`; facts marked **[source]** were read
from `node_modules/@openreachtech/furo-vue/lib/components/molecules/FuroSelect/` and
`node_modules/reka-ui/dist/` and are **not** in the skill.

## Import and identity

- Layer: molecule (both).
- `import { FuroSelect } from '@openreachtech/furo-vue'` /
  `import { FuroAutocompleteField } from '@openreachtech/furo-vue'`
- **Never import the underlying headless primitive** — only the public exports.
- Manifest: `node_modules/@openreachtech/furo-vue/public/furo-vue/components.json` →
  `components[].name === 'FuroSelect' | 'FuroAutocompleteField'`.
- Pass `parcel` + `v-model:value` — the documented exception to the project's no-`v-model` rule,
  which applies only to custom in-project components, not to `furo-vue` components.

### When NOT to use

| Need | Use instead |
| --- | --- |
| Free-form text entry | `hof-cp-text-field` |
| A menu of actions to trigger, not a value to store | `hof-cp-dropdown-menu` |
| A boolean choice | `hof-cp-checkbox-toggle` |
| Label / error message around it | wrap in `FuroControlBlock` (`hof-cp-control-block`) — don't hand-roll a `<label>` |
| Toolbar-style toggle buttons, not a value-bearing field | `hof-cp-toggle-group` |

## Props

| Component | Prop | Type | Default | Notes |
| --- | --- | --- | --- | --- |
| `FuroSelect` | `parcel` | `object \| null` | `null` | Root behavior and Furo fields. |
| `FuroSelect` | `triggerParcel` | `object \| null` | `null` | `v-bind`-ed onto the trigger button. |
| `FuroSelect` | `portalParcel` | `object \| null` | `null` | **[source] Declared but never consumed** — the template renders no portal. See Conflicts 2. |
| `FuroSelect` | `contentParcel` | `object \| null` | `null` | `v-bind`-ed onto the content (`side-offset` defaults to 4). |
| `FuroAutocompleteField` | `parcel` | `FuroAutocompleteFieldParcel \| null` | `null` | Only prop; HTML / form attributes pass through to the root. |

## `FuroSelect` `parcel` fields

| Field | Type | Default | Notes |
| --- | --- | --- | --- |
| `value` | `string \| number \| object \| null` | `null` | Controlled value; an array when `multiple`. Objects supported. |
| `defaultValue` | same as `value` | — | Uncontrolled initial value. |
| `options` | `Array<FuroSelectOption>` | `[]` | Unified list of groups and leaf rows. |
| `invalid` | `boolean` | `false` | Invalid visual state plus `aria-invalid` on the trigger. |
| `loading` | `boolean` | `false` | Spinner on the trigger; suppresses the empty row. |
| `placeholder` | `string` | `''` | Placeholder in the value display. |
| `textDirection` | `'toRight' \| 'toLeft'` | `'toRight'` | Maps to the root `dir` (`ltr` / `rtl`). |
| `multiple` | `boolean` | `false` | Multi-select on the root. |
| `disabled` | `boolean` | `false` | Disables the control; may also be a fallthrough attribute. |
| `optionLabel` | `string \| ((option) => string)` | — | **[source]** Key name or function selecting the label. Falls back to `option.label`. **Not in the skill.** |
| `optionValue` | `string \| ((option) => value)` | — | **[source]** Key name or function selecting the model value. Falls back to `option.value`. **Not in the skill.** |

Any other parcel key is spread onto the root primitive; `optionLabel` / `optionValue` are stripped
first **[source]** (`FuroSelectContext.extractRootAttributes()`, line 340).

### `FuroSelectOption` **[source]** (`SelectEmitPayload.js` typedef, line ~133)

```js
{
  label: string | null,
  value: FuroSelectValue,
  disabled: boolean,                   // optional
  separator: boolean,                  // optional — renders a separator after this row
  options: Array<FuroSelectOption>,    // optional — presence makes this entry a GROUP
}
```

A group's own `disabled` disables every child. An entry with a nested `options` array renders as a
`<SelectGroup>` with a label; everything else is a leaf option. Extra keys are allowed and reachable
through `optionLabel` / `optionValue`.

## `FuroAutocompleteField` `parcel` fields

| Field | Type | Default | Notes |
| --- | --- | --- | --- |
| `value` | `option \| value \| Array \| null` | — | Controlled selection; fallthrough `value` is the fallback. |
| `options` | `Array<option \| value>` | `[]` | Groups and leaves. |
| `placeholder` | `string` | `'Select an option'` | Input placeholder. |
| `loading` | `boolean` | `false` | Input spinner; filtered list returns empty and the empty state is suppressed. |
| `invalid` | `boolean` | `false` | Root invalid class. |
| `disabled` | `boolean` | `false` | Disables the combobox. |
| `textDirection` | `'toRight' \| 'toLeft'` | `'toRight'` | Maps to the primitive's `dir`. |
| `multiple` | `boolean` | `false` | Multi-select; chips in the anchor. |
| `showClear` | `boolean` | `true` | Clear control once a value is selected. |
| `localCompare` | `boolean` | `true` | `true` filters on the client; `false` uses the parent-owned options. |
| `debounceMilliseconds` | `number` | `300` | Debounce for `search-keyword` (TimerClerk). |
| `openOnClick` | `boolean` | `true` | Passed through to the root. |
| `showSelectAll` | `boolean` | `false` | Multi only; sentinel select-all row. |
| `selectAllText` | `string` | `'Select All'` | Select-all row label. |
| `emptyText` | `string` | `'No options.'` | Default empty-slot copy. |
| `creatable` | `boolean` | `false` | Create row when the trimmed keyword has no exact label / value match. |
| `createOptionText` | `string` | `'Create "{keyword}"'` | Create-row template; `{keyword}` is replaced with the trimmed keyword. |
| `searchKeyword` | `string` | — | Partial: parent → internal input via watch; there is **no** `update:search-keyword`. |
| `open` | `boolean` | — | **Not implemented** — the template's open state uses the internal popover ref. |

## Events

### FuroSelect

| Event | Payload | Fires when |
| --- | --- | --- |
| `change-value` | `SelectEmitPayload` | On selection. |
| `commit-value` | `SelectEmitPayload` | **Same gesture** — selection is the commit point. |
| `update:value` | `string \| number \| object \| null \| Array` | For `v-model:value`. |

**[source]** All three fire from one `onChangeValue()` (line 517), in the order `change-value` →
`commit-value` → `update:value`. `update:value` is **not** "the first selected value" as the skill
says: `extractUpdateValue()` returns the **whole array** of model values in multiple mode, and the
single resolved `optionValue` otherwise.

`SelectEmitPayload` **[source]** exposes `get eventDetail`, `get controlElement`, `get value`,
`get values`, `get selectedOption`, `get selectedIndex`, `get selectedOptions`,
`get selectedIndices`, `hasEmptyValue()`, `hasSelectedValue()`, `matchesOption({ … })`. The raw
detail also carries `selectedGroupIndex` / `selectedGroupIndices`.

### FuroAutocompleteField

| Event | Payload | Fires when |
| --- | --- | --- |
| `change-value` | `AutocompleteEmitPayload` | Select option, select-all, clear, or chip remove. |
| `update:value` | option / array / null | Same gesture; for `v-model:value`. |
| `search-keyword` | `string` | After the debounce while the popover is open; also `''` on clear. |
| `create-option` | `AutocompleteEmitPayload` | The create row was chosen; read the typed text via `extractTrimmedSearchKeyword()`. |

**No `commit-value`** — `change-value` already fires at the selection point.

## Slots

### FuroSelect **[source]** — the skill lists 4 of 12

| Slot | Scoped props | Description |
| --- | --- | --- |
| `trigger-content` | — | Whole trigger interior (value display + icon). |
| `trigger-text` | — | Text inside the value display. |
| `trigger-icon` | — | Caret icon (default `ph:caret-down`). |
| `trigger-loading-icon` | — | Replaces the caret while `loading` (default `ph:circle-notch`). |
| `content-before` / `content-after` | — | Content above / below the scrollable list. |
| `scroll-up-button` / `scroll-down-button` | — | Scroll affordances (default `ph:caret-up` / `ph:caret-down`). |
| `group` | `{ group, groupIndex }` | Group label and its children. |
| `group-label` | `{ group, groupIndex }` | Group header text only. |
| `option` | `{ option, optionIndex, group?, groupIndex? }` | Whole option row (text + indicator). |
| `option-text` | `{ option, optionIndex }` | Option label text only. |
| `option-indicator` | `{ option, optionIndex }` | Selected indicator (default `ph:check`). |
| `separator` | `{ option, optionIndex }` | Rendered after an option whose `separator` is true. |
| `empty` | `{ loading }` | Shown when `options` is empty **and** not loading. Default text `No options`. |

### FuroAutocompleteField

| Slot | Scoped props | Description |
| --- | --- | --- |
| `trigger-left` | — | Leading anchor content. |
| `trigger-right` | — | Caret icon in the trigger. |
| `select-all` | — | Select-all label (scoped checked / indeterminate / disabled are **not** wired). |
| `group` | `{ group, groupIndex }` | Group and children. |
| `group-label` | `{ group, groupIndex }` | Group header text. |
| `option` | `{ option, optionIndex, group?, groupIndex? }` | Row and indicator. |
| `option-text` | `{ option, optionIndex }` | Label text. |
| `empty` | `{ searchKeyword, loading }` | Filtered list empty and not loading; defaults to `emptyText`. |
| `loading` | `{ searchKeyword }` | Spinner in the list. |
| `create-option` | `{ searchKeyword }` | `createOptionText` with `{keyword}` replaced; only when `creatable` and no exact match. |

## Usage

Written to `D:\ORT\rules\` house style and §8 rule 12 — the skill's own examples violate both (see
Conflicts 1):

```vue
<template>
  <FuroControlBlock :parcel="context.expenseCategoryControlBlockParcel">
    <FuroSelect
      v-model:value="context.expenseCategoryId"
      :parcel="context.expenseCategoryParcel"
      :trigger-parcel="context.expenseCategoryTriggerParcel"
    />
  </FuroControlBlock>
</template>
```

Every parcel is a getter on the page Context. Option-list fetching, filtering (when
`localCompare: false`) and create-option persistence are named Context methods
(`onSearchExpenseCategory`, `onCreateExpenseCategory`), never inline in the template.

## Label association — an `id` on `<FuroSelect>` reaches nothing

**[source]** `FuroSelect` puts `$attrs.class` / `$attrs.style` on its wrapper `<div>` and spreads the
rest of `this.attrs` onto reka's `SelectRoot`. `SelectRoot` has no `id` prop and renders
`PopperRoot`, which renders **only its slot** (`reka-ui/dist/Popper/PopperRoot.js:16`) — a fragment.
A fallthrough `id` therefore lands on no DOM element, and a `FuroControlBlock` `controlId` pointing
at it produces a `<label for>` that resolves to nothing.

**Put the id on the trigger instead:** `:trigger-parcel="{ id: 'expense-category' }"` (built in the
Context) — `triggerParcel` is `v-bind`-ed onto `SelectTrigger`, which renders a real `<button>`.
The trigger already carries `aria-invalid`, `aria-expanded`, `aria-controls`, `aria-required`
(`reka-ui/dist/Select/SelectTrigger.js:85-94`) but **no accessible name of its own**, so this is also
the only way the field gets a name for the checkpoint 18 audit.

## Appearance and tokens

**[source]** `<style>` in `@layer furo`; root classes `.unit-form-control.select` (trigger) and
`.unit-select-content` (popover). Colours and sizes come from `--color-input`, `--color-foreground`,
`--size-input-height`, `--size-border-radius-medium`, `--size-thinnest`, `--size-space-2x-small`,
`--font-body-medium` — **none declared in this application**. Same standing gap recorded in
`hof-cp-button.md`.

## Conflicts — named, not resolved

1. **Inline `:parcel="{ … }"` with a method call in a property value.** Both of the skill's examples
   write the parcel as a template object literal, one of them containing
   `invalid: context.hasStatusError()`. Barred by §8 rule 12 of
   `expense-note-frontend-staff/ai/contexts/uiux-context.md` and by `D:\ORT\rules\javascript-style.md`
   ("No logic in object-literal property values"); recorded once for all five component digests in
   `.hora/digests/hof-cp-table.md` (Conflicts 2). **The project wins.**
2. **`portalParcel` is a dead prop.** `FuroSelect.vue` declares it (line 53) and
   `FuroSelectContext` exposes a `get portalParcel` (line 171), but the
   template renders `SelectContent` directly under `SelectRoot` with **no `SelectPortal`**, so
   nothing consumes it. Do not plan on teleporting the list; **the template wins.**
3. **`update:value` in multiple mode.** The skill calls it "first selected value for multi-select";
   the source returns the full array. **The source wins.**
4. **The skill lists 4 of `FuroSelect`'s 12 slots.** Use the table above.
5. **The `empty` row is a real `SelectOption`** with `value: null` and `@select.stop.prevent`
   **[source]** — it appears in the listbox and is reachable by keyboard, which an audit may flag.
   It renders only when `options` is empty and `loading` is false.
