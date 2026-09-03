# expense-note-backend
<!-- boilerplate: renchan-boilerplate 1.11.0 -->

Read in place on 2026-09-03. **There is no `CLAUDE.md` in this boilerplate**, so everything below
was read off the tree. This is a cache; on any disagreement the tree wins and this gets rewritten.

## Directory layout

```
app/                `globals/` (`_.js`, `env.js`, `require.js`, `root-path.js`), `constants/`,
                    `session/` (the session classes — see "The session and cookie layer")
constants/          shared `*.cjs` constants
sequelize/          `_.js` (the activator), `config.cjs`, models/, migrations/, seeders/
server/             index.js, graphql/, restfulapi/
tests/              __tests__/ (no DB write), _orders/ (DB write), _live/, setup-after-env.js
types/              model.d.ts, renchan.d.ts, graphql.d.ts
public/             static files served by the engines
pm2.config.cjs      process definitions
```

Empty directories are held open by `.directorykeeper.cjs`, not `.gitkeep`.

## How servers are split

`server/index.js` is the single entry point, and it starts three servers by hand — no scanning:

| Server | Engine | Endpoint | Port |
|---|---|---|---|
| customer GraphQL | `CustomerGraphqlServerEngine` | `/graphql-customer` | 3900 |
| admin GraphQL | `AdminGraphqlServerEngine` | `/graphql-admin` | 5800 |
| REST | `AppRestfulApiServerEngine` | `/v1`, `/v2` renderers | 8001 |

Every server binds `127.0.0.1` — they sit behind a reverse proxy. `pm2.config.cjs` runs one app,
`GraphQL API` → `./server/index.js`.

**The boilerplate ships `customer` and `admin`. `expense-note` declares one server, `staff-graphql`**
(spec 2.1), so implementation adds a `staff` audience: a schema file, a resolvers directory pair, an
engine, and one entry in `server/index.js`.

## How things get registered — by directory scanning

**This is the highest-value fact, and the answer is the good one.** An engine's `static get config ()`
names paths, and renchan scans them:

```js
schemaPath: rootPath.to('server/graphql/schemas/customer.graphql'),
actualResolversPath: rootPath.to('server/graphql/resolvers/customer/actual/'),
stubResolversPath: rootPath.to('server/graphql/resolvers/customer/stub/'),
postWorkersPath: null,
```

**Dropping a resolver file into the directory registers it.** There is no aggregation file to append
to, so no checkpoint has to touch a shared list. A post-worker directory has to be named before post
-workers work — it ships `null`.

## The existing GraphQL schema

**One `.graphql` file per audience**, at `server/graphql/schemas/<audience>.graphql` — not a numbered
directory of files. Both shipped files declare a `healthCheck` query and nothing else.

## The session and cookie layer

**This is the part the first pass missed, and stage 4 of the spec is what needed it.** The row
already ships the two-token cookie session — a short-lived access token plus a rotating refresh
token — so the convention exists here rather than having to be invented.

```
app/session/
  SessionClerk.js               the single window onto a session's tables. Every find / save /
                                update / delete across the access-token and refresh-token
                                tables goes through it, and callers depend on nothing else
  SessionCredentialGenerator.js generates the session key and the tokens
  BaseSessionResult.js          the base of the two result objects below
  SavingSessionResult.js        the outcome of starting or rotating a session — never throws
  RevokingSessionResult.js      the outcome of revoking one

server/graphql/contexts/
  BaseAppGraphqlContext.js      a plain DTO. It exposes `cookieHeader` and the engine config,
                                and deliberately holds no cookie logic
  tools/RefreshTokenExpressCookieClerk.js
                                every bit of cookie logic — reading, setting and clearing the
                                refresh-token cookie. A session resolver builds one from the
                                context it is handed; every other resolver never touches it
```

**`SessionClerk` takes its tables injected, not imported** — `AccessTokenModel` and
`RefreshTokenModel` are constructor arguments, which is how one implementation serves every
audience. So a new audience needs the two token tables and nothing more of this layer.

**The cookie's name, lifetime and `Secure` flag are composed in one place.**
`BaseAppGraphqlServerEngine` holds `refreshTokenCookieConfig` (lifetime from
`refreshTokenCookieLifetimeDays`, `secure` from `usesSecureRefreshTokenCookie`), and each
audience engine adds only its own cookie name through `buildRefreshTokenCookieConfig()`. A
maintainer changes the defaults in the base engine alone.

**What this layer expects of the database is not written here.** The credential lives in a
**five-table cluster** — the entity, a secret holding the sign-in identifier, a password hash,
an access-token table and a refresh-token table — rather than as columns on the entity. **The
column shapes come from the equipped cookie-authentication skill's own token-model reference,
not from reading `SessionClerk`**: the clerk shows which fields it passes
(`sessionKey`, `generatedAt`, `tokenHash`), and the skill is what states nullability, the
unique constraints and the rotation columns (`usedAt`, `revokedAt`, `expiredAt`). Read the
skill before designing against this layer.

## Naming conventions

Read off what ships: `<Action><Operation>Resolver.js` (`HealthCheckQueryResolver.js`), engines
`<Audience>GraphqlServerEngine.js`, a shared `BaseApp*` prefix for the per-project base classes
(`BaseAppGraphqlServerEngine`), contexts `<Audience>GraphqlContext.js` and `<Audience>GraphqlShare.js`.

## Existing model definitions

`sequelize/models/` and `sequelize/migrations/` ship empty (`.directorykeeper.cjs` only).
`sequelize/_.js` is the activator `server/index.js` awaits before building any server;
`sequelize/config.cjs` holds the per-environment dialect.

## How tests are written

`tests/__tests__/**` mirrors the source path for anything that does not write to the DB;
`tests/_orders/**` for anything that does; `tests/_live/**` for what needs the live DB, pulled in by
a `_.test.js` barrel. `tests/setup-after-env.js` is the Jest setup file.

## npm scripts

```
dev            the server
test / jest    Jest            test:live   the live-DB suite
lint / l       ESLint
db:setup  db:seed:master  db:seed:dev  db:refresh  db:drop  db:teardown
```

`test.sh` and `test-live.sh` sit at the root beside them.

## A local end-to-end environment

**None ships.** There is no `e2e/` directory. The middleware is what `/hora` wrote:
`docker-compose.development.yml` and `docker.sh` (`start` / `stop`), MariaDB 10.5.12 in the default
profile, everything else behind a profile.
