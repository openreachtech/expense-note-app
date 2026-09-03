# expense-note-backend
<!-- boilerplate: renchan-boilerplate 1.11.0 -->

Read in place on 2026-09-03. **There is no `CLAUDE.md` in this boilerplate**, so everything below
was read off the tree. This is a cache; on any disagreement the tree wins and this gets rewritten.

## Directory layout

```
app/                globals (`_.js`, `env.js`, `require.js`, `root-path.js`), constants, tools
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
| admin GraphQL | `AdminGraphqlServerEngine` | `/graphql-admin` | read from the engine |
| REST | `AppRestfulApiServerEngine` | `/v1`, `/v2` renderers | read from the engine |

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
