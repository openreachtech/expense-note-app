# hof-cp-empty-state
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-cp-empty-state/ -->

**Read the source above whenever this leaves a question open.**

`FuroEmptyState` / `FuroErrorState` — two presentational molecules with the **same shape** (title,
description, `icon` slot, `action` slot) signalling opposite conditions. Empty = the region loaded
and legitimately has nothing to show (action slot is typically a create button). Error = the
fetch/request failed (action slot is typically retry); its root carries `role="alert"`. Neither
fetches, and neither decides which applies — the page Context inspects the result and chooses.
Verified against the installed `@openreachtech/furo-vue@1.3.2`; facts marked **[source]** were read
from `node_modules/@openreachtech/furo-vue/lib/components/molecules/FuroEmptyState/` and
`.../FuroErrorState/` and are **not** in the skill.

## Import and identity

- Layer: molecule (both).
- `import { FuroEmptyState, FuroErrorState } from '@openreachtech/furo-vue'`
- **Never import the underlying primitive** — only the public exports.
- Manifest: `node_modules/@openreachtech/furo-vue/public/furo-vue/components.json` →
  `components[].name === 'FuroEmptyState'` / `'FuroErrorState'`.
- Neither is a form control: **no `value`, no `v-model`, and no events at all** **[source]** —
  neither component declares `emits`.

### When NOT to use

| Situation | Use instead |
| --- | --- |
| The request is still in flight | `FuroSkeleton` — the skill's cross-reference here is broken, see Conflicts 1 |
| Transient auto-dismissing success/failure notice | `hof-cp-toast` |
| A failure that must block the flow and force a decision | `hof-cp-dialog` (`FuroAlertDialog`) |
| Zero rows **inside** a `FuroTable` | the table's own `empty` / `error` states — see below |

## Props

Identical on both:

| Prop | Type | Default | Notes |
| --- | --- | --- | --- |
| `parcel` | `EmptyStateParcel` / `ErrorStateParcel` `\| null` | `null` | Holds the title and description text. |

## `parcel` fields — identical on both

| Field | Type | Default | Notes |
| --- | --- | --- | --- |
| `title` | `string` | `null` **[source]** | Heading. The `<p class="title">` renders **only when supplied**; the consumer owns the wording. |
| `description` | `string` | `null` **[source]** | Supporting text. The `<p class="description">` renders **only when supplied**. |

**[source]** Both are optional in the `Props` typedef; both context getters are
`this.parcel?.<field> ?? null`. There is nothing else on either parcel.

## Events

**None, on either component.** They carry no value and emit nothing. `FuroErrorState` announces the
failure through `role="alert"` on its root.

## Slots — identical on both

| Slot | Scoped props | Description |
| --- | --- | --- |
| `icon` | — | Leading icon or illustration above the title. **[source]** No default content, and the wrapping `<div class="icon">` renders only when the slot is provided. |
| `action` | — | Action controls below the text (create / retry). **[source]** Same conditional wrapper (`<div class="actions">`). |

## Rendered shape **[source]**

Both render the identical structure, in this fixed order, with the only difference being the root
class and the error state's `role`:

```html
<div class="unit-empty-state">        <!-- FuroErrorState: class="unit-error-state" role="alert" -->
  <div class="icon">…</div>           <!-- only when the icon slot is provided -->
  <p class="title">…</p>              <!-- only when parcel.title is set -->
  <p class="description">…</p>        <!-- only when parcel.description is set -->
  <div class="actions">…</div>        <!-- only when the action slot is provided -->
</div>
```

Both are `inheritAttrs: false` and `v-bind="$attrs"` the remaining attributes onto that root, so a
consumer `class` / `style` / `id` composes onto it.

## Usage

Written to `D:\ORT\rules\` house style and §8 rule 12 — the skill's own example violates both
(Conflicts 2):

```vue
<template>
  <FuroErrorState
    v-if="context.hasFetchError"
    :parcel="context.entriesErrorStateParcel"
  >
    <template #action>
      <FuroButton
        :parcel="context.retryButtonParcel"
        @click="context.onClickRetryFetchEntries()"
      >
        Retry
      </FuroButton>
    </template>
  </FuroErrorState>

  <FuroEmptyState
    v-else-if="context.hasNoEntries"
    :parcel="context.entriesEmptyStateParcel"
  />
</template>
```

Decide which state applies (loading / empty / error / loaded) **in the Context** and expose plain
boolean getters (`hasFetchError`, `hasNoEntries`) for the template to branch on; never inspect a
fetch result in the template.

## Against `FuroTable` — which empty, which error

`FuroTable` already renders its own empty and error rows, and they take precedence in a fixed order
(error beats loading beats empty — see `.hora/digests/hof-cp-table.md`). Two ways to combine them,
and the choice is the implementer's:

| Approach | How |
| --- | --- |
| Inside the table | put `FuroEmptyState` / `FuroErrorState` into the table's `empty` / `error` slots — the placeholder sits in a `<td colspan>` spanning every column, and the header row stays |
| Around the table | branch with `v-if` before the table renders at all — the header row disappears with it |

`FuroTable`'s `error` slot is scoped `{ message }`, which is the natural source for the error
state's `description`.

## Appearance and tokens

**[source]** Both ship `<style>` inside `@layer furo`. Root classes `.unit-empty-state` /
`.unit-error-state` — consistent with the `unit-` prefix rule. They read `--color-muted-foreground`
(and `--color-destructive` for the error state's icon),
`--color-foreground`, `--size-space-small`, `--size-space-large`, `--size-space-2x-large`,
`--font-headline-small`, `--font-body-small` — **none declared in this application**, so the
centering survives but the colour and type do not. Same standing gap recorded in
`hof-cp-button.md`. The icon slot is sized by `font-size: 2rem` on its wrapper.

## Accessibility (project target: WCAG 2.2 AA — checkpoint 18)

- **[source]** `role="alert"` is on `FuroErrorState`'s root only, and it is **static** — the element
  is announced when it is inserted into the DOM. Toggling it with `v-if` (as above) is what makes the
  announcement happen; leaving it mounted and only changing its text may not re-announce.
- `FuroEmptyState` has **no role** — it is silent, correctly, since an empty list is not an alert.
- Neither renders a heading element: the title is a `<p>`, not an `<h2>`. If the region needs a
  heading in the document outline, that is the page's job, not the component's.
- An icon supplied through the `icon` slot is **not** `aria-hidden` by the component — decorative
  icons must carry it themselves.

## Conflicts — named, not resolved

1. **Broken cross-reference in the skill.** Its "when NOT to use" says a still-loading region should
   "use `app-avatar` (`FuroSkeleton`)". `app-avatar` is not a skill in this kit and not a furo
   component; the component meant is **`FuroSkeleton`**
   (`lib/components/molecules/FuroSkeleton/`), which no equipped skill covers. Use the component,
   ignore the name.
2. **Inline `:parcel="{ … }"` in the skill's example.** Barred by §8 rule 12 of
   `expense-note-frontend-staff/ai/contexts/uiux-context.md` and by `D:\ORT\rules\javascript-style.md`;
   recorded once for all five component digests in `.hora/digests/hof-cp-table.md` (Conflicts 2).
   **The project wins.**
3. **The skill types `title` / `description` as required `string` with no default.** Both are
   optional and default to `null`, and each renders only when set — so a state with neither is a
   legal, silent, empty box. Supply at least a title.
