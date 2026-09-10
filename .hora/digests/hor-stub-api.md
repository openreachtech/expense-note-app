# hor-stub-api
<!-- hora-skills-ort-renchan 0.1.0 -->
<!-- source: .claude/skills/hor-stub-api/ -->

**Read the source above whenever this leaves a question open.**

## Grand principle: a stub returns hardcoded literals only — no logic of any kind

Everything a stub's `resolve()` returns is a **hardcoded literal**. No DB access, no service
calls, no validation, no computation, no conditionals, no `map`/`filter`/`reduce` over input. The
**single exception** is the pagination slice (§4).

- **Schema-accurate** = the returned shape matches the SDL **exactly**: every non-null field
  present with the correct type, nullable fields present or explicitly `null`, list fields real
  arrays. A stub that omits a field forces the frontend to discover the gap against the real API —
  the one failure a stub exists to prevent.
- **Echo input back only where the schema result mirrors it** (a mutation returning the id it was
  given). Echoing is reading a field; *transforming* input is logic and belongs to the real
  resolver.

```js
// Avoid — computing/branching in a stub
const normalizedCode = input.postalCode.replace('-', '') // computation
return {
  shippingAddressId: normalizedCode.length > 0
    ? 1003
    : null, // a stub never branches
}
```

## 1. Placement & naming (this repository, `staff` audience)

```
expense-note-backend/server/graphql/resolvers/staff/stub/queries/<Operation>QueryResolver.js
expense-note-backend/server/graphql/resolvers/staff/stub/mutations/<Operation>MutationResolver.js
```

- File name **equals the class name**: `SignInMutationResolver.js` → `class SignInMutationResolver`.
- **The stub carries the same class name as the actual resolver it will be swapped for** — the skill
  states the real resolver "later lives under `…/actual/…` with the **same class name and
  interface**", and this repository already shows the pair:
  `customer/stub/queries/HealthCheckQueryResolver.js` and
  `customer/actual/queries/HealthCheckQueryResolver.js` are identical in shape.
- `queries/` vs `mutations/` is **organizational only**. `DeepBulkClassLoader` recurses the whole
  pool path and skips dotfiles (so `.directorykeeper.cjs` is inert); the operation kind comes from
  the base class's `static get operation ()` (`'Query'` / `'Mutation'`), not the folder.
- The engine already points at the tree: `StaffGraphqlServerEngine.config.stubResolversPath =
  rootPath.to('server/graphql/resolvers/staff/stub/')`.

## 2. How renchan chooses stub vs actual — actual wins, per schema field, automatically

Verified in `node_modules/@openreachtech/renchan/lib/server/graphql/resolvers/GraphqlResolversBuilder.js`
(`buildResolverHash()`):

```js
const schemas = [...new Set([
  ...Object.keys(this.actualResolverSchemaHash),
  ...Object.keys(this.stubResolverSchemaHash),
])]

const resolver =
  this.actualResolverSchemaHash[it]
  ?? this.stubResolverSchemaHash[it]
```

Both pools are loaded (`createAsync` → `buildResolverSchemaHash` once per path) and keyed by
`resolver.schema` (`ResolverSchemaHashBuilder#buildSchemaHash`).

- **Adding an `actual/` resolver for a field automatically supersedes the stub. The stub does not
  have to be deleted** — the `??` never reaches it. Two files with the same `schema`, one per pool,
  is the framework's intended state.
- **But a stub-served field is served with NO authentication filter.** `filterSchemaHash` is built
  from `extractSchemas({ schemaHash: actualResolverSchemaHash })` — **actual only** — and the
  wrapper calls `await filter?.(envelope)`. A field that exists only as a stub gets
  `filter === undefined`, so `generateFilterHandler()` never runs and `Unauthenticated` /
  `Unauthorized` / `DeniedSchemaPermission` cannot fire. `signedInStaffMember` is deliberately
  absent from `schemasToSkipFiltering` because the filter is what must refuse it without a session
  — **as a stub it is a public endpoint regardless.** Do not treat a stubbed operation as
  demonstrating its authentication requirement.

## 3. Class shape

Query stub — extend `BaseQueryResolver`; mutation stub — extend `BaseMutationResolver`, both from
`@openreachtech/renchan`. `schema` is a **`static get`** returning the camelCase GraphQL operation
name (declared explicitly, though `BaseResolver` would otherwise derive it from the class name).

```js
import {
  BaseMutationResolver,
} from '@openreachtech/renchan'

/**
 * Stub resolver: signIn mutation.
 *
 * @extends {BaseMutationResolver}
 */
export default class SignInMutationResolver extends BaseMutationResolver {
  /**
   * get: Operation name.
   *
   * @override
   * @returns {string}
   */
  static get schema () {
    return 'signIn'
  }

  /**
   * get: Error code hash. Kept empty in a stub.
   *
   * @override
   * @returns {Record<string, string>}
   */
  static get errorCodeHash () {
    return {
      ...super.errorCodeHash,
    }
  }

  /**
   * Resolve. Returns hardcoded, schema-accurate data.
   *
   * @override
   */
  async resolve ({
    variables: {
      input: {
        email,
        password,
      },
    },
    context,
  }) {
    return {
      staffMemberId: 1,
      accessToken: 'stub-access-token-signin-0001',
    }
  }
}
```

`resolve()` destructures `{ variables: { input }, context }` — input nested under `variables`, no
`args`. **The skill and the always-on `D:/ORT/rules/graphql-resolvers.md` agree here; no override
is needed.**

**An operation with no input takes no argument** (the SDL declares `signOut: SignOutResult!`, with
no empty input type). The skill: "If the query takes no input, drop the `input` destructuring and
keep just `resolve ()` or `resolve ({ context })`." So, for the three argument-less staff
operations:

```js
/**
 * Resolve. Returns hardcoded, schema-accurate data.
 *
 * @override
 */
async resolve () {
  return {
    signedOut: true,
  }
}
```

Shapes the four staff stubs owe (from `schemas/staff/002-sign-in.graphql`, all fields non-null):

| Operation | Kind | Argument | Returned literal shape |
| --- | --- | --- | --- |
| `signIn` | Mutation | `input: SignInInput!` | `{ staffMemberId: Int, accessToken: String }` |
| `signOut` | Mutation | none | `{ signedOut: Boolean }` |
| `renewAccessToken` | Mutation | none | `{ accessToken: String }` |
| `signedInStaffMember` | Query | none | `{ staffMemberId: Int, name: String, email: String }` |

Result fields are flat on the returned object — `SignedInStaffMemberResult` has no wrapper key.

## 4. Paginated query stub (not needed for `#sign-in`; owed by `#expense-entry`)

Hardcode `const all<Things>` of **≥ 10 records, every value unique**, then return the page:

```js
const totalRecords = allBillingHistories.length
const billingHistories = allBillingHistories.slice(offset, offset + limit)

return {
  billingHistories,
  pagination: {
    limit,
    offset,
    sort,
    totalRecords,
  },
}
```

- **`slice(offset, offset + limit)`** — never `splice` (it mutates the source).
- `totalRecords` is the array's `.length`, **never a hardcoded number**.
- This slice + `.length` is the only computation any stub may perform.

## 5. Error codes in a stub

**A stub declares `static get errorCodeHash ()` but keeps it empty** — `{ ...super.errorCodeHash }`
and nothing else. Real codes are added at migration (§6). A stub throws nothing, so it owns no
code.

- The resolver id is separate and **already allocated**: `server/graphql/resolver-id-hash-staff.js`
  holds `signedInStaffMember: 'Q001'`, `signIn: 'M001'`, `signOut: 'M002'`,
  `renewAccessToken: 'M003'`. The always-on rule requires an entry for every query and mutation
  "even one that never throws", so the stub needs no change there.
- Prefixes for later: **`203`** input-validator, **`204`** database, **`205`** external —
  `'203.M001.001'`.
- `tests/__tests__/server/graphql/resolvers/validate-unique-error-code.js` scans **`actual/` pool
  paths only**, so stub hashes are outside its uniqueness check.

## 6. Migrating a stub into `actual/` (checkpoint 6)

Preserve the interface, swap only the body — the frontend must notice nothing but real data.

1. **Keep the class name, `static get schema ()`, and `static get errorCodeHash ()`.** Move the
   file from `…/staff/stub/…` to `…/staff/actual/…`.
2. **Fill `errorCodeHash` with real codes**, `...super.errorCodeHash` first
   (`InvalidPostalCode: '203.M004.001'`).
3. **Replace the literal return with real work** — validate input, run DB/transaction logic through
   Sequelize models, format the response with **the same shape the stub returned**. Structure it
   per `hor-query-resolver` / `hor-mutation-resolver`: extracted `validateInput` / transaction
   callback / `formatResponse`, not everything inlined in `resolve()`.
4. Throwing an `errorCodeHash` key surfaces the code added in step 2.

Moving the file is the skill's instruction, and §2 means leaving a copy behind would still work —
but it hides which pool serves the field, and it leaves the operation's filter status ambiguous to
a reader. Move it.

## 7. Testing a stub

**The skill says nothing about testing a stub** — no test file, no example, no checklist item.
What the surrounding conventions imply:

- A stub writes nothing, so under `.hora/digests/hor-backend-testing.md` §1 and
  `D:/ORT/rules/testing.md` any test belongs in **`tests/__tests__/`, mirroring the source path**:
  `tests/__tests__/server/graphql/resolvers/staff/stub/mutations/SignInMutationResolver.js`. Never
  `tests/_orders/` (nothing to order, no barrel entry).
- Style is the always-on `D:/ORT/rules/testing.md`: class `describe` repeated per member,
  `.get:schema` / `#resolve()` notation, AAA with one Act, `test.each` by default (a plain `test()`
  for the arg-less `static get schema ()`), one `toEqual` against a full `expected`.
- Precedent in this repository: **neither existing `HealthCheckQueryResolver` (stub or actual) has
  a test at all.** `tests/__tests__/server/graphql/` covers engines and contexts only.

**Decide in the main session:** whether the four staff stubs get tests, or whether a stub — being
literals whose only contract is the SDL — is left untested until it becomes an `actual/` resolver.
The skill does not rule, and repository precedent is "untested".

## Not settled by the skill — decide and record

- **`errorCodeHash` on a query stub.** The skill declares it for mutations only and is silent for
  queries; both of this repository's query resolvers (`customer` and `admin`
  `HealthCheckQueryResolver`) declare it, spread `super.errorCodeHash`, and keep it empty. Ruling
  needed for `SignedInStaffMemberQueryResolver`.
- **JSDoc density.** The skill's samples carry a full block per member (description, `@override`,
  `@returns`) and `@extends {BaseX}` on the class; the two existing HealthCheck resolvers carry
  only `/** @override */`. The always-on `D:/ORT/rules/jsdoc.md` requires a short description on
  every member, which agrees with the skill — so follow the skill's fuller form and read the
  HealthCheck terseness as pre-existing boilerplate.
- Whether a stub is registered anywhere beyond the file itself: nothing else is required — the
  engine's `stubResolversPath` plus deep loading is the whole wiring.

## Finishing checklist

- [ ] Every returned field a **hardcoded literal** (no DB, no service calls, no validation, no
      conditionals, no `map`/`filter`/`reduce`)?
- [ ] Returned shape matches the SDL **exactly** — all non-null fields present, correct types?
- [ ] File under `staff/stub/{queries,mutations}/`, class `<Operation>QueryResolver` /
      `<Operation>MutationResolver`, file name matching?
- [ ] `static get schema ()` returns the camelCase operation name?
- [ ] `static get errorCodeHash ()` spreads `super.errorCodeHash` and stays empty?
- [ ] No-input operation declared with no argument — `async resolve ()`?
- [ ] Paginated stub: ≥ 10 unique records, `slice(offset, offset + limit)`, `totalRecords` = `.length`?
- [ ] Input echoed back only where the schema result mirrors it (never transformed)?
- [ ] Migrating: same class name / `schema` / `errorCodeHash`, file moved to `actual/`, real codes
      filled in, response shape unchanged?
