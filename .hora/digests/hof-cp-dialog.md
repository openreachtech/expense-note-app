# hof-cp-dialog
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-cp-dialog/ -->

**Read the source above whenever this leaves a question open.**

Three organism overlays. `FuroDialog` is the general modal (header / body / footer + close button).
`FuroAlertDialog` is the purpose-built confirmation prompt — **no close button, outside-click never
dismisses, built-in confirm / cancel buttons with a `destructive` tone**; it is what
`#expense-entry`'s removal confirmation uses. `FuroDrawer` is an edge-anchored sliding panel. All
three share `title` / `description` / `busy` / `initialFocus` / `closeOnEscape` in their parcel, and
all three use `parcel` + `v-model:open` — the documented exception to the project's no-`v-model`
rule, which applies only to custom in-project components, not to `furo-vue` components. Verified
against the installed `@openreachtech/furo-vue@1.3.2`; facts marked **[source]** were read from
`node_modules/@openreachtech/furo-vue/lib/components/organisms/FuroAlertDialog/` (and the sibling
`FuroDialog/` / `FuroDrawer/` contexts) and are **not** in the skill.

## Import and identity

- Layer: organism (all three).
- `import { FuroDialog, FuroAlertDialog, FuroDrawer } from '@openreachtech/furo-vue'`
- **Never import the underlying headless primitives** (`reka-ui`'s `DialogRoot`, …) — only the
  public exports.
- Manifest: `node_modules/@openreachtech/furo-vue/public/furo-vue/components.json` →
  `components[].name === 'FuroDialog' | 'FuroAlertDialog' | 'FuroDrawer'`.
- None of the three follows the form-control (`parcel.value`) contract.
- All three take a single `parcel` prop — no other props.

### Which one

| Need | Use |
| --- | --- |
| Explicit yes/no decision, destructive or not | `FuroAlertDialog` — do **not** hand-roll `FuroDialog` + footer buttons; the alert dialog already encodes "block until decided" |
| Form, detail view, freeform modal content | `FuroDialog` |
| Persistent side panel, mobile sheet, content too long for a centered modal | `FuroDrawer` |
| Small click-anchored panel that must not block the page | `hof-cp-popover` (`FuroPopover`), not a non-modal `FuroDialog` |
| Hover / focus hint text | `hof-cp-popover` (`FuroTooltip`) |
| Menu of discrete actions | `hof-cp-dropdown-menu` |
| Transient feedback, not a decision | `hof-cp-toast` |
| Inline expand / collapse in the page flow | `hof-cp-collapsible` |

## `FuroAlertDialog` `parcel` fields

| Field | Type | Default | Notes |
| --- | --- | --- | --- |
| `open` | `boolean \| null` | `null` | Controlled open state. `null` (omitted) = the dialog manages its own. |
| `disabled` | `boolean` | `false` | Disables the trigger. |
| `title` | `string \| null` | `null` | Heading text. Supply it (or the `title` slot) for accessibility. |
| `description` | `string \| null` | `null` | Sub-heading below the title. Rendered only when present. |
| `confirmText` | `string` | `'Confirm'` | Built-in confirm button label. |
| `cancelText` | `string` | `'Cancel'` | Built-in cancel button label. |
| `tone` | `'default' \| 'destructive'` | `'default'` | `destructive` renders the confirm button with the destructive variant. |
| `closeOnEscape` | `boolean` | `true` | `false` ignores the Escape key. |
| `size` | `'sm' \| 'default' \| 'lg'` | `'default'` | Content width variant. |
| `busy` | `boolean` | `false` | Dismissal blocked; both buttons disabled; confirm shows the loading state. |
| `initialFocus` | `string \| null` | `null` | CSS selector scoped to the content. When unset, the cancel button is focused. |

**[source]** `isOpenControlled()` tests `(parcel.open ?? null) !== null` — so passing
`open: false` explicitly puts the dialog in **controlled** mode, and only omitting the key (or
passing `null`) leaves it uncontrolled.

## `FuroDialog` / `FuroDrawer` `parcel` fields

Both add to the shared five **[source]** (verified against each context's `get parcel`):

- `FuroDialog`: `disabled` (`false`), `modal` (`true`; `false` = no scrim, page stays interactive),
  `closeText` (`'Close'`), `closeOnOutsideClick` (`true`), `size`
  (`'sm' | 'default' | 'lg' | 'fullscreen'`, `'default'`).
- `FuroDrawer`: `disabled`, `modal` (`true | false | 'trap-focus'`; `'trap-focus'` traps focus while
  the page stays interactive), `closeText`, `closeOnOutsideClick`, `side`
  (`'left' | 'right' | 'top' | 'bottom'`, default `'right'`), `size` (as dialog — cross-axis extent),
  `snapPoints` (`Array<number | string> | null`: fractions 0–1, pixels > 1, or `'148px'` / `'30rem'`),
  `activeSnapPoint` (`null`; when set the parent owns it), `sequentialSnap` (`false`),
  `swipeToOpen` (`false`), `dragHandle` (`false`).

## Events

| Component | Event | Payload | Fires when |
| --- | --- | --- | --- |
| `FuroAlertDialog` | `confirm` | — (no argument) | The confirm action was chosen. |
| `FuroAlertDialog` | `cancel` | — (no argument) | The cancel action was chosen — button **or Escape**. |
| `FuroAlertDialog` | `update:open` | `boolean` | Open state changed — enables `v-model:open`. |
| `FuroAlertDialog` | `open:change` | `{ open: boolean }` | Open state changed (semantic form). |
| `FuroDialog` | `update:open` / `open:change` | `boolean` / `{ open }` | Open state changed. |
| `FuroDrawer` | `update:open` / `open:change` | `boolean` / `{ open }` | Open state changed. |
| `FuroDrawer` | `update:activeSnapPoint` | `number \| string \| null` | Enables `v-model:activeSnapPoint`. |
| `FuroDrawer` | `snap:change` | `{ snapPoint }` | Active snap point changed. |

## Confirm / cancel / Escape — the exact sequence **[source]**

`FuroAlertDialogContext.js`, lines 385-510:

| Gesture | What happens |
| --- | --- |
| confirm button | `onConfirm()` → emits `confirm` → `closeInternal()` |
| cancel button | `onCancel()` → emits `cancel` → `closeInternal()` |
| Escape | `onEscapeKeyDown()` always `preventDefault()`s, then returns early when `busy` or `closeOnEscape === false`; otherwise calls `onCancel()` — **Escape emits `cancel`, it is not a separate dismissal** |
| outside click | `onInteractOutside()` unconditionally `preventDefault()`s — **never dismisses, at any setting** |
| `closeInternal()` | returns immediately when `busy`; otherwise clears the internal ref (uncontrolled only) and emits `update:open` `false` then `open:change` `{ open: false }` |

**Consequence to design around:** `busy` is read from the `parcel` **prop**. `onConfirm()` emits
`confirm` and calls `closeInternal()` in the same synchronous tick, before a flag the handler set
has propagated through props — so a plain "set `isRemoving = true` inside `onConfirmRemove`" does
**not** keep the dialog open; `update:open false` fires and the dialog closes. If it must stay open
for the duration of the mutation, `busy` has to be true in the parcel *before* the confirm.

`generateConfirmButtonParcel()` returns `{ variant: tone === 'destructive' ? 'destructive' : 'default',
loading: busy, disabled: busy }`; `generateCancelButtonParcel()` returns
`{ variant: 'outline', disabled: busy }`. The buttons are real `FuroButton`s — see
`.hora/digests/hof-cp-button.md` for what `loading` + `disabled` do together.

## Slots

### FuroAlertDialog

| Slot | Scoped props | Description |
| --- | --- | --- |
| `trigger` | — | Element that opens the dialog, rendered **as-child**. Rendered only when the slot is provided **[source]** (`Object.hasOwn($slots, 'trigger')`); omit it when driving `open` from the Context. |
| `title` | — | Overrides the heading (falls back to `parcel.title`). |
| `description` | — | Overrides the description (falls back to `parcel.description`, and the whole element is dropped when there is none). |
| `default` | — | Optional body between heading and footer; the wrapper `<div class="body">` renders only when the slot is provided. |
| `footer` | `{ confirm, cancel }` | Replaces the built-in confirm / cancel buttons. Both slot props are zero-argument functions. |

### FuroDialog / FuroDrawer

`trigger`, `title`, `description`, `default` (scoped `{ close }`), `footer` (scoped `{ close }`),
`close` (content of the built-in close button, defaults to ✕).

## Usage

Written to `D:\ORT\rules\` house style and §8 rule 12 — the skill's own example violates both
(Conflicts 1):

```vue
<template>
  <FuroAlertDialog
    v-model:open="context.isRemoveConfirmOpen"
    :parcel="context.removeConfirmParcel"
    @confirm="context.onConfirmRemove()"
    @cancel="context.onCancelRemove()"
  />
</template>
```

`removeConfirmParcel` is a getter on the page Context carrying `title`, `description`,
`tone: 'destructive'`, `confirmText`, `cancelText` and `busy`. **Always handle both `confirm` and
`cancel`**, and run the removal mutation inside the `confirm` handler on the Context — never inline
in the template.

## Accessibility (project target: WCAG 2.2 AA — checkpoint 18)

**[source]** The content is portalled and rendered by reka's `AlertDialog` primitives, so the
`alertdialog` role, the modal focus trap and the `aria-labelledby` / `aria-describedby` wiring to
`AlertDialogTitle` / `AlertDialogDescription` come from the library. What this project still owns:

- **`title` must be supplied** — the heading element renders unconditionally, so an absent `title`
  (and no `title` slot) leaves an empty accessible name.
- **`description` is conditional** — no description means no `aria-describedby` target.
- Focus on open is the **cancel** button unless `initialFocus` names a selector inside the content;
  `onOpenAutoFocus()` `preventDefault()`s and focuses that element only when the selector matches.
- The overlay and content read `--furo-dialog-overlay-background`,
  `--value-z-index-layer-overlay`, `--size-space-medium` and friends inside `@layer furo` —
  **none declared in this application**, the same standing gap recorded in `hof-cp-button.md`. The
  z-index token being unset means the overlay has no stacking guarantee.

Root classes are `.unit-alert-dialog` (+ `disabled`), `.unit-alert-dialog-overlay` and
`.unit-alert-dialog-content` (+ the size name when not `default`, + `busy`).

## Conflicts — named, not resolved

1. **Inline `:parcel="{ … }"` in the skill's examples.** All three examples write the parcel as a
   template object literal. Barred by §8 rule 12 of
   `expense-note-frontend-staff/ai/contexts/uiux-context.md` and by `D:\ORT\rules\javascript-style.md`;
   recorded once for all five component digests in `.hora/digests/hof-cp-table.md` (Conflicts 2).
   **The project wins.**
2. **`sm` / `lg` size names vs the no-abbreviation rule.** `05-frontend.md` 6-14 and §8 rule 16 forbid
   abbreviations, but `FuroAlertDialogSize` is the closed library vocabulary
   `'sm' | 'default' | 'lg'`. Library values are passed verbatim; the rule still governs every name
   this project writes.
3. **The skill's "`cancel` fires on button or Escape" is right, but understates it** — Escape is
   routed *through* `onCancel()`, so a cancel handler that does real work runs on Escape too, and
   `closeOnEscape: false` suppresses the whole path. Read the sequence table above before relying on
   either event.
