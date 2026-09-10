# hor-graphql-server-engine
<!-- hora-skills-ort-renchan 0.1.0 -->
<!-- source: .claude/skills/hor-graphql-server-engine/ -->

**Read the source above whenever this leaves a question open.**

## Grand principle: one endpoint = one engine, and the engine only declares and wires

Each audience (`customer` / `admin` / a new `staff`) gets **exactly one** `<Audience>GraphqlServerEngine` class under `server/graphql/`, extending `BaseAppGraphqlServerEngine`. The engine **holds no business logic** — no DB queries, no response building, no per-request state. It is routing config + DI wiring (`Share` / `Context`) + cross-cutting policy (auth filter, error mapping, scalars, middleware). Adding an audience never edits an existing engine.

- Per-request state → `Context`. Per-process state → `Share`. Domain work → resolvers.
- Do **not** put logic in `static get config` — a getter returns a value, it does not compute one.
- Structural comments in the generated engine file are English.

## Members of a new `<Audience>GraphqlServerEngine`, in the tree's order

Copy the order of `server/graphql/CustomerGraphqlServerEngine.js` exactly (`AdminGraphqlServerEngine.js` is that file with only the audience names, endpoint, schema/resolver paths and cookie name changed — nothing else).

| # | Member | Required? | For |
| --- | --- | --- | --- |
| 1 | `static get config` | required (framework throws `ConcreteMemberNotFound`) | endpoint routing declaration |
| 2 | `static buildRefreshTokenCookieConfig ()` | repo-local, no `@override` | composes `config.refreshTokenCookie` from the app base + this audience's cookie name |
| 3 | `static get standardErrorCodeHash` | required | framework error name → this app's numeric code |
| 4 | `collectMiddleware ()` | required (instance method — it reads `this.config`) | Express middleware array |
| 5 | `get schemasToSkipFiltering` | optional (base default `[]`) | allowlist of operations that skip the auth filter |
| 6 | `generateFilterHandler ()` | required | the per-operation auth gate |
| 7 | `get visaIssuers` | optional (base default `{}`) | the hooks that build the request's visa |
| 8 | `static get Share` | required | per-process DI class |
| 9 | `static get Context` | required | per-request DI class |
| 10 | `async collectScalars ()` | optional (base default `[]`) | custom scalars this endpoint exposes |

Every one of 1, 3–10 carries `/** @override */`. Also available, both left at their defaults in this tree: `defineOnResolved()` (engine-wide after-resolve hook, default noop) and `passesThoughError()` (default `env.isPreProduction()` — never disable error mapping in production).

Import block of the engine file, in order: `express`, `cors` → `@openreachtech/renchan` (`graphqlUploadExpressWithResolvingContentType`) → `../../app/globals/_.js` (`rootPath`) → `../../app/constants/authConstants.js` → `./BaseAppGraphqlServerEngine.js` → `./contexts/<Audience>GraphqlShare.js`, `./contexts/<Audience>GraphqlContext.js`.

## `static get config` — the routing declaration

```js
/** @override */
static get config () {
  return {
    graphqlEndpoint: '/graphql-staff',
    refreshTokenCookie: this.buildRefreshTokenCookieConfig(),
    staticPath: rootPath.to('public/'),
    schemaPath: rootPath.to('server/graphql/schemas/staff.graphql'),
    actualResolversPath: rootPath.to('server/graphql/resolvers/staff/actual/'),
    stubResolversPath: rootPath.to('server/graphql/resolvers/staff/stub/'),
    postWorkersPath: null,

    /*
     * NOTE: Uncomment the following line to enable Redis PubSub
     *   When disabled, LocalPubSub is used.
     */
    redisOptions: null,
  }
}
```

| Key | Meaning |
| --- | --- |
| `graphqlEndpoint` | URL path this engine serves. **Must be unique** across all engines. Also the path the refresh-token cookie is scoped to. |
| `refreshTokenCookie` | Repo-local key (not in the skill's table). `{ ...this.refreshTokenCookieConfig, name: REFRESH_TOKEN_COOKIE.<AUDIENCE>.NAME }`. |
| `staticPath` | `rootPath.to('public/')`. |
| `schemaPath` | The `.graphql` file **or** a directory of them. This tree uses a single file per audience (`schemas/customer.graphql`). |
| `actualResolversPath` | Real resolver directory, or `null`. |
| `stubResolversPath` | Stub resolver directory, or `null`. |
| `postWorkersPath` | Post-worker directory, or `null` when unused (`null` = post-workers never run). |
| `redisOptions` | Redis PubSub for subscriptions; `null` → in-process `LocalPubSub`. Enable Redis only when another process publishes to subscribers. |

- **Always resolve paths with `rootPath.to(...)`** — never hand-built relative strings.
- Resolve Redis in the **`env` layer** and just reference it; do not branch inside `config`.

## `standardErrorCodeHash` — the error-code scheme

Both existing engines declare this block **identically** (codes are not varied per audience in this tree) — copy it verbatim:

```js
/** @override */
static get standardErrorCodeHash () {
  return {
    Unknown: '100.X000.001',
    ConcreteMemberNotFound: '101.X000.001',
    Unauthenticated: '102.X000.001',
    Unauthorized: '102.X000.002',
    DeniedSchemaPermission: '102.X000.003',
    Database: '104.X000.001',

    CanNotSubscribe: '102.S000.001',
  }
}
```

Shape `<category>.<owner-id>.<serial>`:

| Segment | Meaning |
| --- | --- |
| `100` / `101` / `102` / `104` | category — `100` unknown, `101` missing concrete member, `102` authentication/authorization, `104` database |
| `X000` / `S000` | the **owner id**. `X000` = engine-level (not any one resolver); `S000` = subscription-level. A **per-resolver** code puts the resolver's own id here instead (`M###` mutation / `Q###` query, per `~/.claude/rules/graphql-resolvers.md`, e.g. `203.M018.001`). |
| `.001` … | serial within that owner |

`this.errorHash.<Name>.create()` — used inside `generateFilterHandler()` — is built by the framework from this hash, so a name thrown in the filter must exist here.

## Authentication: `schemasToSkipFiltering` + `generateFilterHandler()` + `visaIssuers`

**What the skip list does, mechanically.** The framework passes `engine.schemasToSkipFiltering` as `ignoredSchemas` to `FilterSchemaHashBuilder`; the filter handler is attached to **every actual schema except the ignored ones**. So an entry in the list means *the filter never runs for that operation* — it is **fully public**. A public operation missing from the list will fail `Unauthenticated`.

**What still happens for a skipped operation.** The Context is built for **every** request regardless: `BaseGraphqlContext.createAsync()` runs `extractAccessToken()` (header `x-renchan-access-token`) → `static findUser()` → `createVisa()` from the engine's `visaIssuers`. So the visa exists and `hasAuthenticated` is still *computed* (`userEntity !== null`) for a skipped operation — nothing **enforces** it. Consequence: a resolver of a skipped operation must not rely on `context.<audience>Id` / `context.<audience>` being present, and must identify its subject some other way (for a session operation, from the refresh-token cookie via `RefreshTokenExpressCookieClerk`).

**For the `staff` audience as specified:** put exactly `signIn`, `signOut`, `renewAccessToken` in `schemasToSkipFiltering`; every other operation then runs the gate below, and with `generateSchemaPermissionHash` returning `null` (all schemas permitted) plus `hasAuthorized` returning `true`, the gate reduces to `hasAuthenticated` — i.e. `findUser()` resolved a user entity from the access-token header. `healthCheck` is the one entry the existing engines carry; keep or drop it per the schema you write.

```js
/** @override */
get schemasToSkipFiltering () {
  return [
    'healthCheck',
  ]
}

/** @override */
generateFilterHandler () {
  return async ({
    variables,
    context,
    information,
    parent,
  }) => {
    const schema = information.fieldName

    const canResolve = context.canResolve({
      schema,
    })

    if (canResolve) {
      return
    }

    if (!context.hasAuthenticated()) {
      throw this.errorHash.Unauthenticated.create()
    }

    if (!context.hasAuthorized()) {
      throw this.errorHash.Unauthorized.create()
    }

    if (!context.hasSchemaPermission({
      schema,
    })) {
      throw this.errorHash.DeniedSchemaPermission.create({
        value: {
          schema,
        },
      })
    }
  }
}
```

- The four checks are a **fixed ladder** in this order with the early return: `canResolve` → `hasAuthenticated` → `hasAuthorized` → `hasSchemaPermission`. All four read `this.visa` on the context; `canResolve` is itself `hasAuthenticated && hasAuthorized && hasSchemaPermission`.
- **No real work in the filter** — allow/deny from the visa only; no queries, no mutation of state.
- **Avoid:** `generateFilterHandler () { return async () => {} }` on a non-stub engine — that disables auth for the whole endpoint. Only a stub engine may do it.
- `schemasToSkipFiltering` is security-critical (check #3 of `hor-security-audit`): only genuinely unauthenticated operations, never a sensitive mutation "to make it work".

**`visaIssuers`** supplies the three hooks the visa is built from. The tree's values (the framework defaults are the same shapes: `hasAuthenticated` → `userEntity !== null`, `hasAuthorized` → `true`, permission hash → none):

```js
/** @override */
get visaIssuers () {
  return {
    hasAuthenticated: async ({ expressRequest, userEntity, engine }) => userEntity !== null,
    hasAuthorized: async ({ expressRequest, userEntity, engine }) => true,
    generateSchemaPermissionHash: async ({ expressRequest, userEntity, engine }) => null,
  }
}
```

`generateSchemaPermissionHash` returning `null` means **all schemas permitted**; returning `{ someSchema: true, other: false }` restricts per operation (an absent key denies). Widen `hasAuthorized` when the audience has an active/blocked distinction to enforce.

*(In the tree these callbacks are written with each param chopped onto its own line, and the `null` return wrapped in a `/** @type {Record<string, boolean> | null} */` cast carrying an inline `@example` — copy that file's formatting verbatim.)*

## `collectMiddleware()` and `collectScalars()`

Both existing engines declare `collectMiddleware()` themselves, identically: `cors({ origin: '*' })`, `express.json({ limit: '10mb' })`, `express.static(this.config.staticPath)`, `graphqlUploadExpressWithResolvingContentType({ maxFileSize: 10000000, maxFiles: 10 })`, `express.urlencoded({ extended: true, verify })` setting `req['rawBody']`. Copy that array; raise `limit` only if this endpoint accepts larger uploads. `async collectScalars ()` returns `[]` in this tree — add only the scalars the schema actually uses.

## The Share / Context pair

`server/graphql/contexts/` holds one pair per audience plus the shared `BaseAppGraphqlContext.js` and `tools/RefreshTokenExpressCookieClerk.js`. A new audience adds `StaffGraphqlShare.js` + `StaffGraphqlContext.js`. **Each endpoint pairs its own Share + Context**; never point one audience's engine at another's Context.

**`<Audience>GraphqlShare`** — per-**process**, extends `BaseGraphqlShare`, built once at boot. Its whole body is the factory:

```js
static async createAsync ({
  config,
}) {
  const broker = this.createBroker({
    config,
  })

  return this.create({
    env: this.generateEnv(),
    broker,
  })
}
```

with `@typedef {Parameters<GraphqlType.ShareCtor['createAsync']>[0]} <Audience>GraphqlShareAsyncFactoryParams` at the end of the file. Resolvers / post-workers reach it as `context.share`.

**`<Audience>GraphqlContext`** — per-**request**, extends the repo's `BaseAppGraphqlContext` (which extends the framework `BaseGraphqlContext`). What it declares:

1. `static async findUser ({ expressRequest, accessToken, requestedAt })` → the authenticated user entity or `null`. This **is** the access-token check: look the token up on the audience's access-token table and include the principal. Both existing contexts are still `// TODO: Must fulfill this method.` delegating to `super.findUser(...)` (which returns `null`), so a real `staff` audience must implement it; the shape to follow is the JSDoc `@example` in `CustomerGraphqlContext.js`.
2. `get <audience> ()` → `this.userEntity`, and `get <audience>Id ()` → `this.userId` — audience-named aliases, each with a `@returns` and a resolver-side `@example`.

**`canResolve` / `hasAuthenticated` / `hasAuthorized` / `hasSchemaPermission` are NOT implemented on the Context.** They are inherited from the framework `BaseGraphqlContext` and delegate to `this.visa`, which is built from the **engine's `visaIssuers`**. Override `visaIssuers` on the engine, not these methods.

`BaseAppGraphqlContext` already provides everything a cookie-based session needs, so a new Context adds none of it: `get config` (the engine config, typed `AppGraphqlConfig` = `GraphqlType.Config & { refreshTokenCookie }`), `get cookieHeader`, `get expressResponse` (reachable at `expressRequest['context'].res`).

**A two-token audience (short-lived access token on the header + rotating refresh token in an httpOnly cookie)** differs only in wiring, all of it on the Engine and in constants:

- add `refreshTokenCookie: this.buildRefreshTokenCookieConfig()` to `config`, plus the `static buildRefreshTokenCookieConfig ()` method returning `{ ...this.refreshTokenCookieConfig, name: REFRESH_TOKEN_COOKIE.STAFF.NAME }`;
- add a `STAFF: { NAME: 'staff_refresh_token' }` entry to `constants/authConstants.cjs` — a **separate name per audience**, so the browser never sends another audience's cookie here; `DOMAIN` is deliberately absent, and the cookie's path is the engine's `graphqlEndpoint` (derived by the clerk, not declared);
- shared cookie attributes (`lifetimeDays` from `env.AUTH_REFRESH_TOKEN_TTL_DAYS`, default 14; `secure` from `env.AUTH_COOKIE_SECURE !== 'false'`; `sameSite: 'lax'`; `httpOnly: true`) stay in `BaseAppGraphqlServerEngine.refreshTokenCookieConfig` — change them there, **never in a Context**;
- all cookie reading/writing lives in `RefreshTokenExpressCookieClerk`, built from the context by the session resolvers only (`signIn` / `signOut` / `renewAccessToken`). The Context stays a plain DTO.

An audience with **no** refresh cookie omits the `refreshTokenCookie` config key, the `buildRefreshTokenCookieConfig()` method, the `authConstants` import and the constant entry; its credential check then lives entirely in `findUser()`.

full text on the cookie half: `.hora/digests/hor-cookie-authentication.md` and `.claude/skills/hor-cookie-authentication/`

## Booting in `server/index.js`

**This repository's `server/index.js` starts every server by hand — there is no directory scanning and no engine registry.** A new audience is dead code until an entry is added there. The existing pattern, verbatim:

```js
import CustomerGraphqlServerEngine from './graphql/CustomerGraphqlServerEngine.js'
import AdminGraphqlServerEngine from './graphql/AdminGraphqlServerEngine.js'

/*
 * Bind to loopback only: the app servers sit behind a reverse proxy (see docs/reverse-proxy), so
 * they must not accept connections from other network interfaces.
 */
const LOOPBACK_HOST = '127.0.0.1'

await activate()

GraphqlServerBuilder.createAsync({
  Engine: CustomerGraphqlServerEngine,
})
  .then(builder =>
    builder.buildHttpServer()
      .listen(3900, LOOPBACK_HOST)
  )
```

- Add the `import` beside the others, then one `GraphqlServerBuilder.createAsync({ Engine })` → `.then(builder => builder.buildHttpServer().listen(<port>, LOOPBACK_HOST))` block. Always pass `LOOPBACK_HOST`.
- **Pick a port not already used**: `3900` customer GraphQL, `5800` admin GraphQL, `8001` `AppRestfulApiServerEngine`.
- `createAsync({ Engine })` calls the engine's `createAsync`, which builds the `Share` via `Share.createAsync({ config })` and constructs the engine; `buildHttpServer().listen(port)` mounts it.

## Stub engines (not present in this tree)

The skill's pattern: each audience may also get a `Stub<Audience>GraphqlServerEngine` on its own endpoint (`…-stub`) and port, with `actualResolversPath` pointed at the **stub** directory, `schemasToSkipFiltering` `[]` and a **noop** `generateFilterHandler()`. **This repository has no `Stub*GraphqlServerEngine` class and boots none** — the actual engines merely declare `stubResolversPath`. Auth-off is acceptable only on a separate stub port serving fake data; never let that config reach a real endpoint.

## Not settled — decide in the main session and record

- **Where the shared members live.** The skill (§1) says to lift endpoint-common settings into `BaseAppGraphqlServerEngine` and keep concrete engines to `config` + `Share` + `Context` + auth policy. This tree does the opposite: `collectMiddleware()` and `standardErrorCodeHash` are **duplicated verbatim** in both concrete engines, and the app base holds only the refresh-token cookie config. Copying `CustomerGraphqlServerEngine.js` matches the tree; lifting matches the skill. Pick one before writing `StaffGraphqlServerEngine.js`.
- **Whether a new audience varies the error codes.** The skill does not define the `<category>.<owner>.<serial>` shape at all, and both engines use identical `X000` codes — so nothing settles whether `staff` gets its own owner segment or reuses `X000`. Reusing `X000` is what the tree shows.
- **The port for the `staff` endpoint** — the existing numbering (3900 / 5800 / 8001) implies no rule; choose one and record it.
- **`healthCheck` in the skip list** — carried by both existing engines; whether `staff`'s schema declares it is a schema decision.
