# hor-query-resolver
<!-- hora-skills-ort-renchan 0.1.0 -->
<!-- source: .claude/skills/hor-query-resolver/ -->

**Read the source above whenever this leaves a question open.**

## 1. Placement, class name, registration

```
server/graphql/resolvers/<endpoint>/<actual|stub>/queries/<Operation>QueryResolver.js
```

- `<endpoint>` is the audience/endpoint — here **`staff`**, mapping to
  `server/graphql/StaffGraphqlServerEngine.js`.
- `actual/` = real logic; `stub/` = hardcoded fake data. Both files carry **the same class name
  and the same `schema`**; only the body differs. The stub already exists at
  `server/graphql/resolvers/staff/stub/queries/SignedInStaffMemberQueryResolver.js`, so the actual
  one is `server/graphql/resolvers/staff/actual/queries/SignedInStaffMemberQueryResolver.js`
  (that directory is currently empty — this is the repository's first actual resolver).
- One class per file, PascalCase `<Operation>QueryResolver`, `export default`.
- **Auto-loaded — no manual registration.** The engine loads every file under `.../queries/` by
  directory (`actualResolversPath`). There is no index to edit.
- `static get schema ()` returns the **exact GraphQL Query field name** (camelCase) declared in
  `server/graphql/schemas/staff/*.graphql` → `'signedInStaffMember'`.

Import depth from `.../staff/actual/queries/` (6 levels to the backend root):
`'../../../../../../sequelize/models/StaffMember.js'`; the context is
`'../../../../contexts/StaffGraphqlContext.js'`.

## 2. Member order (fixed)

1. `static get schema ()` — `@override`
2. `static get errorCodeHash ()` — spread `...super.errorCodeHash` first
3. `async resolve ()` — `@override`
4. `validateInput ({ input })`
5. `createInputValidator ({ input, errorHash = this.errorHash })`
6. finders / helpers **in call order** (`buildWhereClause` → `countX` → `findX`); unrelated
   helpers in dictionary order
7. `formatResponse ({ ... })`
8. error creators (`createXxxNotFoundError`), when built through helpers

Then a trailing `@typedef` block after the class: `ErrorCodeHash`, `Input` / `Result` aliases,
`ErrorHash` (`Record<string, RenchanGraphqlErrorCtor>`), `RenchanGraphqlErrorCtor`
(`typeof import('@openreachtech/renchan').RenchanGraphqlError`), the context alias, and any
associated-entity typedef describing an `include` tree.

- **No `constructor` / `static create ()`** unless the resolver needs an extra dependency. The
  framework's `BaseResolver.create()` builds the instance and injects `errorHash` from
  `errorCodeHash`.

## 3. `resolve()` — the fixed pipeline and its signature

`resolve()` receives `{ variables, context, information, parent }`; **destructure only what you
use**, and omit `context` entirely when nothing is read from it.

1. **Validate** — `validateInput()` returns the error **or `null`**; `resolve()` throws it.
2. **Read** — finders (`findX()` / `countX()`).
3. **Throw domain errors** on not-found / empty, via `this.errorHash.Xxx.create()`.
4. **Format** — `return this.formatResponse({ … })`.

```js
/** @override */
async resolve ({
  variables: {
    input,
  },
}) {
  const validationError = this.validateInput({
    input,
  })

  if (validationError) {
    throw validationError
  }

  const user = await this.findUser({
    userId: input.userId,
  })

  if (!user) {
    throw this.errorHash.UserNotFound.create()
  }

  return this.formatResponse({
    user,
  })
}
```

**The skill agrees with the always-on rule** `D:/ORT/rules/graphql-resolvers.md`: input is nested
under `variables`, there is no `args`. Nothing to override here.

- **No transaction, ever.** Queries only read — `.findOne` / `.findByPk` / `.findAll` / `.count`;
  never `db.transaction(...)`. Do not copy the mutation template for a read.
- **`resolve()` orchestrates; the small methods do the work.** No 60-line find, no giant response
  object inlined.
- **Never wrap the read/throw in `try/catch`.** A declared error must propagate so the GraphQL
  layer maps its code; swallowing or re-wrapping hides which code the caller receives.

## 4. A query with no argument (this is `signedInStaffMember`)

The skill's **minimal variant**: when a field takes no `input` (or reads only `context`), drop the
validation wiring **entirely** — no `validateInput()`, no `createInputValidator()`, **no
`*InputValidator` class at all**. Still declare `schema` and `errorCodeHash`.

```js
/** @override */
async resolve () {
  // read, then return the schema shape
}
```

`signedInStaffMember` is exactly this case: the SDL declares
`signedInStaffMember: SignedInStaffMemberResult!` with no argument, `types/StaffGraphQL.d.ts`
deliberately declares no `SignedInStaffMemberInput`, and the customer audience's existing `actual/`
`HealthCheckQueryResolver` is the same arg-less `async resolve ()` shape. **Input validation does
not apply.**

The no-argument form is not a conflict with the always-on rule's `{ variables: { input }, context }`
— that shape presumes an operation that takes an input, and this one has none to destructure.

## 5. Reaching the authenticated caller

`context` carries the authenticated principal and request-scoped providers the engine injects.
The principal lives under the **endpoint's own name** (`context.customer`, `context.employee`, …),
with its associations preloaded.

For the **staff** audience the context is `server/graphql/contexts/StaffGraphqlContext.js`, which
publishes two aliases of the framework's generic members:

| read | is an alias of | meaning |
| :-- | :-- | :-- |
| `context.staffMember` | `#userEntity` | the member-of-staff entity; `null` when the request carried no live access token |
| `context.staffMemberId` | `#userId`, which the framework derives as `userEntity.id` | id of the member of staff |

```js
async resolve ({
  context,
}) {
  const staffMemberEntity = context.staffMember

  // ...
}
```

**The trap the context's own docblock records:** the framework publishes `userEntity.id` as
`#userId`, and therefore as `#staffMemberId`. If `StaffGraphqlContext.findUser()` returns the
**access-token row** rather than the member of staff, `context.staffMemberId` silently becomes the
*token's* id. `findUser` is currently an unimplemented stub returning `null` for every request
(checkpoint 6 of `#sign-in` discharges it, and owes either returning the member of staff or
overriding `#get:userId`). **So do not treat `context.staffMemberId` as trustworthy until that
checkpoint lands** — if the resolver needs the id, prefer reading it off the entity whose shape you
know, and state which entity you assumed.

**Authentication is enforced by the engine before `resolve()` runs** (the `102` standard codes).
The skill is explicit: *"Do not re-check auth here; just read the trusted principal."*

§10's acceptance criterion for this operation — refused without a session, returns nobody — is met
by the engine, not the resolver: `StaffGraphqlServerEngine`'s `schemasToSkipFiltering` deliberately
lists only `signIn` / `signOut` / `renewAccessToken`, so the framework's auth filter refuses
`signedInStaffMember` without a session. Per the skill the resolver may therefore assume a caller
exists.

> ⚠ **The skill does not settle the residual case:** whether to guard a `null`
> `context.staffMember` anyway. Its "do not re-check auth" is about *policy*, not about a null
> principal, and while `findUser` returns `null` the filter refuses the call before `resolve()` is
> reached — so the guard is currently unreachable either way. Decide and record it; a
> `204.Q001.00n` not-found error is the shape available if you want one.

Alias the context type in the typedef block and reference it from the `resolve()` JSDoc:
`@typedef {import('../../../../contexts/StaffGraphqlContext.js').default} StaffGraphqlContext`.

## 6. `errorCodeHash` and domain errors

Spread the parent first, then group this resolver's codes under a one-line family comment.
Code format is **`<family>.<identifier>.<seq>`**.

| family | meaning | raised by |
| :-- | :-- | :-- |
| `203` | invalid input | the `*InputValidator`, through the resolver's `errorHash` |
| `204` | database / not-found | thrown from `resolve()` |
| `205` | business logic / external API | `resolve()` |

- **`<identifier>`** is the **stable per-operation id** — `Q###` for a query, `M###` for a
  mutation. It is already assigned in `server/graphql/resolver-id-hash-staff.js`:
  `signedInStaffMember: 'Q001'`. Keep one resolver on one identifier and add errors sequentially.
- **`<seq>`** (`001`, `002`, …) numbers errors within this resolver, **per family**.
- **Framework standards sit above these and must never be redeclared:** `102` unauthenticated /
  unauthorized / denied-permission, `104` database, `100` unknown, `101`
  concrete-member-not-found.

A real code for this audience and operation:

```js
/** @override */
static get errorCodeHash () {
  return {
    ...super.errorCodeHash,

    // Database Errors (204 prefix)
    StaffMemberNotFound: '204.Q001.001',
  }
}
```

A resolver that throws nothing declares `return { ...super.errorCodeHash }` and nothing else —
that is what both `HealthCheckQueryResolver`s and the `signedInStaffMember` stub do. It still owes
its entry in `resolver-id-hash-staff.js` (already present).

Domain errors are thrown where detected, right after the finder:
`throw this.errorHash.StaffMemberNotFound.create()`. When a resolver throws several, wrap each in a
**creator method** placed after `formatResponse`, defaulting `errorHash`:

```js
createStaffMemberNotFoundError ({
  errorHash = this.errorHash,
} = {}) {
  return errorHash.StaffMemberNotFound.create()
}
```

**Validator ↔ errorHash contract:** every `InvalidXxx` the validator can raise must have a key
here, because `createInputValidator()` forwards `errorHash = this.errorHash`. Not applicable to an
arg-less query.

> Codes must be **unique across the whole `staff/actual/` pool** —
> `tests/__tests__/server/graphql/resolvers/validate-unique-error-code.js` already loads that pool
> and asserts it.

## 7. `formatResponse()`

Yes, the skill uses one, and it is the **only** place that builds the GraphQL output object.
It maps model entities to the schema's field names and **never returns a raw model instance**.

- Rename model columns to schema fields (`staffMember.id` → `staffMemberId`).
- Default missing scalars (`?? ''` / `?? null`) — never leak `undefined`.
- `filter` out records whose required association is missing.
- **Money / decimal → string** via `BigNumber(value).toFixed(2)`, computed into a named `const`
  first, matching the `String!` SDL money type. Never a raw JS number.
- Echo pagination (`limit` / `offset` / `sort` / `totalRecords`) for lists.

> ⚠ **OVERRIDDEN IN THIS PROJECT — do NOT put logic in an object-literal property value.** The
> skill's `formatResponse` samples write `users: users.map(…)` and
> `department: departmentId ? {…} : null` **inside** property values. The always-on rule
> `D:/ORT/rules/javascript-style.md` forbids exactly that: "**Never in a property value:** Method
> calls, Ternary operator, `??` after a chain of 3+ property accesses, Any operator/expression that
> needs a line break", and prescribes the fix — "compute into named consts, then assemble", or
> extract a `generate<Field>()` method. Always-on wins (Q10). **Compute each field into a `const`
> above the `return`, then assemble a literal of bare identifiers.**

**A field-path extractor is not available in this repository.** The skill's `data-access.md`
prescribes `FieldPathValueExtractor` for deep association values, and the always-on
`javascript-style.md` likewise prefers one over long `?.` / `??` chains — but
`@openreachtech/mentsu-field-path-value-extractor` is **not installed** here (the installed
`@openreachtech/*` set is `renchan`, `renchan-env`, `renchan-sequelize`, `mentsu-logger`,
`mentsu-mixin-builder`, `mentsu-random-text-generator`, `eslint-config`, `jest-constructor-spy`).
So: **read the nested value with a plain guarded access assigned to a named `const`** — which
satisfies the property-value rule above and keeps the chain out of the literal. Do not add the
dependency to satisfy a style preference. The same gap applies to `ValueInspector` from
`@openreachtech/mentsu-value-inspector`, which the `buildWhereClause` samples import and which is
likewise absent.

**For `SignedInStaffMemberResult` specifically:** it is
`{ staffMemberId: Int!, name: String!, email: String! }`, but `email` is **not** on the
`StaffMember` model — `StaffMember` declares only `name`, and `email` lives on `StaffMemberSecret`
(`StaffMember.hasOne(StaffMemberSecret)`). So the value needs either an `include` of
`StaffMemberSecret` in a finder or a read off whatever entity the context hands you. **The skill
does not settle which** — it depends on what `StaffGraphqlContext.findUser()` ends up preloading
(checkpoint 6).

## 8. Reading data (compressed — this operation reads one record)

- **One record:** `Model.findByPk(id, { include })` or `Model.findOne({ where, include })`, followed
  by a not-found guard in `resolve()`.
- **A list:** `Model.findAll({ where, offset, limit, order })`. **A count:**
  `Model.count({ where })`.
- Finders are **thin wrappers over one Sequelize call**; the where clause, pagination and includes
  are built by their own small methods so each is independently testable.
- Cast the Sequelize return to your associated-entity typedef with a `/** @type {…} */ ( … )`
  wrapper — model methods return loosely-typed instances.
- `include` tree: a bare model includes the whole association; the object form adds `where` /
  `required` / `as` / nested `include`. `required: true` = INNER JOIN (filters the parent).
  `separate: true` on a `hasMany` that needs its own `order` / `limit`.

full text: `.claude/skills/hor-query-resolver/references/data-access.md`

## 9. Pagination — not used by this operation

`signedInStaffMember` is not a list query; this section is compressed hard.

**The canonical trio, driven from `resolve()`:** `buildWhereClause()` → `countX({ whereClause })`
(total **before** paging, for `totalRecords`) → `findX({ whereClause, pagination })` (applying
`offset` / `limit` / `order`). **Both** the count and the find use the **same** `whereClause`. The
finder destructures `pagination` with a **default sort** so a missing sort still orders
deterministically, and `formatResponse()` **echoes** `limit` / `offset` / `sort` / `totalRecords`
back. Allowed sort columns are enforced by the `*InputValidator`, not the resolver.
`buildWhereClause()` accumulates active filters onto a `conditions` array and returns `{}` when
empty, `{ [Op.and]: conditions }` otherwise; `Op` is imported from `sequelize`.

**`#expense-entry`'s `expenses` query must re-read the source before writing its pager** — the
trio, the default-sort form, the `[Op.like]` / `[Op.or]` / `[Op.in]` + `Model.subquery()` filter
shapes, and the pagination echo are all there in full.
full text: `.claude/skills/hor-query-resolver/references/data-access.md` (§"Pagination: the count +
findAll trio", §"buildWhereClause") and `references/anatomy.md` (§"Full template")

The contract's `Pagination` / `PaginationInput` / `Sort` / `SortInput` already exist in
`server/graphql/schemas/staff/001-common.graphql` and `types/StaffGraphQL.d.ts`, declared and
deliberately unused at 1.0.0. `Sort` is `{ key: String!, direction: String! }` — the framework's own
`RequestSort` shape, **not** the skill's placeholder `{ targetColumn, orderBy }`. Use the declared
contract types.

## 10. What a query must never do

- **Never write.** No `db.transaction(...)`, no `create` / `update` / `destroy`. This is the skill's
  own "No transaction. Queries only read" plus the always-on CQRS rule in
  `D:/ORT/rules/architecture.md` (a mutation returns only its write result; re-read via a separate
  query).
- **Never return a raw model instance** — `formatResponse()` shapes every output.
- **Never `try/catch` a declared error** around the read/throw.
- **Never re-check authentication** — the engine's filter already ran.
- **Never redeclare a framework error code** (`100` / `101` / `102` / `104`).
- **Never leak `undefined`** into the response.
- **Never validate inline** — delegate to a `*InputValidator` (and for an arg-less query, have none
  at all).

## 11. Testing (the resolver-specific part)

- Test path mirrors the resolver **without the `actual/` segment**:
  `tests/__tests__/server/graphql/resolvers/staff/queries/SignedInStaffMemberQueryResolver.js`.
  (The existing stub tests keep their `stub/` segment.)
- Build with the framework factory, **no args**: `const resolver = Resolver.create()` — it injects
  `errorHash` from `errorCodeHash`.
- One top-level `describe('<Resolver>')` → `describe('#member()')` **per public member**: typically
  `#resolve()`, `#formatResponse()`, the finders, and `.get:schema`.
- `#resolve()` gets a happy path asserting the **whole** formatted result, plus a throws-case **per
  error code**, asserted by its code string: `.rejects.toThrow('204.Q001.001')`.
- `#formatResponse()` is pure and DB-free: build input entities with `Model.build({ … })`, call it,
  and `toEqual` the exact GraphQL shape (renamed fields, defaulted scalars).
- `.get:schema` is a plain `test` (the arg-less-static-getter exception).
- A resolver reading `context` gets a **stub context** in the case, shaped like the real principal —
  here `{ context: { staffMember: … } }`.
- Cast deliberately-invalid input as `/** @type {*} */ (…)`.

> ⚠ The skill's error cases carry the code in an **`errorCode`** case field. That field is not in
> the rule-sanctioned set (`params` / `expected` / `factoryParams` / `mock*` / `tally` / `label` /
> `expectedPattern` / `expectedTotalLength`) — put the code in **`expected`**. See
> `.hora/digests/hoc-jest.md` §"Case field names".

full text: `.claude/skills/hor-query-resolver/references/testing.md`

## 12. Not settled by the skill

- Whether to guard a `null` `context.staffMember` when the engine's filter already refuses the call
  (§5).
- Where `SignedInStaffMemberResult.email` is read from — an `include` of `StaffMemberSecret` in a
  finder, or off a context entity that preloads it (§7). Depends on checkpoint 6's `findUser`.
- What to substitute for the uninstalled `FieldPathValueExtractor` / `ValueInspector` is this
  digest's reading of the always-on rules, not the skill's instruction (§7).

---

## Settled by the main session — the three §12 raised

**RULING 1 — guard a null principal, and give it its own error code.** The engine's filter is the
primary guarantee: `signedInStaffMember` is deliberately absent from `schemasToSkipFiltering`, so a
caller with no session is refused before `resolve()` runs, and §10's criterion is met there. **Guard
it in the resolver anyway**, and say in a comment that the filter is the guarantee and this is the
second line.

The reason is specific, not ceremonial. `schemasToSkipFiltering` is a **hand-maintained list**, and
one wrong entry in it silently turns this operation public (Q28 records the same mechanism for a
stub-served field). Without a guard, that misconfiguration surfaces as a null-field GraphQL error or
a crash; with one, it surfaces as a clean refusal that reveals nothing.

**This mirrors a ruling already made at checkpoint 5 and should read consistently with it.** There,
`SessionClerk`'s guard was made the guarantee so the resolver's `isAvailable` pre-check became
defence in depth rather than load-bearing — the pre-check was kept, not deleted. Same shape here:
put the guarantee where it cannot be bypassed, keep the second check, and label which is which.

The guard is directly testable — construct a context whose `staffMember` is null and assert the
refusal — so it is not untestable dead code.

**RULING 2 — the address is read from `staff_member_secrets`, by the resolver.** It is **not** on
`StaffMember`, and §9.1 says why in as many words: "The sign-in address and the password digest are
not here: they sit in `staff_member_secrets` and `staff_member_password_hashes`, so a read of
somebody's name cannot carry a credential in its result set."

So `SignedInStaffMemberResult` needs two reads: the member of staff (already on the context, via
`findUser`) and their current address, by `StaffMemberId`. Put that in its own `find~` method on the
resolver per `architecture.md`'s one-responsibility rule — `find~` reads and returns, nothing else.

**Do not widen `findUser` to fetch the address.** It runs on **every request to this audience**,
including the three operations that never need it, and `SessionClerk` deliberately holds only the
two token models. One operation's field is not a reason to make every request pay a join.

**A member of staff with no current address is a real state here**, not a hypothetical: the
development seeders deliberately include two such rows, because §4 has accounts issued by hand and
Q22 records that the product offers the operator no mechanism — so a half-issued account is the most
likely way one goes wrong. Decide what the operation answers for one, and note that `email` is
`String!` in the contract, so returning null is not available.

**RULING 3 — guarded access into a named `const`, and that is the always-on rule's own fallback.**
`@openreachtech/mentsu-field-path-value-extractor` is not installed here, and the rule that prefers
it is written conditionally: "if your project has a field-path extractor (this repo uses …)". This
project does not, so the preference does not bind, and **the rule still forbids logic in an
object-literal property value** — so compute into named `const`s and assemble the literal from them.
Do not install a package to satisfy a preference for one operation.
