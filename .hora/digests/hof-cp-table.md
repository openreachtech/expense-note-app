# hof-cp-table
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-cp-table/ -->

**Read the source above whenever this leaves a question open.**

`FuroTable` (organism) + `FuroPagination` (molecule) — the entries list of `#expense-entry`.
`FuroTable` is a controlled, presentational `<table>`: it renders `columns` + `rows` and emits
intent; it never sorts, pages or fetches. `FuroPagination` knows nothing about `FuroTable` — it
emits which page was picked. Verified against the installed `@openreachtech/furo-vue@1.3.2`; facts
marked **[source]** were read from
`node_modules/@openreachtech/furo-vue/lib/components/organisms/FuroTable/` and
`.../molecules/FuroPagination/` and are **not** in the skill.

## Import and identity

- `import { FuroTable, FuroPagination } from '@openreachtech/furo-vue'`
- **Never import the rendering internals** — only the two public exports.
- Manifest: `node_modules/@openreachtech/furo-vue/public/furo-vue/components.json` →
  `components[].name === 'FuroTable'` / `'FuroPagination'`. Read it before writing markup to
  confirm this surface is still current.
- Neither follows the form-control (`parcel.value`) contract. `FuroTable` uses
  `parcel.selectedRowKeys` + `v-model:selectedRowKeys`; `FuroPagination` uses `parcel.page` +
  `v-model:page`. Both are the documented exception to the project's no-`v-model` rule, which
  applies only to custom in-project components, not to `furo-vue` components.

## Props

Both take a single `parcel` prop — no other props.

| Component | Prop | Type | Default |
| --- | --- | --- | --- |
| `FuroTable` | `parcel` | `FuroTableParcel \| null` | `null` |
| `FuroPagination` | `parcel` | `FuroPaginationParcel \| null` | `null` |

## `FuroTable` `parcel` fields

| Field | Type | Default | Notes |
| --- | --- | --- | --- |
| `columns` | `Array<FuroTableColumn>` | `[]` | Column configuration. |
| `rows` | `Array<object>` | `[]` | Raw row records, rendered in the order given. |
| `rowKey` | `string` | `'id'` | Field used as each row's unique key. |
| `selectable` | `boolean` | `false` | Selection column + select-all checkbox. |
| `selectedRowKeys` | `Array<string \| number> \| null` | `null` | Controlled selection. `null` = table holds its own. |
| `sort` | `{ field: string, direction: 'asc' \| 'desc' } \| null` | `null` | Active sort indicator (controlled). |
| `loading` | `boolean` | `false` | Loading state. |
| `errorMessage` | `string \| null` | `null` | When set, renders the error state — takes precedence over loading / empty / rows. |
| `emptyText` | `string` | `'No records'` | Default empty-state text. |
| `virtual` | `FuroTableVirtual \| null` | `null` | Fixed-height virtual scrolling. |
| `totalRecords` | `number \| null` | `null` (falls back to `rows.length`) | Full dataset size — total scroll height in virtual mode. |
| `rowOffset` | `number` | `0` | Global index of `rows[0]` — lets `rows` be a sparse window. |
| `resizable` | `boolean` | `false` | Drag handles on column edges. |
| `compact` | `boolean` | `false` | Denser rows. |
| `gridLines` | `boolean` | `false` | Vertical borders between columns. |
| `rowActionsLabel` | `string` | `''` | **[source]** Header text of the row-actions column. **Not in the skill.** |
| `rowActionsWidth` | `string` | `'6rem'` | **[source]** `<col>` width of the row-actions column. **Not in the skill.** |

`FuroTableColumn` **[source]** (`FuroTableContext.js` typedef, line ~1903) — `field` required,
the rest optional:

```js
{
  field: string,
  label: string,        // falls back to `field`
  sortable: boolean,
  align: 'start' | 'center' | 'end',
  width: string,
}
```

`FuroTableVirtual` **[source]**: `{ itemSize: number, height: number, overscan?: number, endThreshold?: number }`.

### State precedence **[source]**

| Rendered | When |
| --- | --- |
| error row | `errorMessage !== null` — beats everything |
| centered loading row | `loading` **and** `rows.length === 0` (first load) |
| floating loading overlay | `loading` **and** `rows.length > 0` (refresh keeps rows mounted, layout never collapses) |
| empty row | not loading, not error, `rows.length === 0` |

While `loading` the scroll viewport gets `inert` and the root gets `aria-busy="true"` **[source]**.
The selection checkboxes are disabled while loading.

## `FuroPagination` `parcel` fields

| Field | Type | Default | Notes |
| --- | --- | --- | --- |
| `page` | `number \| null` | `null` | Controlled current page (1-based). Omitted = component manages its own. |
| `offset` | `number \| null` | `null` | Controlled page as a zero-based record offset; page = `floor(offset / limit) + 1`. **Provide only one of `page` / `offset`.** |
| `limit` | `number` | `20` | Records per page. |
| `totalRecords` | `number` | `0` | Total records across all pages. |
| `siblingCount` | `number` | `2` | Pages each side of the current before an ellipsis. |
| `showEdges` | `boolean` | `true` | Pin first and last page with ellipsis gaps. |
| `disabled` | `boolean` | `false` | Disables the whole control. |

**[source]** The skill says "when both `page` and `offset` are set, `page` takes precedence". That
holds only in production: outside `NODE_ENV === 'production'`, `throwOnControlledConflict()`
(`FuroPaginationContext.js:207`) **throws** when the two disagree — the message names both values
and says "Provide only one to avoid divergence." Pass exactly one.

## Events

### FuroTable

| Event | Payload | Fires when |
| --- | --- | --- |
| `sort:change` | `{ field, direction: 'asc' \| 'desc' \| null }` | A sortable header is clicked. Cycles **asc → desc → null**; `null` restores the default order. |
| `selection:change` | `{ selectedRowKeys: Array<string \| number> }` | A row or select-all toggles. |
| `update:selectedRowKeys` | `Array<string \| number>` | `v-model:selectedRowKeys` (fires with `selection:change`). |
| `range:change` | `{ first: number, last: number }` | Virtual mode — visible global row window moved. |
| `reach-end` | payload | **[source]** Virtual mode — a downward scroll came within `endThreshold` rows of the end; fires once per loaded window, never on upward scroll. Prefer it over `range:change` for infinite append. **Not in the skill.** |
| `row:click` | `{ row: object, rowKey: string \| number \| null }` | A data row is clicked. |

In the template the kebab forms are `@sort:change`, `@selection:change`,
`@update:selected-row-keys`, `@range:change`, `@reach-end`, `@row:click`.

### FuroPagination

| Event | Payload | Fires when |
| --- | --- | --- |
| `update:page` | `number` | Page changed — enables `v-model:page`. |
| `page:change` | `{ page, offset, limit }` | Page changed. `offset` = `(page - 1) * limit`, so an offset/limit data source can use the payload directly. |

**[source]** Both fire from one `onPageChange`, `update:page` first, on every page pick.

## Slots

### FuroTable

| Slot | Scoped props | Description |
| --- | --- | --- |
| `header-cell` | `{ column }` | Header cell content. Falls back to `column.label`. |
| `sort-icon` | `{ column }` | **[source]** Sort indicator, rendered only for a sortable column. Defaults to `ph:arrows-down-up` / `ph:arrow-up` / `ph:arrow-down`. **Not in the skill**, and not in the component's own `FuroTableSlots` typedef either — it exists only in the template (`FuroTable.vue:128-140`). |
| `cell` | `{ row, column, value }` | Cell content. Falls back to the raw value. |
| `row-actions` | `{ row }` | **[source]** Per-row action controls. **Not in the skill — see Conflicts 1.** |
| `loading` | — | Loading content. Falls back to a `ph:spinner`. |
| `empty` | — | Empty content. Falls back to `parcel.emptyText`. |
| `error` | `{ message }` | Error content. Falls back to `parcel.errorMessage`. |
| `placeholder` | `{ index }` | Unloaded virtual row. Falls back to a shimmer skeleton. |
| `pagination` | — | Host a `FuroPagination` here. Empty by default. |
| `footer` | `{ selection: { keys, count } }` | **[source]** Footer region below the rows (bulk-action bar, summary). Rendered only when the slot is provided. **Not in the skill.** |

### FuroPagination

| Slot | Scoped props | Description |
| --- | --- | --- |
| `previous` | — | Previous-page button content (default `ph:caret-left`). |
| `next` | — | Next-page button content (default `ph:caret-right`). |
| `ellipsis` | — | Gap marker (default `ph:dots-three`). |
| `page` | `{ page }` | Per-page entry content. Rendered as-child, so the slot may supply an anchor; the click still drives `page:change`. |

## The row-actions column — providing the slot builds the column

**[source]** `hasRowActionsColumn()` is `Boolean(this.componentContext.slots?.['row-actions'])`
(`FuroTableContext.js:380`). Providing the slot makes the table, by itself:

- add a trailing `<col class="actions-col">` at `parcel.rowActionsWidth`;
- add a trailing `<th class="actions-cell">` holding `parcel.rowActionsLabel`;
- add a trailing `<td class="actions-cell" @click.stop>` per row, in **both** the plain and the
  virtual branch (`FuroTable.vue:266-277` and `356-370`), passing `{ row }` to the slot;
- count that column in the `colspan` of the error / loading / empty rows.

`@click.stop` is **already applied by the table** on the actions `<td>` (and on the selection
`<td>`), so an action button inside `row-actions` does not need its own `.stop` to suppress
`row:click`. A control placed in the **`cell`** slot still does.

## Usage

Use the `row-actions` slot; do **not** follow the skill's example (Conflicts 1). Written to
`D:\ORT\rules\` house style and §8 rule 12 (Conflicts 2):

```vue
<template>
  <FuroTable
    :parcel="context.expenseTableParcel"
    @sort:change="context.onSortChange($event)"
    @row:click="context.onRowClick($event)"
  >
    <template #row-actions="{ row }">
      <FuroButton
        :parcel="context.correctButtonParcel"
        @click="context.onClickCorrect(row)"
      >
        Correct
      </FuroButton>
    </template>

    <template #pagination>
      <FuroPagination
        :parcel="context.paginationParcel"
        @page:change="context.onPageChange($event)"
      />
    </template>
  </FuroTable>
</template>
```

Every parcel is a getter on the page Context; sort / page / selection state, the fetch, and the
row-action handlers live there. The table holds no GraphQL, router or fetch logic.

## Appearance and tokens

**[source]** Both components ship `<style>` inside `@layer furo`. Root classes are
`.unit-table` (plus `loading` / `compact` / `grid-lines`) and `.unit-pagination` (plus `disabled`) —
consistent with the `unit-` prefix rule. Colours and dimensions are `var()` references the
components do not declare (`--color-*`, `--size-space-tiny`, …); **this project imports no furo
stylesheet**, so they resolve to nothing. Same standing gap recorded in `hof-cp-button.md`.

## Accessibility (project target: WCAG 2.2 AA — checkpoint 18)

**The skill says nothing about accessibility.** All **[source]**:

- **No `aria-sort`.** A sortable header is a `<span class="header-label" role="button" tabindex="0">`
  inside a plain `<th>`; the only sort cue is the icon's `active` class and the icon name. Sort state
  is not exposed programmatically.
- **Selection checkboxes have no accessible name.** `extractSelectAllParcel()` /
  `extractRowSelectionParcel()` return `{ value, indeterminate?, disabled }` only — the table renders
  no `<label>` and no `aria-label` around the `FuroCheckbox`.
- **`aria-busy="true"`** is set on the root while `loading`, and the scroll viewport becomes `inert`
  — focus inside the table is lost on a refresh. Decide where focus goes; the component does not.
- **`FuroPagination` names are hard-coded English** by reka-ui: `aria-label="Previous Page"` /
  `"Next Page"` (`reka-ui/dist/Pagination/PaginationPrev.js:27`, `PaginationNext.js:27`) and
  `aria-label="Page N"` + `aria-current="page"` per entry (`PaginationListItem.js:33-35`). They are
  not reachable through the parcel. The root renders a `<nav>` (`PaginationRoot.js:51`) with no
  accessible name of its own — add one as a plain attribute.

## Conflicts — named, not resolved

1. **The skill's `cell`-slot "actions column" example is wrong for the installed library.** The
   skill writes an `actions` column into `parcel.columns` and branches inside `#cell` on
   `column.field === 'actions'`, with a hand-written `@click.stop`. The installed component has a
   dedicated **`row-actions` slot** that renders its own trailing header and cell with `@click.stop`
   already applied (verified above). **The library wins — use `row-actions`; do not follow the
   skill's example.** That example also puts a comparison expression in the template, which §8 rule
   12 forbids on its own.
2. **Inline `:parcel="{ … }"` in every component skill's example loses to §8 rule 12.** Each of the
   five component skills writes the parcel as an inline object literal in the template
   (`:parcel="{ columns: context.memberColumns, … }"`, `:parcel="{ limit: …, totalRecords: … }"`,
   and the same shape in `hof-cp-date-time`, `hof-cp-select`, `hof-cp-dialog`, `hof-cp-empty-state`).
   `expense-note-frontend-staff/ai/contexts/uiux-context.md` §8 rule 12 — "**No JavaScript logic in a
   `<template>`.** Logic moves onto a member of the page's context class. The template reads
   `context.*`" — plus `D:\ORT\rules\javascript-style.md` (an object literal passed as an argument is
   always chopped, one property per line, trailing comma) both bar it. `hof-uiux-forge` itself states
   the project context wins over a skill's example. **The project wins: every parcel is a named
   getter on the page Context.** Recorded here once for all five component digests.
3. **`FuroTableSlots` omits `sort-icon`.** The component's own slot typedef lists nine slots and not
   `sort-icon`, which the template does render. The template wins; type-checking a `#sort-icon`
   usage may complain.
