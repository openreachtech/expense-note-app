# hor-sequelize-seeder
<!-- hora-skills-ort-renchan 0.1.0 -->
<!-- source: .claude/skills/hor-sequelize-seeder/ -->

**Read the source above whenever this leaves a question open.**

A seeder (`sequelize/seeders/**/*.cjs`) fills migration-built tables with rows via `bulkInsert`.
A seed row is keyed by the **physical (snake_case) column names** — the migration's `field:` names,
**not** the model's camelCase — because `bulkInsert` writes raw columns.

**Core principle: a seeder is a filled-in template.** Keep the skeleton identical across files so
review attention goes to the **data**, not the plumbing.

## THIS FEATURE (project-specific; overrides the skill's id examples)

- **Allocated row-id prefix is `100`.** Every explicit `id` must be of the form `100xxxxx` —
  8 digits, in `10000000`–`10099999` — and derived from that prefix alone. Use it in place of the
  skill's 6-digit block bases (`100000`, `110000`, …) *and* in place of the master-exemption's
  small sequential ids: e.g. `10000001`, `10000002`, `10000003`, `10000004`. These ids stay far
  above `SMALLINT`'s max, so the SMALLINT-detection benefit below still holds.
- The four expense categories (`transport`, `meals`, `supplies`, `other`) are **deliberately seeded
  rows, not a code enum** — the spec states that as a design seam. The seeder is the **only** place
  these four values appear; do not mirror them into an `app/constants/*.cjs` enum.

## Which directory

| Directory | Script (`--seeders-path`) | Environment | Kind of data |
| --- | --- | --- | --- |
| `master-000001/`, `master-000002/`, … | `db:seed:prod` | production | Canonical **master** (reference / config) data |
| `dev-master/` | `db:seed:dev-master` | dev / CI (local + CI tests) | The **master** data for dev / CI |
| `development/` | `db:seed:dev` | dev / CI | **Operational fixtures** for unit tests |

**How the choice is made** — two axes, environment and kind of data:

- Is it canonical data the running product depends on (providers, models, tools, JSON schemas,
  templates, rates, packages)? → **master**. A fixed master-data set such as `expense_categories`
  belongs here.
- Is it data a user / operator would create at runtime (customers, admins, payments, orders)? →
  **`development`**, and only to give tests something to read. Production never seeds these.
- `dev-master/` is the *same kind of canonical / config data as production master*, loaded for
  dev / CI. It mostly **re-exports** the production master files, plus a few dev-only master
  samples for data production creates through the admin CRUD.
- Master data ships to production; fixture data must never reach production — the directory split
  keeps that boundary unambiguous.

**Release-split master.** Production master is split **one directory per release**,
`master-<6-digit>/`, applied in ascending order. **Current state:** this repo still has the single
pre-split `sequelize/seeders/master/`; the release split is the convention from here on.

**dev-master re-export (DRY) — keep the identical filename as the `master-*/` file:**

```js
// dev-master/20260717135803-000001-ai_models.cjs
'use strict'

/*
 * dev-master duplicate: apply the same AI config as the production master in dev / CI (db:refresh).
 * The real data/logic lives in the production master release dir (DRY).
 */
module.exports = require('../master-000001/20260717135803-000001-ai_models.cjs')
```

Never copy-paste the data. A `dev-master/` file that is **not** in production master (a dev-only
sample) is a normal seeder, not a re-export.

**`.directorykeeper.cjs`** — a no-op seeder (`up`/`down` do nothing) that keeps an otherwise-empty
directory tracked and gives the runner a valid file to load. **Do not delete it.**

## Scripts

| Script | What it does |
| --- | --- |
| `db:seed:prod` | seed the production `master-*` set |
| `db:seed:dev-master` | seed `sequelize/seeders/dev-master` |
| `db:seed:dev` | seed `sequelize/seeders/development` |
| `db:refresh` (alias `npm run r`) | `NODE_ENV=development` → teardown → migrate → **seed:dev-master** → **seed:dev** |

`db:refresh` deliberately does **not** run `seed:prod`. After adding or changing a seeder, run
`npm run r` (or the single matching `db:seed:*` script to load just that set without a teardown).

> Tree note: this repo's `package.json` currently names `db:seed:master` (pointing at the
> `dev-master` path) and `db:seed:dev`; `db:refresh` = teardown → setup → seed:master → seed:dev.
> There is no `db:seed:prod` script yet, so a file placed in `master/` is not loaded by `npm run r`
> — the dev-master counterpart is what dev / CI reads.

## Filename

`{timestamp}-{6-digit-seq}-{table_name}.cjs` — e.g. `20260717135803-000001-customers.cjs`,
`20260717141020-000002-customers_suite.cjs`.

- **`{timestamp}`** — creation time `YYYYMMDDHHmmss` (14 digits; `date +%Y%m%d%H%M%S`). Files are
  applied in **ascending timestamp = creation order** (same as migrations).
- **`{6-digit-seq}`** — zero-padded running number (`000001`, `000002`, …); orders files sharing a
  timestamp and doubles as a human-facing identifier.
- **`{table_name}`** — the physical table name in **snake_case**, the same name as `TABLE_NAME` and
  the migration's table. A **suite** uses `<domain>_suite`.
- **Order parents before children**: create the parent's seeder first so its timestamp is earlier.
- **Older seeders** use the previous `<8-digit-seq>-<kebab-entity>.cjs` form
  (`00050002-ai-models.cjs`). Leave them; write new files in the timestamp form.
- The release number lives on the **directory**; the file inside uses the standard filename.

## File skeleton (verbatim — a whole single-table master seeder)

```js
'use strict'

const TimestampSeedsSupplier = require('@openreachtech/renchan-sequelize/lib/tools/TimestampSeedsSupplier.cjs')

/*
 * Development sample: content plan rates (dev-master).
 * In production these are created via the admin CRUD. 3 plans × 3 tiers (JPY, per month).
 */

const TABLE_NAME = 'content_plan_rates'

const EFFECTIVE_AT = new Date('2024-01-01T00:00:00.000Z')

const seeds = [
  { id: 500001, plan: 'basic', tier: 'small', unit_rate: 50000, currency: 'JPY', effective_at: EFFECTIVE_AT, is_active: true },
  { id: 500002, plan: 'basic', tier: 'medium', unit_rate: 70000, currency: 'JPY', effective_at: EFFECTIVE_AT, is_active: true },
  { id: 500003, plan: 'basic', tier: 'large', unit_rate: 100000, currency: 'JPY', effective_at: EFFECTIVE_AT, is_active: true },
  // ...
]

module.exports = {
  up: async (queryInterface, Sequelize) => {
    await queryInterface.bulkInsert(TABLE_NAME, TimestampSeedsSupplier.supplyAll(seeds), {})
  },

  down: async (queryInterface, Sequelize) => {
    await queryInterface.bulkDelete(TABLE_NAME, { id: seeds.map(it => it.id) })
  },
}
```

Order of parts: `'use strict'` → `require` `TimestampSeedsSupplier` (+ any domain constants) →
intent comment → `TABLE_NAME` → one or more `seeds` arrays → `module.exports = { up, down }`.

- Seeders are **CommonJS** `.cjs`; sequelize-cli loads them that way.
- **Intent comment** (`/* ... */`) near the top: what this seed is, and its design-doc / plan
  reference. Structural comments in English; domain prose is often Japanese in this repo — match
  the surrounding files.
- **`TABLE_NAME`** is a **string** for a single table, or an **object** when one file fills several
  related tables (a suite): `const TABLE_NAME = { CUSTOMERS: 'customers', CUSTOMER_BASICS: 'customer_basics' }`.
- Optionally `require` domain constants (`app/constants/*.cjs`) for ids / names so the seed and the
  app share one source of truth: `const seeds = [{ id: AI_PROVIDER.DEFAULT.ID, name: AI_PROVIDER.DEFAULT.NAME }]`.
  (Not for this feature — see the project note above.)
- `up`'s third `{}` is `bulkInsert`'s options. `down` deletes exactly the rows `up` inserted, by the
  explicit id list.
- **Suites insert parent → child in `up` and delete child → parent in `down`** (reverse order), so a
  child's FK column always sees its parent row.
- **Async fulfillment**: when a value must be computed (hashing a password via `Encipher`), build the
  fulfilled array with `await Promise.all(seeds.map(fulfillX))` inside `up` before `bulkInsert`, and
  keep the plain `seeds` array for `down`'s id list.
  full text: references/notation.md#async-fulfillment-computed-values-such-as-password-hashes

## Timestamps

- Do **not** put `created_at` / `updated_at` in seed rows. `TimestampSeedsSupplier.supplyAll(seeds)`
  injects both (set to "now") for every row; `deleted_at` is left unset (defaults to null).
- `supplyOne` returns `{ created_at: now, updated_at: now, ...seed }` — because `...seed` is spread
  **after**, a row *can* override them, but normally you don't.
- **Business datetimes are different** — a column that means something in the domain
  (`registered_at`, `saved_at`, `effective_at`, `generated_at`, `expired_at`) **is** written
  explicitly in the seed; it is not an audit column.

## Row ids

**Every seed row carries an explicit `id`, never auto-increment** — `down` deletes by the exact id
list, and other seeds' FK columns point at known ids. `id` comes **first** in the row object, then
the snake_case columns.

- **Skill default:** each table's rows occupy a distinct id block whose **step is 10,000**, with
  bases at multiples of 10,000 **at or above `100000`** (6 digits: `100000`, `110000`, `120000`, …);
  rows increment by 1 inside the block (~10,000 rows per block). **Never use a sub-100,000 base.**
- **Why ≥ `100000`:** such an id exceeds `SMALLINT`'s max (32,767 signed / 65,535 unsigned), so a
  column wrongly declared `SMALLINT` in the migration **overflows and fails at seed time** — the
  schema mistake is caught while seeding, not in production.
- **A base may be reused across suites** — uniqueness only has to hold *within one table*
  (`customers` and `admins` may both start at `100000`).
- **A FK column holds an id from the referenced table's block**
  (`{ id: 110001, customer_id: 100001 }`); a child table's block sits above its parent's so FK ids
  stay readable.
- **Hierarchical rows** use a **structured id inside the block**: access tokens are `14` + 2-digit
  customer + 2-digit sequence → `140101`, `140201`, `140202`, all inside `140000`.
- `development/` suites: 1st table (root) `100000`, 2nd (`*_basics`) `110000`, 3rd (`*_secrets`)
  `120000`, 4th (`*_password_hashes`) `130000`, 5th (`*_access_tokens`) `140000`.
- **Production master (`master-*/`) is exempt from blocks:** its ids are small sequential (`1`…`7`)
  or taken from `app/constants` — master ids are part of the product, stable and meaningful.
  (This feature instead uses the allocated `100` prefix; see the project note above.)
- Blocks already in use — do not reuse a base for a new seeder in the same table:
  `content_plan_rates` `500000`, `content_benchmark_samples` `510000`, `software_packages` `520000`,
  `software_package_options` `530000`, `integration_clients` `600000`.

## Keep every value distinct within a seeder

Choose seed values so that **the same value never appears in two different columns** — a unit test
that accidentally reads `user_id` where it meant `id` must fail loudly, not match a row and pass.

- **id vs same-row FK / numeric columns**: keep them in different ranges. If a row's `id` is
  `100001`, a `user_id` on that row must sit in a different block (e.g. from `110001`), never
  `100001`.
- **text columns**: two string columns (`username` and `name`) always hold **different** strings.

```js
// Bad example (id == user_id, and username == name — a column mix-up would pass silently)
{ id: 100001, user_id: 100001, username: 'taro', name: 'taro' }

// Good example (every column in its own value range / space)
{ id: 110001, user_id: 100001, username: 'user-110001', name: 'Alpha Taro' }
```

## `development/` fixtures cover the app's operational cases

Applies to `development/` **only** — **not** to master / dev-master, which hold canonical config
data, not operational case variety. A `development/` seeder must cover, at minimum: **success
cases**; **failure cases** (rejected by a business rule); **error cases** (abnormal / inconsistent
states the code must still handle); and **status variety within the success cases** — one row per
distinct status the entity defines. Label each row's case with a short comment (`// active`,
`// suspended`, `// orphaned — error path`) and group a status's rows on adjacent ids so the block
reads as a case list. Scope it to what the app actually branches on — one representative row per
branch, not a combinatorial explosion.
full text: references/development-coverage.md
