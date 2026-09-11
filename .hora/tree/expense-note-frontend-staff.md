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

**Corrected at `#sign-in`'s checkpoint 12. The previous text here was wrong in a way that hid a
defect, so what it said is kept below rather than deleted.**

It read: that this project's two stylesheets "sit on top of the stylesheets
`@openreachtech/furo-nuxt` ships — a `reset, base, furo, app` `@layer` declaration, a palette of
colour scales, z-index custom properties, a native-element reset and a base design", and that the
`@layer` order is what keeps the project's own rules winning.

**Three things in that are false for this repository:**

1. **`@openreachtech/furo-nuxt` 2.x ships ZERO `.css` files** — counted, not inferred. The palette,
   the z-index properties and the semantic colours are `@openreachtech/furo-vue`'s, a package this
   repository did not have until checkpoint 12 added it (Q42). The sibling `crm-kit-frontend` loads
   a reset from `furo-nuxt/lib/assets/css/0100.reset.css`, but it is on furo-nuxt **1.x**; that file
   does not exist in 2.x.
2. **There is no `@layer` declaration anywhere in this repository.** `main.css` is seven lines of
   iOS input sizing and declares none; furo-vue's own token files declare none either. The
   `reset, base, furo, app` layering described is `crm-kit-frontend`'s, from its own
   `0000.crm-kit-layers.css`.
3. **Nothing was sitting on top of anything.** Until checkpoint 12, `nuxt.config.js` loaded only the
   app's two files, so every `--color-*` the components read resolved to nothing.

### What is actually true now

```
css: [
  '@openreachtech/furo-vue/lib/assets/css/furo.css',   <- the library's tokens, FIRST
  '~/assets/css/variables.css',                         <- the app's own, still empty
  '~/assets/css/main.css',                              <- 7 lines, iOS input font-size
]
```

`furo.css` is the library's documented "single public entry" and imports, in order: palette colour
scale → z-index → semantic colour → semantic dimension → component overrides → editor content. It is
loaded first so the app's own files override it rather than the reverse.

**`assets/css/variables.css` is still an empty `:root {}`**, deliberately — the boilerplate's comment
says what a colour should be is the application's decision. The app declares its own semantic
properties there when a screen needs one, two-tier (`--palette-*` then `--color-*`) per
`05-frontend.md` §6-2, on top of furo's.

**Why this mattered rather than being a documentation nit.** `--color-ring` is the only focus
indicator `FuroButton` has: its stylesheet removes the native outline and rebuilds the ring from
that property. Undefined, it removed the outline and rebuilt nothing — no visible focus for a
keyboard user, against a stated WCAG 2.2 AA target. An undefined custom property is not an error,
it is an empty value, so nothing failed and nothing warned.

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

## Middleware (the infrastructure sense)

**None.** A frontend row holds neither a DB client nor a Redis client, and no compose file is placed
here.

**The heading is the spec's word, not Nuxt's, and the two do not mean the same thing.** §8's table
is headed `Middleware | Version | profile | Purpose` and lists MariaDB — that is the sense here.
Nuxt route middleware is a different thing entirely and this repository ships two of it; see below.
The overload is worth naming because it has already been misread once, at `#sign-in`'s checkpoint 10,
as this section being stale.

## What the boilerplate already ships, and what is load-bearing

Read off the tree at `#sign-in`'s checkpoint 10. The layout block above names these directories but
not their contents, and one of them decides this feature's route.

| File | What it does |
|---|---|
| `middleware/000.gateway.global.js` | **Load-bearing for every feature.** A global route middleware: if `AccessTokenClerk.create().existsToken()` it passes, otherwise it redirects to `` `${SIGN_IN_PATH}?redirect=${to.fullPath}` `` — and `SIGN_IN_PATH` is a module constant reading `'/sign-in'`, carrying a `// TODO: should be moved to configuration`. **This is why `#sign-in`'s screen is at `/sign-in` and not somewhere chosen.** It is also what makes §10.2's "every other screen sends somebody here when theirs has gone" already true rather than something a later feature wires |
| `middleware/010.pageTitle.global.js` | Reads `FuroMeta.create({ routeTo }).pageTitle` and falls back to `'Furo Nuxt'`, so a page sets its title through `definePageMeta` |
| `plugins/000.furo.js` | Reads `ENDPOINT_URL`, `WEBSOCKET_URL` and `RENCHAN_RESTFUL_API_BASE_URL` off `runtimeConfig.public` |
| `layouts/default.vue` | A bare `<slot />`. **The only layout.** The `hof-nuxt` digest expects auth pages to take a `gateway` layout; this repository has none, so nothing can use one until somebody writes it |
| `composables/useRedirect.js` | Reads `?redirect=` off the query and navigates. **A bare exported function with inner function declarations** — which is what this document's "How screens are written" section rules out. See the open question raised at checkpoint 10; it becomes a decision at checkpoint 16, which is what wires the post-sign-in redirect |

**`pages/index.vue` is still the boilerplate stub** — an empty template and a `<!-- TODO: fulfill here -->`. Route `/` renders nothing, though the gateway redirects an unauthenticated visitor away from it before that shows.
