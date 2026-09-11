# hor-sequelize-migration
<!-- hora-skills-ort-renchan 0.1.0 -->
<!-- source: .claude/skills/hor-sequelize-migration/ -->

**Read the source above whenever this leaves a question open.**

Migrations define the **physical schema** (tables, columns, indexes); the logical declarations
(attributes / association / scope / hook) belong to `hor-sequelize-model`. Keep the two in
one-to-one correspondence. This repo does **not** `sync` — the schema is built entirely by
migrations (dev = SQLite / staging = MySQL / live = MariaDB). A model's `unique: true` is only a
declaration; **the real object is always created by the migration**.

**Grand principle: a migration is a filled-in template.** Every migration has the same skeleton;
keep it identical so review attention goes to the file-specific columns / indexes. Do not write it
cleverly; lean toward the layout of the existing migrations.

**Comments: English for structural conventions** (e.g. `// ForeignKey must start with upper case.`),
the surrounding language for domain notes (often Japanese in this repo).

## Naming & placement

| | |
|---|---|
| location | one migration = one file, directly under `sequelize/migrations/` |
| extension | `.cjs` (CommonJS — sequelize-cli will not load `import`/ESM) |
| filename | `{timestamp}-{seq}-<operation>-<table>[-<column>].cjs` |
| `{timestamp}` | creation time `YYYYMMDDHHmmss`, 14 digits (`date +%Y%m%d%H%M%S`); ascending = apply order |
| `{seq}` | **6-digit zero-padded** running number (`000001`, `000002`, …) |
| `<operation>` | `create_table` / `alter_table` |
| examples | `20260717135803-000001-create_table-content_generations.cjs`<br>`20260717140512-000002-alter_table-content_generation_jobs-webhook_url.cjs` |

Older migrations use the legacy `<8-digit-seq>-create_table-...` form (no timestamp). Write new
files in the form above.

## Create-table skeleton (verbatim shape)

```js
'use strict'

const MigrationAttributeFactory = require('@openreachtech/renchan-sequelize/lib/tools/MigrationAttributeFactory.cjs')

const TABLE_NAME = 'content_generations'
const COLUMN_NAME = {
  CONTENT_GENERATION_JOB_ID: 'content_generation_job_id',
  RESULT_JSON: 'result_json',
  MODEL: 'model',
}

// Define an initialism only for a column whose index name would run long.
const SHORT_COLUMN_NAME = {
  CONTENT_GENERATION_JOB_ID: 'cgji',
}

module.exports = {
  async up (
    queryInterface,
    Sequelize
  ) {
    const factory = MigrationAttributeFactory.create(Sequelize)

    await queryInterface.createTable(TABLE_NAME, {
      ...factory.ID_BIGINT,

      // ForeignKey must start with upper case.
      ContentGenerationJobId: {
        type: Sequelize.BIGINT,
        field: COLUMN_NAME.CONTENT_GENERATION_JOB_ID,
        allowNull: false,
      },
      resultJson: {
        type: Sequelize.JSON,
        field: COLUMN_NAME.RESULT_JSON,
        allowNull: true,
      },
      model: {
        type: Sequelize.STRING(64),
        field: COLUMN_NAME.MODEL,
        allowNull: false,
      },

      ...factory.TIMESTAMPS,
    })

    // A 1:1 relation is enforced by a UNIQUE index (no DB FK).
    await queryInterface.addIndex(TABLE_NAME, [
      COLUMN_NAME.CONTENT_GENERATION_JOB_ID,
    ], {
      unique: true,
      name: [
        TABLE_NAME,
        SHORT_COLUMN_NAME.CONTENT_GENERATION_JOB_ID,
        'unique',
      ].join('_'),
    })

    return Promise.resolve()
  },

  async down (
    queryInterface,
    Sequelize
  ) {
    return queryInterface.dropTable(TABLE_NAME)
  },
}
```

- Both `up` / `down` are `async` and take `(queryInterface, Sequelize)` **in that order**, chopped
  one argument per line as above.
- `up` order: `MigrationAttributeFactory.create(Sequelize)` → `createTable` → `addIndex` →
  `return Promise.resolve()`.
- `down` of a create-table migration is always `return queryInterface.dropTable(TABLE_NAME)`.

## Constants

- `TABLE_NAME` (string) and `COLUMN_NAME` (object) are declared **outside `up`/`down`, before
  `module.exports`**. They are the single source of truth for physical names.
- `COLUMN_NAME` keys are `SCREAMING_SNAKE`; values are the **physical column names (snake_case)**.
  The same value is referenced by both the column's `field:` and the index `name`.
- **Do not inline string literals** into column definitions or index names.

## Presets — never hand-write `id` / timestamps

`...factory.ID_BIGINT` is spread **at the top** of the `createTable` object; the timestamps spread
**at the end**.

| preset | columns created |
|---|---|
| `factory.ID_BIGINT` | `id` (bigint / autoIncrement / primaryKey / NOT NULL) |
| `factory.ID_INTEGER` | `id` (integer version) — only when an integer PK is needed |
| `factory.TIMESTAMPS` | `created_at` / `updated_at` (`DATE(3)` / NOT NULL) |
| `factory.TIMESTAMPS_WITH_DELETED_AT` | the above + `deleted_at` (`DATE(3)`) — for soft-delete tables, paired with `paranoid: true` on the model side |

The model side does **not** declare these in its attributes; only the migration creates the columns.

## Column notation

Each column: **camelCase key** + `type: Sequelize.X` + `field: COLUMN_NAME.X` + **always
`allowNull`**, plus `defaultValue` when there is a default.

```js
currency: {
  type: Sequelize.STRING(8),
  field: COLUMN_NAME.CURRENCY,
  allowNull: false,
  defaultValue: 'JPY',
},
```

Omitting `field:` makes the physical name camelCase (mismatching the model's `underscored: true`);
omitting `allowNull` silently defaults it to `true`.

| Type | Use |
|---|---|
| `BIGINT` | PK / FK-like id |
| `INTEGER` | small integer PK, quantity, version |
| `STRING(n)` | variable-length string (identifier `32`, display name / email `191` — pick by use; never default to "255 for now") |
| `TEXT` / `TEXT('medium')` | long text (body, message, error). Plain `TEXT` caps ~64KB; use `TEXT('medium')` (MEDIUMTEXT, ~16MB) for anything that may grow; `TEXT('long')` only if that is not enough |
| `DATE(3)` | millisecond-precision datetime — **the default for all datetimes** (`registeredAt` / `expiresAt`) |
| `BOOLEAN` | truth value (`isActive`) |
| `JSON` | structured data (`resultJson`; an FK-less id array `optionIdsJson`) |
| `DECIMAL(p, s)` | money / rate (`dailyRate` as `DECIMAL(14, 2)`) |
| `BLOB('long')` | binary (uploaded file body) |
| `ENUM(...)` | **avoid by default** — use a domain constant + `STRING(n)` |

Keep the length and the `allowNull` / `defaultValue` identical on the model and migration sides.

## Foreign keys — column only, never a DB constraint

- Create the FK-like column: type `BIGINT`, key **starting with an uppercase letter**
  (`CustomerId`, `ContentGenerationJobId`), with `// ForeignKey must start with upper case.`
  in English immediately above it. `field:` maps it to snake (`..._id`).
  The uppercase start is what lets the model's association resolve `<associated model name>` + `Id`.
- **Never** write `references`, `onDelete`, `onUpdate`, or `queryInterface.addConstraint(...)` for an
  FK. There is not a single FK constraint in the existing migrations. Referential integrity is
  enforced in the app layer.
- **1:1** → a **UNIQUE index** on the parent-id column, with the comment verbatim:
  `// A 1:1 relation is enforced by a UNIQUE index (no DB FK).`
- **Circular FK / optional relation** → column only, `allowNull: true`, comment
  `// ForeignKey must start with upper case. (circular FK, column only)`.
- **A set of ids you do not want as FKs** → a `JSON` array column (`optionIdsJson`) with
  `// An array of ids without an FK.`
- Index every FK column: **1:1 → UNIQUE index, 1:N → plain index, a join table's composite key →
  composite UNIQUE index.**

## Indexes

`queryInterface.addIndex(TABLE_NAME, [columns...], { name, unique? })` — the second argument is an
array of **physical column names** (`COLUMN_NAME.X`). `name` is **always** given explicitly, built
by `.join('_')` on an array (never auto-named).

| kind | shape | name ends in |
|---|---|---|
| plain | no `unique` key | `'index'` → `<table>_<column>_index` |
| UNIQUE | `unique: true` | `'unique'` → `<table>_<column>_unique` |

- **One index → `await` it directly. Multiple indexes on the same table → wrap them in
  `Promise.all([...])`.**

> ⚠ **OVERRIDDEN IN THIS PROJECT — do NOT use `Promise.all` for indexes.** The always-on rule
> `D:/ORT/rules/migrations-and-seeders.md` states: "**Indexes: sequential
> `await queryInterface.addIndex(...)` — never `Promise.all`.**" Always-on rules win over an
> equipped skill here (Q10), and the repository already follows the rule —
> `…-000009-create_table-staff_member_refresh_tokens.cjs` awaits its three indexes one after
> another. **Write sequential `await`s.** The `Promise.all` sample below is the skill's text,
> kept so the disagreement is visible; do not copy its shape.

```js
await Promise.all([
  queryInterface.addIndex(TABLE_NAME, [
    COLUMN_NAME.ACCESS_TOKEN,
  ], {
    unique: true,
    name: [TABLE_NAME, COLUMN_NAME.ACCESS_TOKEN, 'unique'].join('_'),
  }),
  queryInterface.addIndex(TABLE_NAME, [
    COLUMN_NAME.STATUS,
    COLUMN_NAME.EXPIRES_AT,
  ], {
    name: [TABLE_NAME, COLUMN_NAME.STATUS, COLUMN_NAME.EXPIRES_AT, 'index'].join('_'),
  }),
])
```

- **Composite index**: either list all the columns (`content_sessions_status_expires_at_index`) or
  fold them into one meaningful label when that runs long
  (`[SHORT_TABLE_NAME, 'plan_tier_currency_effective_at', 'unique'].join('_')` →
  `cpr_plan_tier_currency_effective_at_unique`).
- A UNIQUE named index is **the real constraint** behind a 1:1 relation or a natural key.

### Shortening a long index name

DB identifier limit is 64 chars; as a safety margin **shorten once a name would run past ~50
characters** — and not before (do not mechanically initialize; `customers_registered_at_index`
stays full). Shorten in this priority order:

> ⚠ **OVERRIDDEN IN THIS PROJECT — the ~50-character threshold does not apply.** The always-on
> rule states: "**Index name always uses `SHORT_COLUMN_NAME`**", with no length condition, and all
> ten existing migrations follow it — `expenses_smi_so_index` is nowhere near 50. Always-on wins
> (Q10).
>
> **How far to abbreviate is settled separately, and it is not "initialize every word":**
> abbreviate **for length only, and keep a short name whole**. So a composite of `email` and
> `attempted_at` becomes `sign_in_attempts_email_aa_index`, not `sign_in_attempts_e_aa_index` —
> `email` is one short word and carries whole, while `attempted_at` abbreviates the way `spent_on`
> → `so` does in the expenses migration. The priority order below still holds for **which** name
> to shorten first.

1. **Shorten the column name(s) only** — define `SHORT_COLUMN_NAME`, keep `TABLE_NAME` in full.
   `content_generations_content_generation_job_id_unique` (52) → `content_generations_cgji_unique` (31).
2. **Only if still too long, also shorten the table name** — define `SHORT_TABLE_NAME`.
   `cgrf` + `cgji` + `index` → `cgrf_cgji_index`.

Abbreviate by splitting the snake_case name on `_` and concatenating the **first character of each
word**. Always store abbreviations in `SHORT_COLUMN_NAME` / `SHORT_TABLE_NAME` constants and
reference them from the `addIndex` `name` — never inline. If two abbreviations collide within a
table, add a second letter.

| Original | Abbreviation |
|---|---|
| `content_generation_job_id` | `cgji` |
| `content_generation_requirement_files` | `cgrf` |
| `content_generation_job_packages` | `cgjp` |
| `content_plan_rates` | `cpr` |
| `chat_room_id` | `cri` |
| `customer_id` | `ci` |

## alter_table (add / remove a column)

Filename `{timestamp}-{seq}-alter_table-<table>-<column>.cjs`. `up` uses `addColumn`; `down` is
**symmetric** (drops exactly what `up` added). No `MigrationAttributeFactory` require is usually
needed (the PK / timestamps already exist).

- **Never use `queryInterface.removeColumn`** — it does not work correctly on MariaDB (live). Drop a
  column with raw SQL via `queryInterface.sequelize.query`, quoting identifiers with backticks:
  `ALTER TABLE \`${TABLE_NAME}\` DROP COLUMN \`${COLUMN_NAME}\``, preceded by the comment
  `// removeColumn does not work on MariaDB, so drop via raw SQL DROP COLUMN.`
- A `/* ... */` **intent comment at the top of the file** is required: ticket number, spec
  reference, backward-compat notes.
- Single column → `COLUMN_NAME` may be a string. Multiple → `COLUMN_NAME` is an object; the
  `addColumn`s may be grouped with `Promise.all`, but the drops run as **one raw statement per
  column, sequentially** (SQLite drops only one column per statement).
- Adding a column to a table that already holds rows → `allowNull: true` or a `defaultValue`.
  Never add a NOT NULL column after the fact.

full text: .claude/skills/hor-sequelize-migration/references/alter-table.md

## Applying changes to the local DB

- `npm run r` (= `npm run db:refresh`) — with `NODE_ENV=development`: teardown → migrate →
  seed:master → seed:dev. Run after changing a migration / seeder.
- Individually: `npm run db:setup` (migrate only) / `npm run db:teardown` (delete the SQLite files).
