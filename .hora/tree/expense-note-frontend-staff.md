# expense-note-frontend-staff
<!-- boilerplate: furo-boilerplate-nuxt 2.1.0 -->

Read in place on 2026-09-03. **There is no `CLAUDE.md` in this boilerplate**, so everything below was
read off the tree. This is a cache; on any disagreement the tree wins and this gets rewritten.

## Directory layout

```
app/globals/        furo-env and the other globals
app/graphql/client/ the GraphQL client base classes, plus mutations/ and queries/ to fill
app/restfulapi/     the REST client side
app/vue/contexts/   BaseAppContext.js — the one context base class this project extends
app/shares/         values shared across contexts
pages/              index.vue, and nothing else yet
components/         empty (.gitkeep)
layouts/ middleware/ plugins/ composables/
assets/css/         variables.css and main.css, layered on furo-nuxt's own stylesheets
tests/__tests__/    split jsdom/ and node/
eslint/ jest/       the config pieces
```

## How screens are written

**Furo is OOP: a page is a `.vue` paired with a context class.** `app/vue/contexts/BaseAppContext.js`
is the base every page context extends, and shared logic is a class under the app's own folders —
never a composable and never a bare function.

Nuxt's component auto-registration is configured in `nuxt.config.js`, so a component dropped into
`components/` is registered without an import.

## How it calls the API

`app/graphql/client/` ships the base classes and two directories to fill:

```
BaseAppGraphqlPayload.js        BaseAppGraphqlCapsule.js        BaseAppGraphqlLauncher.js
BaseAppSubscriptionGraphqlPayload.js  BaseAppSubscriptionGraphqlCapsule.js  BaseAppGraphqlSubscriber.js
mutations/  queries/            one Payload/Capsule pair per operation goes here
graphql.config.js               points the tooling at the schema
```

So an operation is a class per side, not an inline query string. **The queries and mutations
directories are empty** — every operation this version needs is written into them.

## Styling

`assets/css/variables.css` and `assets/css/main.css` are this project's own, and they sit on top of
the stylesheets `@openreachtech/furo-nuxt` ships — a `reset, base, furo, app` `@layer` declaration, a
palette of colour scales, z-index custom properties, a native-element reset and a base design. The
`@layer` order is what keeps a project's own rules winning without specificity fights.

## How tests are written

`tests/__tests__/` splits by environment: `jsdom/` for anything that touches the DOM, `node/` for
anything that does not. `tests/setupAfterEnv.js` is the Jest setup file.

## npm scripts

```
dev        the development server (clears the Nuxt cache first, via `cache`)
generate   the static build          build / start   the server build, and running it
test       Jest
lint / l   ESLint
postinstall  nuxt prepare
```

## Middleware

**None.** A frontend row holds neither a DB client nor a Redis client, and no compose file is placed
here.
