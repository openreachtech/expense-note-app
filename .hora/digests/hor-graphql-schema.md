# hor-graphql-schema
<!-- hora-skills-ort-renchan 0.1.0 -->
<!-- source: .claude/skills/hor-graphql-schema/ -->

**Read the source above whenever this leaves a question open.**

## Grand principle: the SDL is a derived contract

The operation name drives the type names one-to-one (`<Operation>Input` / `<Operation>Result`), non-null `!` is the default unless a value is genuinely optional, each business domain gets its own numbered file, and each audience keeps its own schema. **Do not invent ad-hoc names, nullable-by-default fields, or a shared "misc" file** — a reader who knows the operation name must be able to predict the file, the type names, and the nullability without opening anything.

- Resolvers are matched to operations by name (`static get schema()`), and JSDoc types reference `<Operation>Input` / `<Operation>Result` directly.
- SDL `!` only guards *presence*. Value-level checks (ranges, formats, enum membership) belong to `hor-resolver-validator` — do not weaken the SDL to "validate later", and do not assume `!` makes a validator unnecessary.

## 1. Where a schema lives and how it loads — check the engine first

Each audience is loaded by its own `*GraphqlServerEngine.js`, and **how the schema is loaded can differ by audience**. Before adding or editing a file, check that audience's engine `schemaPath`:

| `schemaPath` points at | Verbatim rule |
| --- | --- |
| **Folder path** (e.g. `server/graphql/schemas/customer/`) | "the folder's numbered files are merged into one schema at load time. Follow the numbered-file conventions below." |
| **Single file path** (e.g. `server/graphql/schemas/admin.graphql`) | "edit that one file. A sibling folder of the same audience name may exist but be **unwired** — adding files there changes nothing until the engine's `schemaPath` points at it." |

**So yes — `schemaPath` may point at a directory.** Both forms are valid; the engine is the source of truth for which is in force.

```js
// server/graphql/CustomerGraphqlServerEngine.js — the source of truth for what loads
static get config () {
  return {
    // ...existing config...
    schemaPath: rootPath.to('server/graphql/schemas/customer/'),
  }
}
```

- **Never put a type for one audience in another audience's folder.** Even identical-looking types (e.g. `Pagination`) are declared per audience, because the audiences' contracts evolve independently.

> The skill states the merge only as "merged into one schema at load time" — it does **not** name the loader. Verified in the tree (not stated by the skill): `@openreachtech/renchan/lib/server/graphql/schemas/SchemaFilesLoader.js#loadIntegratedSchema()` does `fs.stat(schemaPath)` → `isFile()` ? `readFile` : `readdir` + concatenate every file's raw text (each prefixed with a generated `## <filename>` banner), and `GraphqlSchemaBuilder` feeds the result to `makeExecutableSchema` from `@graphql-tools/schema`, which merges same-named types. That is what makes repeated `type Query` blocks legal, and why the `NNN-` prefix (readdir order) controls concatenation order.

## 2. File organization — numbered, per-domain (folder form)

```
server/graphql/schemas/customer/
  001-common.graphql          # scalars, Pagination, Sort, shared enums
  002-auth.graphql
  003-order.graphql
  004-billing.graphql
```

- Filename is `NNN-<domain>.graphql`. Domain casing (kebab-case vs camelCase) can vary between existing files — **match the casing of the sibling files** in the folder you are editing.
- **One domain per file.** Add a new numbered file for a new domain rather than growing an existing one.
- The first file (`001-common.graphql`) holds shared plumbing (scalars, `Pagination`, `Sort`, shared enums); every later file is one business domain.

## 3. Custom scalars — declared once, at the top of the common file

Custom scalars are declared **only** in that audience's `001-common.graphql`, at the very top, one per line, no `!`:

```graphql
scalar BigNumber
scalar DateTime
scalar Upload
```

- Do not re-declare a scalar in any other file. Just reference it (`createdAt: DateTime!`).
- Scalars can differ across audiences — declare only the scalars the audience uses, matching the sibling audiences' style rather than copying their scalar list.

**Registration is out of scope for this skill** — it never mentions `collectScalars()`. That method belongs to `hor-graphql-server-engine` §6: "`async collectScalars()` → the custom GraphQL scalars this endpoint exposes (`DateTimeScalar`, `BigNumberScalar`). Return only what the schema uses." full text: `.claude/skills/hor-graphql-server-engine/SKILL.md#6-middleware-scalars-error-codes`

## 4. Query / Mutation blocks — colocated per domain file

Do **not** keep one giant root `Query` / `Mutation`. Each domain file opens its own `type Query` and `type Mutation` block listing only that domain's fields; the loader merges them.

```graphql
# 003-order.graphql
type Query {
  orders(input: OrdersInput!): OrdersResult!
  orderDetail(input: OrderDetailInput!): OrderDetailResult!
}

type Mutation {
  cancelOrder(input: CancelOrderInput!): CancelOrderResult!
  archiveDeliveredOrders: ArchiveDeliveredOrdersResult!
}
```

- Every domain file writes a plain **`type Query`** / **`type Mutation`**. The skill shows this form only and **never mentions `extend type Query`** anywhere — do not introduce `extend`.
- A field with no input takes no argument (`archiveDeliveredOrders: ArchiveDeliveredOrdersResult!`).
- Every operation returns a non-null payload type (`...Result!`).

## 5. Type naming — `<Operation>Input` and `<Operation>Result`

| Kind | Name | Example |
| --- | --- | --- |
| Input | `<Operation>Input` | `cancelOrder` → `CancelOrderInput` |
| Payload | `<Operation>Result` | `orders` → `OrdersResult` |

- Nested / domain object types are plain nouns (`OrderItem`, `ProductImage`, `Status`).
- Sub-summary types carry a `Summary` suffix (`OrderSummary`).
- One operation, one input type, one result type — do not share an input or result type between operations even when the shapes currently coincide; shared types couple operations that will diverge.
- Avoid invented payload names (`OrdersArgs` / `OrdersPayload`) — they break the SDL → resolver → type chain.

## 5.1 Field naming — the SDL borrows the database's vocabulary

A field carries the same name as the column it exposes, so renaming a concept is one change across the migration, the model, and the schema.

- Datetime fields end with `At`; date-only fields end with `On`. A range keeps the suffix and adds `From` / `To` (`modifiedAtFrom` / `modifiedAtTo`).
- **Never expose `updatedAt` / `createdAt`.** A business time is its own named field: `modifiedAt`, `generatedAt`, `registeredAt`.
- A classification field is `xxxCategory`, not `xxxType`; the one exception is a word borrowed verbatim from an external standard (`mimeType`).
- A `sort.targetColumn` allow-list uses these same field names.
- **A single-character key is prohibited** — in a field, an argument, or an input type. Name the key for what it holds (`searchQuery`, `sortKey`, `pageNumber`), never `q` / `s` / `n` / `p`.

## 6. Non-null `!` — the default

`!` is applied to nearly every field and argument. A field is non-null unless the value is **genuinely optional**; nullability is deliberate, not the fallback.

```graphql
type Order {
  id: Int!
  customerId: Int              # nullable: guest orders have no customer
  items: [OrderItem!]!         # non-null list of non-null elements
  modifiedAt: DateTime!
}
```

- Lists are `[T!]!` when the list is required; use `[T!]` only when the whole list is genuinely optional.
- Optional inputs drop the `!` and carry a short comment stating why the null is allowed (`avatarUrl: String # allow null`).

## 7. Money / decimal fields — `String!`, decimal-as-string

Money and decimal amounts are typed **`String!`** — not `Float` (loses precision) and not `BigNumber` (reserved for non-money large integers such as byte sizes). Resolvers emit them via `BigNumber#toFixed(2)`.

```graphql
type OrderItem {
  quantity: Int!
  unitPrice: String! # Decimal as string for precision
  totalPrice: String!
}
```

## 8. Pagination type shape

A list query returns its rows plus a `Pagination` object; the input carries a `PaginationInput`. **Both live in `001-common.graphql`** — the shared-plumbing file, above/before the per-domain files, never duplicated into a domain file.

```graphql
type Pagination {
  limit: Int!
  offset: Int!
  sort: Sort
  totalRecords: Int!
}

input PaginationInput {
  limit: Int!
  offset: Int!
  sort: SortInput
}

type OrdersResult {
  orders: [OrderSummary!]!
  pagination: Pagination!
}

input OrdersInput {
  pagination: PaginationInput!
  filters: OrderFiltersInput   # optional filters
}
```

- `sort` / `SortInput` are **nullable** on the pagination pair; `limit` / `offset` / `totalRecords` are non-null.
- The `Sort` shape can differ across audiences — match the pagination/sort types already used in the audience folder you are editing; never copy one audience's shape into another.
- Enums for filters/sort live in the domain file that uses them, or in `001-common` when shared.
- Enums are written **one member per line**, never inline (`enum SortDirection { ASC DESC }` is forbidden).
- Resolver-side validation of `limit` / `offset` / `sort` values is the shared pagination validator in `hor-resolver-validator`.

## 9. Mirror every SDL type into `types/<Audience>GraphQL.d.ts`

Defining a type in the SDL is **only half the job** — declare the matching TypeScript interface in the audience's `types/<Audience>GraphQL.d.ts` in the same change.

```typescript
declare global {
  namespace server.graphql.customer {
    interface CancelOrderInput {
      orderId: number
    }

    interface CancelOrderResult {
      canceledOrderId: number
    }
  }
}
```

- Name the interfaces exactly `<Operation>Input` / `<Operation>Result`.
- Keep fields in sync field-for-field: GraphQL `!` → required property, nullable → `?`.
- Resolvers reference them via JSDoc (`@param {{ input: server.graphql.customer.CancelOrderInput }}`, `@returns {Promise<server.graphql.customer.OrdersResult>}`), so a type present in the SDL but missing from the `.d.ts` fails type-check.

## 10. Match the surrounding file's style

"Section banners and member ordering are per-audience habits, not one global rule — some folders use light banners (`# ===== Order Mutation Input Types =====`), others use full `#### QUERY` / `#### MUTATION` / `#### TYPE` / `#### INPUT` blocks with dictionary-order rules stated in a header comment. **Follow whichever style the surrounding file already uses**, including any stated ordering rule."

## NOT SETTLED BY THIS SKILL — decide and record

1. **Folder-of-numbered-files vs one flat file when opening a NEW audience.** The skill presents both as valid (§1) and says "check the engine first" — but a new audience has no engine yet, and the skill gives **no rule for choosing**. Its grand principle and §2 lean to the numbered per-domain folder ("each business domain gets its own numbered file"; merging domains into one file "makes it impossible to see which audience a change affects"), while its own single-file example (`schemas/admin.graphql`) is stated neutrally. The existing tree is two flat files (`server/graphql/schemas/customer.graphql`, `server/graphql/schemas/admin.graphql`) wired as `rootPath.to('server/graphql/schemas/customer.graphql')` — so "follow the sibling audiences" and "follow the grand principle" point in opposite directions.
2. **Banner style and member ordering for a NEW audience.** §10 resolves this only by deference to "the surrounding file", which does not exist for a first file. The skill names the two observed styles (light `# ===== … =====` banners, or `#### QUERY` / `#### MUTATION` / `#### TYPE` / `#### INPUT` blocks with a dictionary-order header comment) but does not pick one. full text: `.claude/skills/hor-graphql-schema/SKILL.md#10-match-the-surrounding-files-style`
3. **Where `type Query` / `type Mutation` go inside a single-file audience.** §4's "colocated per domain file" rule presupposes the merged-folder layout; the skill says nothing about the internal layout of a flat file beyond §10's "match the surrounding file".
4. **Scalar registration** (`collectScalars()`) — not this skill's; owned by `hor-graphql-server-engine` §6 (see §3 above).
