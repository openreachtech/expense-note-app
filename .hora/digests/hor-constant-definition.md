# hor-constant-definition
<!-- hora-skills-ort-renchan 0.1.0 -->
<!-- source: .claude/skills/hor-constant-definition/ -->

**Read the source above whenever this leaves a question open.**

## Caution first — what must NOT become a constant here

The four expense **categories** are **seeded database rows** (`expense_categories`, id +
`name` + `display_order`, filled by the master seeder). The spec names this a design seam:
"a category is already a row with an id, seeded, **never an enum written into the code**".
So: **no category constant file, no category enum, no id hash for the four categories.**
Read them from the table; reference them by `expense_category_id`.

The `expenses.status` value (`recorded`, the only value this version writes) is the opposite
case — an enum-like value that lives in code, not in a table. It is what this skill applies to.

| Value set | Constant file? |
|---|---|
| `status` value(s) of `expenses` (`recorded`) — enum-like, code-owned | yes — the two-file pair below |
| the four expense categories — seeded rows with ids | **never** — read the rows |

## Grand principle: one category per file, ALWAYS two files

Every constant is defined as **two files with the same base name** — no per-constant
judgment about whether a seeder will ever need it:

| File | Module system | Role |
| --- | --- | --- |
| `constants/<name>.cjs` (repo root) | CommonJS | **Master — the single source of truth.** The actual values. |
| `app/constants/<name>.js` | ESM | **Bridge — a pure re-export** of the `.cjs` via the custom `require`. |

(In this project, "repo root" = the backend package root, `expense-note-backend/`.)

- App code **always imports the bridge**; app code never reaches into root `constants/*.cjs`.
- Seeders (CommonJS `.cjs`) **always `require` the master** directly.
- The `.cjs` is the **only** copy of the values. The `.js` bridge never restates them.
- One category (one hash / one set) per file. No grab-bag `constants.js`.
- Comments: keep structural comments in English; domain notes in the project's domain-prose language.

## The `.cjs` master — source of truth

```js
// Good: constants/memberRankConstants.cjs — the single source of truth (CommonJS)
'use strict'

module.exports = {
  MEMBER_RANK: {
    BRONZE: {
      ID: 1,
      NAME: 'Bronze',
      IS_ACTIVE: true,
      DISPLAY_ORDER: 10,
    },
    // ...
  },
}
```

## The `.js` bridge — this is the whole file, never more

```js
// Good: app/constants/memberRankConstants.js — thin ESM bridge over the .cjs master
import {
  require,
} from '../globals/_.js'

const MEMBER_RANK_CONSTANT_HASH = require('../../constants/memberRankConstants.cjs')

export default MEMBER_RANK_CONSTANT_HASH
```

- `require` is the **custom** one (`createRequire(import.meta.url)`) re-exported from the
  project's globals module (`app/globals/_.js`) — there is no native `require` in ESM.
- Pure re-export: no reshaping, no merging, no extra keys.

## How the two consumers read the one source

```js
// seeder (.cjs) — requires the master directly
const { MEMBER_RANK } = require('../../../constants/memberRankConstants.cjs')

// app (ESM) — imports the bridge
import MEMBER_RANK_CONSTANT_HASH from '../constants/memberRankConstants.js'
```

```js
// Avoid: stopping at an ESM-only definition (no .cjs master)
export default { MEMBER_RANK: { /* ... */ } } // a seeder (.cjs) cannot import this; always add the .cjs master
```

## Naming

- **File**: camelCase, named after the category (`memberRankConstants`, `userPermission`).
  The `.cjs` master and its `.js` bridge share the **same base name**.
- **Exported constant**: `SCREAMING_SNAKE_CASE` (`MEMBER_RANK_CONSTANT_HASH`, `USER_PERMISSION`),
  as the **default export**. A `_HASH` / `_CONSTANTS` suffix is common for a lookup hash but not
  mandatory — name it for the category, not the type; no forbidden suffixes (`info` / `data` / `list`).
- Tempted to add a second unrelated set? Make another pair of files.
