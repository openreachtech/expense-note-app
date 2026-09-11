# hor-mutation-resolver
<!-- hora-skills-ort-renchan 0.1.0 -->
<!-- source: .claude/skills/hor-mutation-resolver/ -->

**Read the source above whenever this leaves a question open.**

## 1. The class shape

- **Path:** `server/graphql/resolvers/<endpoint>/actual/mutations/<Operation>MutationResolver.js`.
  `actual/` = real logic; sibling `stub/` = fixed data for the same schema. One class per file.
- **Class name** = `<Operation>MutationResolver`, `<Operation>` = the schema field in PascalCase
  (`signIn` → `SignInMutationResolver`) — the four `staff/stub/` files already carry the names the
  `actual/` ones must reuse.
- **Base class:** `BaseMutationResolver` from `@openreachtech/renchan` (queries:
  `BaseQueryResolver`).
- **`static get schema ()`** returns the camelCase schema field, `/** @override */`. The base can
  derive it from the class name — **always declare it explicitly** anyway; it is the single
  greppable link between schema and class.

```js
import {
  BaseMutationResolver,
} from '@openreachtech/renchan'

export default class UpdateArticleMutationResolver extends BaseMutationResolver {
  /** @override */
  static get schema () {
    return 'updateArticle'
  }

  // ... errorCodeHash, resolve(), helpers ...
}
```

### `resolve()` signature — the skill agrees with the always-on rule

The skill's signature is `{ variables: { input }, context }`, input nested under `variables`, **no
`args`** — identical to `D:/ORT/rules/graphql-resolvers.md`. No override needed, and the existing
`staff/stub/mutations/SignInMutationResolver.js` already destructures exactly that.

`context` supplies `now` (the request timestamp — **use it, never `new Date()`**, so every row of
one request shares one instant) and the actor. For this audience that is `StaffGraphqlContext`:
`context.staffMemberId` / `context.staffMember`, plus `context.config`, `context.cookieHeader`,
`context.expressResponse` off `BaseAppGraphqlContext`.

### Method order (verbatim)

1. `static get schema ()` — `@override`
2. `static get errorCodeHash ()` — `@override`
3. *(optional)* `constructor` → `static create ()` → `static createXxx ()` factory helpers — only
   when a dependency must be injected (§5)
4. `async resolve ()`
5. `createInputValidator ({ input })`
6. `validateInput ({ input })`
7. `generateTransactionCallback ({ input, context })`
8. other instance methods — `findXxx` / `buildXxxAttributes` / `updateXxx`, placed **next to their
   caller** (call-related adjacent; unrelated ones alphabetical)
9. `formatResponse ({ ... })`
10. `@typedef` blocks at the bottom (context type, entity types)

### The fixed four-step `resolve()`

```js
/**
 * @param {{
 *   variables: { input: server.graphql.user.UpdateArticleInput }
 *   context: UserGraphqlContext
 * }} params
 * @returns {Promise<server.graphql.user.UpdateArticleResult>}
 */
async resolve ({
  variables: {
    input,
  },
  context,
}) {
  const validationError = this.validateInput({
    input,
  })

  if (validationError) {
    throw validationError
  }

  const callback = this.generateTransactionCallback({
    input,
    context,
  })

  const article = await Article.beginTransaction(callback)

  return this.formatResponse({
    article,
  })
}
```

**`resolve()` never grows a fifth concern**: validate → build the callback → run it in one
transaction → format. A post-commit side effect is an **explicit extra line between step 3 and
step 4**, never logic hidden inside the four calls.

- `createInputValidator ({ input })` builds the `*InputValidator` with
  `{ errorHash: this.errorHash, input }`; `validateInput ({ input })` runs it and returns the error
  or `null`. Keep both as named methods so tests can stub them.
- **Split of duty: shape/format → the validator; existence / ownership / state → the transaction
  callback.** The resolver inlines no value check.
- `formatResponse()` returns the **minimum identifying result** — usually just an id (plus a
  timestamp for a delete). The frontend re-queries for rendered data. A `formatResponse` that
  maps / sorts / joins means logic leaked out of the callback or a Query's job crept in.
- **Which model to call `beginTransaction` on:** any model in the write; pick the operation's
  primary entity. The transaction spans all models regardless.
- Comments in the `.js` are **English**; domain notes may match the surrounding language.

## 2. `errorCodeHash` and throwing

Spread `...super.errorCodeHash` first, then this resolver's codes grouped by category with a blank
line and a comment per group. Code string = **`<category>.<M###>.<seq>`**.

| category | meaning | names |
|---|---|---|
| `203` | invalid input — 1:1 with the validator's predicates (`InvalidTitle` ↔ `isValidTitle`) | `InvalidXxx` |
| `204` | DB / state, thrown **inside** the transaction callback | `XxxNotFound`, `NotAllowedToXxx`, `CurrentXxxIsSameAsTheNewOne` |
| `205` | auth | `InvalidCredentials` |

- **`<M###>` is the resolver's own id**, shared by every code in that resolver. In this repo it is
  read from `server/graphql/resolver-id-hash-staff.js`, which already fixes
  **`M001` signIn, `M002` signOut, `M003` renewAccessToken, `Q001` signedInStaffMember**. Every
  query and mutation holds an entry there, **including one that declares no code of its own**.
- **`<seq>`** is a zero-padded running number within the resolver (`001`, `002`, …). A gap from a
  removed code is fine — **never renumber an existing code**; clients may reference it.
- Name = the reason, not the field alone. No `info` / `data` / `error` suffixes.

Real codes for this audience (`signIn` = `M001`):

```js
/** @override */
static get errorCodeHash () {
  return {
    ...super.errorCodeHash,

    // Invalid Input Errors
    InvalidEmail: '203.M001.001',
    InvalidPassword: '203.M001.002',

    // Authentication Errors
    InvalidCredentials: '205.M001.001',
  }
}
```

**`errorCodeHash` → `errorHash`:** the base's `create()` runs `buildErrorHash({ errorCodeHash })`,
which maps each `{ Name: code }` to a `RenchanGraphqlError` subclass on `this.errorHash` (also
`this.Error`). You declare the hash of **strings**; you throw the **class**:
`throw this.errorHash.InvalidCredentials.create()`. Input errors are returned by the validator and
rethrown at the top of `resolve()`; state / auth errors are thrown inside the transaction callback
so it rolls back. A resolver that overrides `create()` for DI (§5) **must still call
`this.buildErrorHash({ errorCodeHash })` itself** and pass the result as `errorHash`.

Optional: an `ErrorHash` `@typedef` naming the errors the resolver throws — documentation only.

## 3. An operation with no input at all

**The skill does not cover this case** — every example takes an `input`. What the repository has
already settled (`schemas/staff/002-sign-in.graphql`, and the `signOut` / `renewAccessToken` /
`signedInStaffMember` stub docblocks): the spec declares `SignOutInput()` and
`RenewAccessTokenInput()` with no fields, and **the convention gives an operation with no input no
argument at all**, so no empty input type is declared and there is nothing to destructure.

What follows for the `actual/` resolver:

- `resolve ({ context })` — `context` only, since the credential arrives in the refresh-token
  cookie that `context` carries. (An operation needing neither is the stubs' `async resolve ()`.)
- **Step 1 of the four disappears with the input**: no `createInputValidator`, no `validateInput`,
  no `203.*` code. Steps 2–4 stay as written. Nothing else about the skeleton changes.
- These two operations owe the credential check the auth filter is not doing for them (both name
  themselves in the engine's `schemasToSkipFiltering`), so their guards live where §4 puts every
  state guard — inside the transaction callback, thrown as `204.*` / `205.*`.

## 4. Transactions — one per request, and a domain class that opens its own

`generateTransactionCallback({ input, context })` returns an **`async transaction => { ... }`**
closure holding **every find and save for the request**; `resolve()` hands it to
`Model.beginTransaction(callback)`, which opens a managed transaction, invokes the closure with the
live `transaction`, commits on success and rolls back on a throw.

1. **Every query inside takes the same `transaction`** — `findOne` / `findAll` / `create` / `save`
   / `update` / `destroy`. A query without it runs outside and defeats the point. Splitting a
   `find` from its dependent `save` across two transactions lets a concurrent request interleave
   (`find1 → find2 → save1 → save2`).
2. **Guard state inside the closure and throw to roll back** — existence, ownership, no-op
   ("nothing changed" → `CurrentXxxIsSameAsTheNewOne`). Early `if (...) throw`, one guard per
   `if`, no `else`.
3. **`.set()` + `.save()` to update, `Model.create(attributes, { include, transaction })` to
   insert — never `.update()`**: the backup mixin needs the full row and a partial `.update()` can
   drop fields. FK columns start uppercase (`CreatedByUserId`). Nested create puts the child
   object under the association name and lists the child model in `include`.
4. Destructure `input` and `context` **in the parameter list** so the closure body reads flat.
   Bulk existence guard: dedupe with `Set`, count, compare — `map` / `filter` / `Set`, never
   `forEach` / `for`.
5. A large create payload goes into a separate `buildXxxAttributes({ ... })` returning the plain
   object, placed next to the closure that calls it.
6. **Work that must observe the committed result** (cache, read model, notification) runs in
   `resolve()` **after** `beginTransaction` returns — never inside the closure, whose rows a
   rollback discards.

```js
generateTransactionCallback ({
  input: {
    articleId,
  },
  context: {
    now,
    userId,
  },
}) {
  return async transaction => {
    const article = await this.findArticle({
      articleId,
      transaction,
    })

    if (!article) {
      throw this.errorHash.ArticleNotFound.create()
    }

    article.set({
      LastModifiedByUserId: userId,
      lastModifiedAt: now,
    })

    return /** @type {*} */ (
      article.save({
        transaction,
      })
    )
  }
}
```

### Consuming `SessionClerk`, which manages its own transaction

`app/session/SessionClerk.js` is the single window on session data and **never throws**: each
public write (`saveSession`, `rotateSession`, `revokeSession`) takes an **optional `transaction`**
and reports the outcome as a `*SessionResult` carrying `{ response, error }`. The skill has no case
for a collaborator like this; these are the two consumption shapes it leaves you, and the clerk's
own docblocks fix which is which:

| The resolver's write is… | Call it… | Roll back by… |
|---|---|---|
| only the session (nothing else to write) | **without** `transaction` — the clerk self-resolves one (`invokeSaveSession` / `invokeRotateSession` / `invokeRevokeSession`) | already handled inside the clerk |
| session **plus** another row in the same request | **with** the `transaction` the closure received — the clerk joins it | the resolver, by `throw`ing inside its own callback |

- **Read the result; never expect an exception.** `saveSession` / `revokeSession` → check
  `result.hasError()`. **`rotateSession` → check `result.shouldRollBack()`, not `hasError()`**: the
  refusal of a refresh token that is no longer presentable *is itself a write* (the whole series
  revoked, `result.hasRevokedSeries()`), and rolling back on `hasError()` would discard the
  revocation and leave a stolen cookie's series live.
- **Error naming is the resolver's job, explicitly left to it by the clerk.** Translate
  `result.error` into `this.errorHash.<Name>.create()` — never rethrow or surface the clerk's own
  `Error` or its message. The clerk deliberately carries **one** message for all three dead-token
  states because §10 requires the three refused identically; `signIn`'s two refusals (no account /
  wrong password) are likewise **one** code, not two.
- On success read the pair off the result (`accessTokenEntity`, `refreshTokenEntity`, and the plain
  `refreshToken`, which exists only here because the table stores a digest) and hand
  `refreshToken` to the cookie clerk (§7).

## 5. What belongs in the resolver, and what is pushed out

**In the resolver:** the four-step `resolve()`; the `*InputValidator` wiring; the transaction
callback and its state guards; naming every error; `formatResponse`; and — for these three
operations — driving the refresh-token cookie (§7).

**Pushed out** — reached through injected collaborators, never `new`ed or `.create()`d inline in a
method that does other work:

| Concern | Class |
|---|---|
| credential hashing / constant-time compare | `app/session/PasswordEncipher.js` |
| every session find / save / rotate / revoke | `app/session/SessionClerk.js` |
| §7's counted windows (failed sign-ins per address, renewals per series) | `app/tools/rateLimit/limits/SignInFailureRateLimit.js`, `AccessTokenRenewalRateLimit.js` — `await limit.hasReachedMaxEventCount({ ... })` |

**Inject a collaborator that is nondeterministic or side-effecting** (token / random generator,
encipher, external client) so a test can replace it. Do **not** inject models (imported and used
statically) or the validator (built in `createInputValidator`, which a test stubs). The triple, in
the method order of §1:

```js
constructor ({
  passwordEncipher,
  ...remainingParams
}) {
  super(remainingParams)

  this.passwordEncipher = passwordEncipher
}

static create ({
  passwordEncipher = this.createPasswordEncipher(),
  errorCodeHash = this.errorCodeHash,
} = {}) {
  const errorHash = this.buildErrorHash({
    errorCodeHash,
  })

  return new this({
    passwordEncipher,
    errorHash,
  })
}

/**
 * @returns {PasswordEncipher}
 */
static createPasswordEncipher () {
  return PasswordEncipher.create()
}
```

- Each dependency **defaults to its own `static create<Dep>()`** — production stays a bare
  `.create()`, and the creator is the one seam a test overrides.
- `...remainingParams` passed to `super()` carries `errorHash`; **never swallow it**.
- Reach an injected tool through a **thin one-line instance method** rather than scattering
  `this.<dep>.foo()` — it names the intent and gives tests one more seam:

```js
async verifyPassword ({
  originalPassword,
  passwordHash,
}) {
  return this.passwordEncipher.compare(
    originalPassword,
    passwordHash
  )
}
```

## 6. Heavy or slow work, and side effects after success

A resolver answers **synchronously** and must stay fast. Slow or uncertain work (external API, AI
call, bulk records, file generation) → **enqueue a job and return immediately**; the work runs in a
Worker. Decide with the execution-placement convention and implement with the renchan-job-bullmq
convention. **When in doubt, push to the Worker.**

The skill's own placement for a post-mutation side effect is the explicit line between step 3 and
step 4 of `resolve()` (§4.6) — it says nothing about post-workers. The always-on
`D:/ORT/rules/graphql-resolvers.md` routes a side effect that **should run only after a mutation
succeeds** (notification email, provisioning, a follow-up record, starting a job) to a **GraphQL
post-worker** under `server/graphql/post-workers/**` (`BaseGraphqlPostWorker`,
`static get schema ()` matching the mutation, an `onResolved` hook guarded on success) — that
directory does not exist in this repo yet. **`#sign-in` checkpoint 7 makes the placement decision,
with the placement skill rather than by eye** (the pattern `#data-model` checkpoint 7 set); do not
settle it inside a resolver.

## 7. Setting a cookie from a mutation

**The skill is silent on cookies** — it has no case for a resolver that writes a response header,
and `hor-cookie-authentication`'s digest covers only the database half. What the repository
settles, from the docblock of `server/graphql/contexts/tools/RefreshTokenExpressCookieClerk.js`:

> A resolver that manages a session — sign-in, sign-out, renew access token — **builds a clerk from
> the context it receives in `resolve()`** and drives the cookie through it; **every other resolver
> never touches it.**

So the resolver reaches it by construction from `context`, not through an injected dependency and
not off the context as a property:

```js
createRefreshTokenCookieClerk ({
  context,
}) {
  return RefreshTokenExpressCookieClerk.create({
    context,
  })
}
```

(Wrapping the `.create()` in its own method is the always-on collaborator-seam rule; place it below
its caller.) Its three public moves:

| Operation | Call |
|---|---|
| `renewAccessToken`, `signOut` — read the presented credential | `extractRefreshToken()` → `string \| null`. **Calling it is not authenticating**; only these two may treat the value as a credential |
| `signIn`, `renewAccessToken` — hand the new token to the browser | `saveRefreshTokenCookie({ refreshToken })` (the plain token off the `*SessionResult`) |
| `signOut` — remove it | `clearRefreshTokenCookie()` |

Cookie name, lifetime, `Secure` / `SameSite` / `HttpOnly` and path come from the engine config the
context carries — **changed in the Engine class alone**, never in a resolver. The cookie write is
not part of any DB transaction, so it belongs where §4.6 puts post-commit work: **after
`beginTransaction` returns**, before `formatResponse`. And the refresh token never appears in a
response body — `SignInResult` returns `staffMemberId` + `accessToken` only.

## 8. Testing — placement split

| Test hits the DB? | Location | Discovered how | Order |
|---|---|---|---|
| **No** (pure / mocked) | `tests/__tests__/` mirroring the resolver's source path | jest matches **every `.js`** under `__tests__/` | none (independent) |
| **Yes** (real transaction) | `tests/_orders/<Category>/mutations/<Resolver>.js` | imported by the category's `_.test.js` | **guaranteed** within the category |

- **`__tests__/` covers:** `.get:schema`, `.get:errorCodeHash`, `#createInputValidator()` /
  `#validateInput()` (validator **mocked** via `jest.spyOn`), `#formatResponse()` (pure), and the
  DI factory + `static createXxx()` seams.
- **`_orders/` covers:** `#generateTransactionCallback()` (run through
  `Model.beginTransaction(callback)`) and `#resolve()` (full flow) against the seeded DB — a valid
  input persists the row; a missing row throws the `204.*` and rolls back; ownership throws
  `NotAllowedToXxx`; a no-op throws `CurrentXxxIsSameAsTheNewOne`. Separate branches with
  `describe`, never `if`.
- **`<Category>` is the model the resolver *writes to*, not the operation** — a `publishArticle`
  that writes an `ArticleStatus` row goes under `tests/_orders/ArticleStatus/mutations/`.
- The write file is named `<Resolver>.js` (**not** `*.test.js`) so jest ignores it on its own.
  **Add one `import './mutations/<Resolver>.js'` line to the category's `_.test.js`** in the right
  position, or the test never runs.

Everything else — `test.each`, `describe` structure, unique explicit-fake data, AAA, `jest.spyOn`
— follows the always-on jest rules and `hoc-jest` / `hor-backend-testing` unchanged.

## Not settled — for the main session to decide and record

1. **`205` means two different things.** The skill assigns `205.*` to **auth**
   (`InvalidCredentials`); the always-on `D:/ORT/rules/graphql-resolvers.md` labels `205` an
   **external** error and says nothing about where an auth error goes. Not a contradiction of a
   prohibition, so Q10 does not dispose of it. `signIn`'s single credential refusal and
   `renewAccessToken`'s single dead-token refusal need a category before checkpoint 6 writes them.
2. **The `_orders` category for these three.** All three write to `staff_member_access_tokens`
   and `staff_member_refresh_tokens` through `SessionClerk`, and the rule ("the model the resolver
   writes to") does not pick between two tables written together.
3. **Where the `signIn` `*InputValidator` lives.** The skill defers the validator's own shape to
   the resolver-validator convention (skill `hor-resolver-validator`, no digest yet), and this
   repository has **no validator tree at all** — the skill's import path
   (`app/validator/forResolver/<endpoint>/mutations/`) disagrees with the always-on
   `input-validators.md` (`app/tools/validator/resolvers/<audience>/{mutations|queries}/`). Pick
   one before writing `SignInInputValidator`.
4. **The resolver's `resolve()` JSDoc form.** The skill writes the params type inline
   (`@param {{ variables: { input: … } context: … }}`); the existing
   `staff/stub/mutations/SignInMutationResolver.js` uses
   `@param {GraphqlType.ResolverInput<{ input: … }>}`, and **no `GraphqlType` namespace is declared
   anywhere in `types/`** (`types/graphql.d.ts` is an empty `namespace graphql`). One of the two has
   to be made real.

---

## Settled by the main session — the points §"Not settled" raised

**RULING 1 — `204` carries this feature's refusals; `205` stays external-only, and the skill's
`205 = auth` reading is rejected.**

The always-on `graphql-resolvers.md` states the convention outright — "`203` = input-validator
error, `204` = database error, `205` = external error" — and **its own worked example uses `204` for
a not-found**: `OrderNotFound: '204.M018.001'`. That is the precedent, and Q10 makes the always-on
rule the winner where the two texts differ.

So, concretely:

| refusal | family | why |
|---|---|---|
| `signIn` — no account, or wrong password | **`204`** | one code for both, because §10 requires them indistinguishable. Both are "the database did not yield a credential that verified", which is what `OrderNotFound` is an instance of |
| `renewAccessToken` — expired, revoked, already spent, or no cookie | **`204`** | one code for all four, per §10 and §10's fourth renewal criterion. `SessionClerk` already returns a single indistinguishable error for the three it can see |
| malformed input | `203` | the validator's, already ruled in `hor-resolver-validator.md` |

**A gap in the convention, recorded so `#expense-entry` does not re-decide it.** A **rate-limit
refusal** ("too frequent", §7) fits none of the three families: the input was valid, no external
service was called, and although the count comes from a database read the refusal is not a database
*error*. It goes under **`204`** with its own sequence number and a name that says what it is, as
the nearest fit — **not** a newly invented `206`, because the families are fixed by an always-on
rule and inventing one diverges from it silently. The honest statement is that the ORT convention
has no family for a policy refusal.

**Engine-level codes are a separate tier and are not to be reused.** `StaffGraphqlServerEngine
.standardErrorCodeHash` declares `100`/`101`/`102`/`104` (`Unauthenticated: '102.X000.001'` and so
on) with an `X000` owner segment meaning "not a specific resolver". Those are the framework's.
A resolver never redeclares one.

**RULING 2 — the validator lives at `app/tools/validator/`, and it is being written by another unit
of this same checkpoint.** Already ruled in `.hora/digests/hor-resolver-validator.md`: the base at
`app/tools/validator/BaseInputValidator.js`, the concrete at
`app/tools/validator/resolvers/staff/mutations/SignInInputValidator.js`. Import from there, not from
the skill's `app/validator/forResolver/`. Its entrypoint is **`validateInput()`, which returns the
error or `null`** — it does not throw — so the resolver reads:

```js
const validationError = validator.validateInput()

if (validationError) {
  throw validationError
}
```

**Only `signIn` has a validator.** `signOut` and `renewAccessToken` take no argument at all, so
there is no input to validate and none is to be written for them.

**CORRECTION — `GraphqlType` is NOT undeclared, and nothing needs fixing.** The digest flagged the
stubs' `@param {GraphqlType.ResolverInput<…>}` as naming a namespace declared nowhere, on the
grounds that the project's `types/graphql.d.ts` is an empty `namespace graphql`. **That is the wrong
file.** `GraphqlType` is declared by the framework, in
`node_modules/@openreachtech/renchan/types/graphql.d.ts` — `namespace GraphqlType` at line 42, with
`ResolverInput<V>` at line 119, alongside `Config` and `ShareCtor`. The boilerplate's own
`BaseAppGraphqlContext.js` and the three `*GraphqlShare.js` files already reference
`GraphqlType.Config` and `GraphqlType.ShareCtor`, so it is established precedent rather than a
phantom.

**Keep using it.** The project's empty `types/graphql.d.ts` is an unrelated boilerplate stub, left
alone deliberately at checkpoint 3 — the same way `types/model.d.ts` was — because this project
declares its GraphQL types in `types/StaffGraphQL.d.ts` under `namespace server.graphql.staff`, per
the always-on rule.
