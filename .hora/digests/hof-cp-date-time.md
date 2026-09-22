# hof-cp-date-time
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-cp-date-time/ -->

**Read the source above whenever this leaves a question open.**

Three molecules: `FuroDatePicker` (segments + popover calendar, wire value `YYYY-MM-DD`),
`FuroTimeField` (inline segmented time, wire value `HH:MM:SS`), `FuroDateTimePicker` (both in one
popover, wire value `YYYY-MM-DDTHH:MM:SS`). Pick by what the data **is** — date only, time only, or
one timestamp — not by how it should look. `#expense-entry`'s `spentOn` is a date, so
**`FuroDatePicker`**. Verified against the installed `@openreachtech/furo-vue@1.3.2`; facts marked
**[source]** were read from `node_modules/@openreachtech/furo-vue/lib/components/molecules/FuroDatePicker/`
and `node_modules/reka-ui/dist/` and are **not** in the skill.

## Import and identity

- Layer: molecule (all three).
- `import { FuroDatePicker, FuroTimeField, FuroDateTimePicker } from '@openreachtech/furo-vue'`
- **Never import the underlying headless primitive** — only the public exports.
- Manifest: `node_modules/@openreachtech/furo-vue/public/furo-vue/components.json` →
  `components[].name === 'FuroDatePicker' | 'FuroTimeField' | 'FuroDateTimePicker'`.
- Pass `parcel` + `v-model:value` — the documented exception to the project's no-`v-model` rule,
  which applies only to custom in-project components, not to `furo-vue` components.
- No date **range** component exists among the three; check the manifest before hand-rolling two
  pickers.
- Needs a label / error message → wrap in `FuroControlBlock` (`hof-cp-control-block`); never
  hand-roll a `<label>`.

## Props

| Component | Prop | Type | Default | Notes |
| --- | --- | --- | --- | --- |
| `FuroDatePicker` | `parcel` | `DatePickerParcel \| null` | `null` | Furo-only keys are stripped before passthrough to the primitive. |
| `FuroDatePicker` | `triggerParcel` | `Record<string, unknown> \| null` | `null` | `v-bind`-ed onto the date field / trigger. |
| `FuroDatePicker` | `contentParcel` | `Record<string, unknown> \| null` | `null` | `v-bind`-ed onto the popover content. |
| `FuroTimeField` | `parcel` | `TimeFieldParcel \| null` | `null` | Only prop. |
| `FuroDateTimePicker` | `parcel` | `DateTimePickerParcel \| null` | `null` | Only prop; molecule-only keys derive the child parcels. |

## `FuroDatePicker` `parcel` fields

| Field | Type | Default | Notes |
| --- | --- | --- | --- |
| `value` | `string \| null` | — | Selected date as `YYYY-MM-DD`, or null. |
| `invalid` | `boolean` | `false` | Error chrome plus `aria-invalid` on the field. |
| `placeholder` | `string \| null` | — | Placeholder text for the date field. |
| `disabled` | `boolean` | `false` | Disables all interaction. |
| `minValue` | `string \| null` | — | Minimum date (`YYYY-MM-DD`). **Marks, does not block — see below.** |
| `maxValue` | `string \| null` | — | Maximum date (`YYYY-MM-DD`). **Marks, does not block — see below.** |
| `displayFormat` | `string \| null` | — | Reserved for v1.1 localized display; not yet applied. |
| `timezone` | `string \| null` | `null` | IANA id. Interprets `Date` inputs before normalization; no effect on `YYYY-MM-DD` strings. |
| `closeOnSelect` | `boolean \| null` | `true` | Popover closes on selection. |
| `locale` | `string \| null` | browser locale | Drives segment order; falls back to `'en'` outside the browser. |
| `preventDeselect` | `boolean \| null` | `true` | Re-selecting the selected date keeps it instead of clearing. |
| `todayButtonText` | `string \| null` | `'Today'` | Footer Today button label; `null` hides it. |
| `clearButtonText` | `string \| null` | `'Clear'` | Footer Clear button label; `null` hides it. |

## `maxValue` marks an out-of-range date, it does not block one

**Divergence, confirmed.** The skill says dates past `maxValue` are "non-interactive". That is true
of the **calendar grid** and false of the **typed segments**.

| Surface | Out-of-range date |
| --- | --- |
| calendar grid cell | genuinely disabled — `reka-ui/dist/Calendar/useCalendar.js:112-113` returns `isDateDisabled` for anything after `maxValue` / before `minValue`; the cell trigger then sets `aria-disabled` / `data-disabled` and ignores the click (`Calendar/CalendarCellTrigger.js:53,67,125-128`) |
| typed segments | **accepted** — `reka-ui/dist/DateField/DateFieldRoot.js:146` computes `isInvalid` from `minValue` / `maxValue`, and that computed is consumed **only** at lines 247 and 254: a `data-invalid` attribute and a slot prop. `modelValue` still updates |
| the emit | still fires — `FuroDatePickerContext.onChangeValue()` (line 773) normalizes and emits `change-value`, `update:value`, `commit-value` with the out-of-range date, performing **no range check** |

**Consequence for `#expense-entry`:** spec §11 requires "an expense dated after today is refused".
`maxValue: <today>` is a hint, not a guard. The page's Context must check `spentOn` before sending,
and the backend refuses independently.

Related **[source]**: the footer **Today** button *does* clamp — `onSelectToday()` runs
`clampDateToBounds()` first. So does the initial calendar placeholder. Only typed input escapes.

## `FuroTimeField` `parcel` fields

| Field | Type | Default | Notes |
| --- | --- | --- | --- |
| `value` | `string \| null` | — | `HH:MM:SS` (or `HH:MM`), or null. Date / Time accepted at the boundary. |
| `invalid` | `boolean` | — | Error chrome and `aria-invalid`. |
| `placeholder` | `string \| null` | — | Placeholder text. |
| `disabled` | `boolean` | — | Disables all interaction. |
| `readonly` | `boolean` | — | Segments focusable, not editable. |
| `minValue` / `maxValue` | `string \| null` | — | `HH:MM:SS`. **Out-of-range values are flagged, not clamped** — the same marking-only behavior as the date picker. |
| `timeFormat` | `number \| null` | locale default | `12` or `24`. |
| `granularity` | `'hour' \| 'minute' \| 'second' \| null` | `'minute'` | Visible segments. |
| `step` | `number \| null` | `1` | Arrow / button step; unit follows granularity. Forwarded as native `step`. |
| `stepSnapping` | `boolean \| null` | — | Forwarded only when explicitly set. |
| `timezone` | `string \| null` | `null` | Normalizes Date inputs to wall-clock parts; never forwarded to the primitive. |

## `FuroDateTimePicker` `parcel` fields

| Field | Type | Default | Notes |
| --- | --- | --- | --- |
| `value` | `string \| null` | — | `YYYY-MM-DDTHH:MM:SS`, split and recombined on change. |
| `invalid` | `boolean` | — | Error chrome on the shell; forwarded to both atoms. |
| `placeholder` | `string \| null` | — | Date field placeholder. |
| `disabled` | `boolean` | — | Disables all interaction. |
| `minValue` / `maxValue` | `string \| null` | — | `YYYY-MM-DDTHH:MM:SS`. |
| `timezone` | `string \| null` | `null` | Forwarded to both atoms. |
| `granularity` | `'hour' \| 'minute' \| 'second' \| null` | — | Visible time segments. Wire format is always second precision. |
| `step` | `number \| null` | — | Time field increment / decrement step. |
| `timeFormat` | `number \| null` | locale default | `12` or `24`. |

## Events — identical trio on all three

| Event | Payload | Fires when |
| --- | --- | --- |
| `change-value` | `DatePickerEmitPayload` / `TimeFieldEmitPayload` / `DateTimePickerEmitPayload` | The user selects / edits. |
| `commit-value` | same payload | **Same gesture** — selection is the commit point. |
| `update:value` | `string \| null` | ISO string for `v-model:value`. |

**[source]** `FuroDatePickerContext.onChangeValue()` emits all three from one call, in the order
`change-value` → `update:value` → `commit-value`. `FuroDateTimePicker` sets `12:00:00` when a date
is picked and no time exists yet.

`DatePickerEmitPayload` **[source]** (`DatePickerEmitPayload.js`) extends `BaseEmitPayload` and
exposes exactly `get eventDetail`, `get controlElement`, `get value` (`string | null`, ISO
`YYYY-MM-DD`), `hasEmptyValue()`, `hasValue()`.

## Slots

### FuroDatePicker

| Slot | Description |
| --- | --- |
| `trigger-icon` | Replaces the calendar icon in the trigger (default `ph:calendar`). |
| `prev-icon` | Left arrow in the calendar header (default `ph:caret-left`). |
| `next-icon` | Right arrow in the calendar header (default `ph:caret-right`). |
| `field-suffix` | Content inside the trigger field, after the segments and before the trigger icon. |
| `content-footer` | Content inside the popover, below the calendar and above the Today / Clear buttons. |

`FuroTimeField` and `FuroDateTimePicker`: **no slots.**

**[source]** The calendar header is not only prev/next: it renders two native `<select>`s for month
and year. `extractCalendarYears()` spans `minValue.year` … `maxValue.year`, defaulting to
`currentYear - 100` … `currentYear + 50` when the bounds are absent. Both selects move the **view**
only; neither changes the value.

## Usage

Written to `D:\ORT\rules\` house style and §8 rule 12 — the skill's own example violates both (see
Conflicts):

```vue
<template>
  <FuroControlBlock :parcel="context.spentOnControlBlockParcel">
    <FuroDatePicker
      id="spent-on"
      v-model:value="context.spentOn"
      :parcel="context.spentOnParcel"
      @commit-value="context.onCommitSpentOn($event)"
    />
  </FuroControlBlock>
</template>
```

Both parcels are getters on the page Context. The commit / validation logic is a named Context
method (`onCommitSpentOn`), never inline in the template.

## Label association (WCAG 2.2 AA — checkpoint 18)

**[source]** An `id` passed as a plain attribute **does** reach a real element: `extractRootAttributes()`
spreads `this.attrs` onto reka's `DatePickerRoot`, which hands its `id` to `DatePickerField`
(`reka-ui/dist/DatePicker/DatePickerField.js:16`), which puts it on a **visually-hidden `<input>`**
that forwards focus to the first segment (`DateField/DateFieldRoot.js:255-261`). So
`FuroControlBlock` `controlId: 'spent-on'` + `id="spent-on"` on the picker gives a working
`<label for>`, and clicking the label lands focus in the segments.

The visible field itself is a `role="group"` with no accessible name of its own
(`DateFieldRoot.js:243`), and its focus style is `outline: none` + `border-color: var(--color-ring)`
— a token this project does not declare, so there is no visible focus indicator today (same gap as
`hof-cp-button.md`).

## Appearance and tokens

**[source]** `<style>` in `@layer furo`; root classes `.unit-form-control.date-picker` and
`.unit-date-picker-content`. Colours and sizes come from `--color-input`, `--color-foreground`,
`--color-ring`, `--size-input-height`, `--size-border-radius-medium`, `--size-thinnest`,
`--font-body-medium`, `--font-size-small`, `--transition-timer` — **none declared in this
application**. Same standing gap recorded in `hof-cp-button.md` and `hof-cp-control-block.md`.

## Conflicts — named, not resolved

1. **`maxValue` does not refuse a typed future date** (section above). The skill's wording
   ("non-interactive") would lead an implementer to rely on the component for spec §11. **The library
   wins:** the guard is the Context's, and the backend's.
2. **The skill's control-block example passes the wrong field.** It writes
   `<FuroControlBlock :parcel="{ label: 'Due date', error: context.dueDateError }">`. There is **no
   `error` field** on `ControlBlockParcel`; the real field is **`errorMessages: Array<string>`**
   (verified in `lib/components/molecules/FuroControlBlock/` and recorded in
   `.hora/digests/hof-cp-control-block.md`). An `error` key is silently ignored and no message
   renders. **The library wins — use `errorMessages`.**
3. **Inline `:parcel="{ … }"` with method calls in the template.** The skill writes
   `:parcel="{ invalid: context.hasDueDateError(), minValue: context.todayIsoDate }"`. Barred by §8
   rule 12 and by `D:\ORT\rules\javascript-style.md`; recorded once for all five component digests in
   `.hora/digests/hof-cp-table.md` (Conflicts 2). **The project wins.**
4. **`preventDeselect: true` by default.** A user cannot clear `spentOn` by re-clicking the selected
   day; the only clear path is the footer **Clear** button, which the parcel can hide
   (`clearButtonText: null`). Decide deliberately whether the field is clearable.
