# hor-database-design
<!-- hora-skills-ort-renchan 0.1.0 -->
<!-- source: .claude/skills/hor-database-design/ -->

**Read the source above whenever this leaves a question open.**

Design-level only: **what** tables/columns exist and why. The physical declaration (DataTypes, `DATE(3)`, `TEXT('medium')`, `JSON`, no DB foreign-key constraint, indexes) is written per `hor-sequelize-migration`; model attributes, associations and the Mixins per `hor-sequelize-model`. Code comments in examples are English.

## Grand principle

The database is the **single source of truth**, and it stores that truth in its **canonical, non-redundant** form. Anything *derived* (aggregates, search indexes), *presentational* (timezone, human-readable labels), or *volatile / large* (files, dynamic blobs) is kept **out of the normalized core** and lives in a separate structure or another layer.

**When a rule and performance conflict, do not compromise the core** — add a separate, additive, rebuildable structure (a summary table, a search DB) beside it. Never fix a query-speed concern by corrupting the write model.

## Normalization and read scaling

- **3NF by default**: each non-key column depends on the whole key and nothing but the key; no fact stored in two places.
- When reads degrade, **do not** denormalize the canonical tables. Add a separate structure:
  - a **summary / read-model table**, named with a **`summary_` prefix** (`summary_order_listings`), rebuilt from the canonical tables; or
  - an external **search database** (Elasticsearch, etc.) for full-text or faceted search.
- The `summary_` prefix / distinct search DB is a **signal**: "this is derived, not authoritative." Never let application writes treat a summary row as the source of truth.

```
-- Good
orders                 (id, CustomerId, OrderStatusId, ordered_at, ...)  -- canonical truth
customers              (id, name, ...)                                   -- canonical truth
summary_order_listings (id, OrderId, customer_name, status_label, ...)   -- derived, rebuildable

-- Avoid
orders (id, CustomerId, customer_name, ...)  -- drifts when the customer is renamed; two owners
```

## Datetimes

- `created_at` / `updated_at` / `deleted_at` are **audit columns managed by the ORM** — the application does **not** read or write them. They are not model attributes; they come from the shared `...factory.TIMESTAMPS` preset.
- Business logic needing a creation / update / registration time gets a **dedicated, well-named column**: `generated_at`, `modified_at`, `registered_at`, etc.
- Every datetime column is **UTC**, typed `DATETIME(3)` (Sequelize `DATE(3)`) — **millisecond** precision. Timezone conversion is the **application layer's** responsibility, never the database's.

| Name ends with | Meaning |
| --- | --- |
| `_at` | carries a **time of day** (`modified_at`, `trashed_at`, `expired_at`) |
| `_on` | meaning stops at the **calendar date** (`billed_on`, `due_on`) |
| `_at_from` / `_at_to` | **the two ends of a range** — two columns, each keeping the suffix; never one column carrying both ends |

A single column meaning "in effect from this moment" is **not** a range end: name it for the instant — `effective_at`, not `effective_from`.

```
-- Good
articles (id, title, registered_at, created_at, updated_at)  -- business col + framework audit cols
files    (id, modified_at, trashed_at, ...)                  -- instants
invoices (id, billed_on, due_on, ...)                        -- calendar dates

-- Avoid
articles     (id, title, created_at)   -- created_at reused as "registered at" for display
files        (id, modified, ...)       -- granularity unstated
invoices     (id, billed_at, ...)      -- time of day the business never means
price_tables (id, effective_from, ...) -- one instant wearing a range-end suffix
```

## Status / category → master table + key column

- Never a free-form string on the entity. Create a **master table** (`order_statuses`) defining the set, and give the entity a **key column** mapping to it (FK-like `OrderStatusId`, or a stable string key resolved against the master).
- **`ENUM` is discouraged** — reserve it for sets that are **clearly closed and small** and that will not grow, where master-table metadata would be overkill. When unsure, use the master table (the reversible choice).
- The entity's key column is an **FK-like column** (uppercase-initial `OrderStatusId`), **no DB foreign-key constraint**.
- **Name the master table for the classification it holds, in the plural: `*_statuses` for a status set, `*_categories` for a classification set — never `*_types`.** The key column follows the table (`OrderStatusId`, `GranteeCategoryId`; not `GranteeTypeId`). `type` is prohibited as a suffix (collides with the JSDoc type annotation); the one exception is a word borrowed verbatim from an external standard, such as `mimeType`.

### Standard columns of every reference master table

| Column | Role |
| --- | --- |
| `id` | primary key — what the entity's FK-like key column references |
| `name` | **system key**: the stable identifier the application binds to (`'ordered'`). Machine-facing, unique, never renamed once referenced |
| `display_name` | **user-facing label** shown in the UI (`'Ordered'`). Free to reword or localize without touching logic |
| `display_order` | the order in which to present the set in the UI |
| `is_active` | whether the entry is currently selectable / in use |

- Put a **UNIQUE index on `name`** — it is the real key of the set.
- Retire a value by setting `is_active = false`, **never by deleting the row** (deleting orphans historical references; there is no DB FK constraint to stop it). Filter to `is_active = true` when offering choices; keep inactive rows for history.

```
-- Good
order_statuses (id, name, display_name, display_order, is_active)
-- (1, 'ordered',   'Ordered',   1, true)
-- (2, 'cancelled', 'Cancelled', 2, false)   -- retired: kept for history, no longer offered
orders         (id, OrderStatusId, ...)

-- Avoid
orders (id, status VARCHAR(32), ...)   -- 'active' / 'Active' / 'ACTIVE' accumulate
```

## Versioned master (price list, rate table, fee schedule)

A master table managing **many records as one unit that changes over time** is **versioned** as **two separate tables**: a **version table** (one row per published version, carrying the version key, e.g. an `effective_at` datetime — the instant the version takes effect, so `_at`, not `_from`) and the **master-rows table** (the records, each belonging to a version). Read the version effective at a given time; **never edit a published version's rows in place — publish a new version instead.**

Mechanics: `SuiteVersionMixinModel` — the version table `hasMany` the master ("suite") rows, `versionKey` (e.g. `effectiveAt`) selects the version in effect, `findCurrentSuite()` reads it. See the mixin catalog in `hor-sequelize-model`.

```
-- Good
price_tables     (id, effective_at, ...)                       -- version table (one row per version)
price_table_rows (id, PriceTableId, product_key, amount, ...)  -- master rows, immutable per version

-- Avoid
price_list (id, product_key, amount)   -- no history; "which price applied when" is unanswerable
```

## Column types

| Data | Type | Note |
| --- | --- | --- |
| Genuinely dynamic / schemaless values | `JSON` | only for values never queried or joined relationally |
| URL | `TEXT` | not `STRING(n)` — real URLs have no reliable length bound |
| Long content (article body, description) | `TEXT('medium')` (MEDIUMTEXT) | size the column to the content |
| Datetime | `DATE(3)` (UTC) | see Datetimes |
| Status / category | master-table key column | see Status / category |
| Integer | `INTEGER` default; `BIGINT` for ids and anything that accumulates over the table's life | `SMALLINT` / `TINYINT` only with a **specific, permanent bound** (e.g. by definition 0–100), never "to save space" |

- Normalize relational facts into tables; reserve `JSON` for the genuinely schemaless. Normalizable rows stuffed into `JSON` (`tags_json`) hide from joins, indexes and constraints — should be a tags table + join.
- **Never store a file's bytes in the database.** Put the file in **external object storage** (S3, GCS) and keep only a **reference** — the storage key or URL, as `TEXT`.

```
-- Good
settings_json  JSON            -- per-row dynamic structure, never queried relationally
avatar_url     TEXT            -- unbounded in practice
body           TEXT('medium')  -- long article content
quantity       INTEGER
CustomerId     BIGINT          -- id convention
documents (id, OwnerId, storage_key TEXT, content_type, byte_size, uploaded_at)

-- Avoid
avatar_url STRING(255)                  -- silently truncates a long signed URL
view_count SMALLINT                     -- caps at 32767; widening later is an ALTER on a live table
documents (id, OwnerId, file_blob BLOB) -- bloats backups, slows replication
```

## Update history — the archive pattern

Keep the live table holding only the **current** row, and on every save **append** a copy of the row's business attributes — as a new generation — to a **parallel archive table**. Do **not** keep history by piling every revision into the live table (no `is_current` flag, no per-row version numbers).

Mechanics: `BackupMixinModel` + a `*_bk` table (`customer_orders` → `customer_orders_bk`): on `afterSave` it appends the business attributes (**excluding** `id` / `created_at` / `updated_at` / `deleted_at`) plus a `saved_at` generation marker. Pass the mixin from the live model; see the mixin catalog in `hor-sequelize-model`.

```
-- Good
customer_orders    (id, CustomerId, OrderStatusId, ...)                 -- current row only
customer_orders_bk (id, CustomerOrderId, ...business attrs, saved_at)   -- one appended row per save

-- Avoid
customer_orders (id, ..., version_no, is_current)   -- every query filters is_current; table bloats
```
