# hof-nuxt
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-nuxt/ -->

**Read the source above whenever this leaves a question open.**

## Invariants that hold everywhere

- Every UI unit — page, component, layout — is a thin `defineComponent` that wires reactive state and clients in `setup`, delegates all logic to a paired **Context class**, and reads only through `context.*` in the template.
- **Auto-import is disabled** (`nuxt.config.js` sets `components: { dirs: [] }` and `imports: { autoImport: false }`). Import everything by hand.
- Never `<script setup>`, never the Options API. App code is JavaScript + JSDoc.
- Never create `ref` / `computed` / `reactive` inside a Context class, and never call a composable inside a Context class — do both in `setup` and inject the result.

Import sources:

| What | From |
| --- | --- |
| Vue APIs (`defineComponent`, `ref`, `reactive`, `computed`) | `'vue'` |
| Nuxt built-in components (`Icon`, `NuxtLink`) | `'#components'` |
| `definePageMeta`, `useState`, `defineNuxtRouteMiddleware`, `defineNuxtPlugin` | `'#imports'` (the last two also from `'nuxt/app'`) |
| `navigateTo` | `'nuxt/app'` |
| Other components | absolute alias with extension — `import AppDialog from '~/components/units/AppDialog.vue'` |
| Sibling context | relative — `import XxxContext from './XxxContext.js'` |

## Pages and routes

- **Every routable leaf is `index.vue`** inside a feature-named directory — `pages/documents/index.vue`, `pages/wallet/index.vue`. There is no bare `pages/foo.vue` for leaf routes.
- **Dynamic segments** are bracketed folders named for the route param: `pages/products/[id]/`, `pages/settings/address/[addressId]/`, `pages/order-history/[idHash]/` (`[id]` = numeric, `[idHash]` = hashed). Read the param with `useRoute()`.
- **`(parents)` group** — `pages/(parents)/*.vue` are parenthesized wrappers (ignored in the URL) acting as the parent route for a same-named child folder: `pages/(parents)/settings.vue` parents `pages/settings/*`. Each is a minimal `<NuxtPage />` shell, optionally setting shared `definePageMeta`.
- A route is reachable as soon as its `index.vue` exists at the right path; no route table is hand-maintained.

Sibling `.js` files live **in the same folder** as `index.vue` and are stripped from the route table by the `pages:extend` hook in `nuxt.config.js` (`kickOutJsFilesFromPages` filters `page.file.endsWith('.js')`):

| File | Role |
| --- | --- |
| `<Feature>PageContext.js` | Page orchestration — lifecycle, watchers, template getters |
| `<Feature>Fetcher.js` | Data loading |
| `<Feature>SubmitterContext.js` | Mutation submission |
| `<Feature>FormElementClerk.js` | Form validation rules |
| `<Feature>ItemContext.js` | Per-row/item sub-context |

## The page/context pairing

`setup(props, componentContext)` is the **only** place that touches Vue primitives, composables, and client creation. Build an `args` object, create the context(s), return them.

```vue
<script>
import {
  defineComponent,
} from 'vue'

import DocumentsPageContext from './DocumentsPageContext'

export default defineComponent({
  setup (
    props,
    componentContext
  ) {
    const args = {
      props,
      componentContext,
    }

    const context = DocumentsPageContext.create(args)
      .setupComponent()

    return {
      context,
    }
  },
})
</script>

<template>
  <div class="unit-page">
    Documents
  </div>
</template>
```

The paired `<Feature>PageContext.js` extends `BaseAppContext` (in this repository: `app/vue/contexts/BaseAppContext.js`), defines `constructor` + `static create()` + `setupComponent()` (which kicks off mounted fetches and **returns `this`**). Computed values are getters; logic is methods.

```js
export default class DashboardPageContext extends BaseAppContext {
  constructor ({
    props,
    componentContext,
    customerStore,
    fetcher,
    errorMessageHashReactive,
    statusReactive,
  }) {
    super({
      props,
      componentContext,
    })

    this.customerStore = customerStore
    this.fetcher = fetcher
    this.errorMessageHashReactive = errorMessageHashReactive
    this.statusReactive = statusReactive
  }

  static create (args) {
    return new this(args)
  }

  setupComponent () {
    this.fetcher.fetchAnnouncementsOnMounted()
    this.fetcher.fetchWalletBalancesOnMounted()

    return this
  }

  get isFetchingWalletBalances () {
    return this.statusReactive.isFetchingWalletBalances
  }

  get announcements () {
    return this.fetcher.announcementsCapsule
      ?.announcements
      ?? []
  }
}

/**
 * @import { ErrorMessageHash, UserInterfaceState } from './index.vue'
 * @import DashboardFetcher from './DashboardFetcher.js'
 */
```

Templates read **only** through `context.*` — getters (`context.announcements`), predicate methods (`context.hasMembership()`), formatters (`context.formatBalance()`), and the exposed reactive hashes (`context.statusReactive`, `context.errorMessageHashReactive`).

### Shared reactive vocabulary (page level)

Pages create two reactive hashes in `setup` and pass them into the PageContext / Fetcher / SubmitterContext:

- `statusReactive` — typed `Reactive<UserInterfaceState>`; loading flags `isFetching<Entity>` / `isInvoking<Operation>`.
- `errorMessageHashReactive` — typed `Reactive<ErrorMessageHash>`; per-operation error strings.

`UserInterfaceState` and `ErrorMessageHash` typedefs are **exported from `index.vue`** and imported by the sibling `.js` files via JSDoc `@import`.

A data-fetching page's `setup` body, in order: `definePageMeta({ ... })` → stores → the two reactive hashes → GraphQL clients via `useAppGraphqlClient(<Launcher>)` → the `Fetcher` (`create({ graphqlClientHash, errorMessageHashReactive, statusReactive })`) → `args` → context. A page with a mutation returns **both** contexts, and the template submits through the submitter:

```js
return {
  context,
  submitterContext,
}
```

full text: `references/pages-component.md`

## definePageMeta & page styling

Imported from `#imports`. Used for:

- `layout: 'gateway' | 'settings'` — non-default layout (default is implicit).
- `alias: '/'` — e.g. the dashboard aliasing the root.
- `$furo: { pageTitle }` and a custom `headerTitle`.

**Middleware is never defined per page** in a Furo app — no `pages/**` sets `middleware:`.

The page root is always `<div class="unit-page">`. Reusable sub-blocks get their own `unit-` name; selectors use child combinators and design tokens.

## Layouts

```
layouts/default.vue      + layouts/DefaultLayoutContext.js
layouts/settings.vue     + layouts/SettingsLayoutContext.js
layouts/gateway.vue      (no context — purely presentational)
```

| Rule | Value |
| --- | --- |
| Layout file name | lowercase Nuxt filename (`default.vue`); the `layout:` string is the filename stem |
| Context file | directly in `layouts/`, named `<PascalLayoutName>LayoutContext.js` |
| Root element | `<div class="unit-layout">` (modifiers as extra classes, `class="unit-layout settings"`) |
| Context params typedef | extends `BaseFuroContextParams` with injected deps (`customerStore: CustomerStore`) |
| `statusReactive` | page-level convention only; layouts don't need it |

Selection: `layout: 'gateway'` → auth pages (`login`, `sign-up`, `create-account`, `forgot-password`, `reset-password`); `layout: 'settings'` → all `pages/settings/**` leaves; no `layout:` → `default`.

`default.vue` renders app chrome (`AppSidebar`, `AppSidebarOverlay`, `AppHeader`, `AppToastContainer`) with page content in `<main class="main"><slot /></main>`.

**A layout gets a `*LayoutContext` only when it has behavior.** A behavior-free layout is a plain object — not even `defineComponent`:

```vue
<script>
import AppToastContainer from '~/components/toast/AppToastContainer.vue'

export default {
  name: 'GatewayLayout',

  components: {
    AppToastContainer,
  },
}
</script>
```

Nesting: `settings.vue` wraps its content in `<NuxtLayout name="default">` so it inherits sidebar + header chrome, then adds its own sub-nav around `<slot />`. **Prefer `<slot/>` over `<NuxtPage/>`** — it allows nested layouts.

## Middleware

All middleware is **global**, ordered by numeric prefix. Named (per-page) middleware is not used.

```
middleware/000.customer.global.js    # bootstrap customer store from token
middleware/001.gateway.global.js     # auth gateway / redirects
middleware/010.pageTitle.global.js   # SEO page title
```

Naming: `NNN.<name>.global.js` — 3-digit ordering prefix, camelCase name, `.global`.

**A middleware always returns either `navigateTo(...)` or `goNextAsIs()`** — never bare `undefined`. Each file defines the helper at the bottom:

```js
export default defineNuxtRouteMiddleware(async (to, from) => {
  // ...guards...
  return goNextAsIs()
})

/**
 * Go to the next as is.
 *
 * @returns {Promise<void>}
 */
function goNextAsIs () {
  return Promise.resolve()
}
```

Conventions: constants (paths, route lists — `PUBLIC_ROUTES`, `POST_LOGIN_PROHIBITED_ROUTES`, `SIGN_IN_PATH = '/login'`, `SETTINGS_PATH = '/settings'`) at module top. Auth reads via `AccessTokenClerk` / `StorageClerk` plus `useCustomerStore()` — **never read tokens ad hoc**. Paths normalized with `withoutTrailingSlash` from `ufo`. `FuroMeta.create({ routeTo: to }).skipFilter` bypasses the auth guard. The login redirect preserves the target — `navigateTo` with a template literal of `SIGN_IN_PATH` plus `?redirect=` and `to.fullPath`. Server-only work is guarded with `if (import.meta.server) { return goNextAsIs() }`.

full text: `references/middleware-patterns.md` (the three concrete patterns: auth gateway, customer bootstrap, page-title SEO)

## Plugins

```
plugins/000.furo.js                  # AppShare + GraphQL/REST config bootstrap (universal)
plugins/002.channelTalk.client.js    # client only
plugins/005.markdownit.client.js     # client only
```

Naming: `plugins/NNN.<name>[.client|.server].js` — 3-digit ordering prefix, camelCase name. `000.furo` runs first. Auto-registered from the **top level** of `plugins/` only.

Two forms — bare, and object form with a `name`:

```js
export default defineNuxtPlugin(async () => {
  setupGraphqlConfig()
  setupRestfulApiConfig()

  const $furo = await createShare({
    config: graphqlConfig,
  })

  return {
    provide: {
      furo: $furo,
    },
  }
})
```

```js
export default defineNuxtPlugin({
  name: 'markdown-it',

  async setup (nuxtApp) {
    // ...

    return {
      provide: {
        md,
      },
    }
  },
})
```

Conventions: provide keys are camelCase → `$<key>` on `nuxtApp`. **Runtime values always from `useRuntimeConfig().public.*`, never hard-coded.** Plugins may call stores and `watch(...)`.

## Global state — `useState` stores (no Pinia)

A store is a default-exported function named `use<Domain>Store` that wraps `useState('<key>', () => defaultState)` and returns a plain object exposing a single `<domain>StateRef` plus getter/action closures. Actions are **inner named `function` declarations (hoisted), placed below the `return`**.

```js
import {
  useState,
} from '#imports'

/**
 * Use `toast` store.
 *
 * @returns {ToastStore}
 */
export default function useToastStore () {
  /** @type {ToastState} */
  const defaultToastState = {
    toasts: [],
  }

  const toastStateRef = useState('toast', () => defaultToastState)

  return {
    toastStateRef,
    add,
    dismiss,
  }

  function add (toast) {
    // ...
  }

  function dismiss ({
    id,
  }) {
    // ...
  }
}

/**
 * @typedef {{
 *   toastStateRef: import('vue').Ref<ToastState>
 *   add: (toast: Toast) => void
 *   dismiss: (params: { id: string }) => void
 * }} ToastStore
 */
```

Rules:

- `useState` key is a unique global lowercase string (`'customer'`, `'toast'`).
- State ref is named `<domain>StateRef`.
- **Mutate directly**: `stateRef.value.x = ...`; partial updates spread (`{ ...stateRef.value.x, ...patch }`); clears reset to `null` or to the default state object.
- Actions take a **single destructured params object**.
- Async actions may call other composables (`fetchCustomer` uses `useAppGraphqlClient(...)`, checks `capsule.hasError()`, then calls its own setters).
- Typedefs at the bottom: `<Domain>Store` (return shape), `<Domain>State`, plus one per params object. GraphQL-derived types come from the global `schema.graphql.*` namespace.
- `customer.js` groups members with `// State`, `// Getters`, `// Actions` comment sections.

**Consumption:** call `use<Domain>Store()` in `setup` / middleware / plugins, then pass the store into a context. Contexts read `this.customerStore.customerStateRef.value.*` and call its actions. **Never call `useCustomerStore()` inside a context class.**

## App share — `AppShare` on `nuxtApp` (`$furo`)

`app/shares/AppShare.js` is a class **extending `FuroShare`** from `@openreachtech/furo-nuxt`, holding app-wide services/state and exposing getters/mutators over them.

```js
import {
  FuroShare,
} from '@openreachtech/furo-nuxt'

export default class AppShare extends FuroShare {
  constructor ({
    graphqlShare,
    sharedInterfaceStateReactive,
  }) {
    super({
      graphqlShare,
    })

    this.sharedInterfaceStateReactive = sharedInterfaceStateReactive
  }

  static create ({
    graphqlShare,
    sharedInterfaceStateReactive,
  }) {
    return new this({
      graphqlShare,
      sharedInterfaceStateReactive,
    })
  }

  get showsSidebar () {
    return this.sharedInterfaceStateReactive.showsSidebar
  }

  openSidebar () {
    this.sharedInterfaceStateReactive.showsSidebar = true
  }
}

/**
 * @typedef {FuroShareParams & {
 *   sharedInterfaceStateReactive: Reactive<SharedInterfaceState>
 * }} AppShareParams
 */
```

- Lives in `app/shares/`, PascalCase filename, extends a Furo `*Share` base, has a `static create()`.
- Built in `plugins/000.furo.js` and provided under the `furo` key → `nuxtApp.$furo`. That plugin also mutates the singleton config objects (`graphqlConfig`, `renchanRestfulApiConfig`) from `useRuntimeConfig().public.*`, builds a `FuroGraphqlShare` via `useSubscriptionConnector`, and creates `reactive({ showsSidebar: false })` as `sharedInterfaceStateReactive`.
- **Any reactive UI state it holds is created in the plugin and injected — never created inside the class.**
- Read it with `const { $furo } = useNuxtApp()` in `setup`, pass it into the Context as `furo`, and delegate (`this.furo.toggleSidebarVisibility()`). **Templates never call `$furo` directly.**
- Choose `AppShare` for app-wide services/UI state reached via `nuxtApp`; choose a `useState` store when the state is a domain data model (`customer`, `toast`).

## Runtime & ambient types

```
types/furo-nuxt.d.ts       # RuntimeConfig / PublicRuntimeConfig augmentation + furo.* namespace
types/global.d.ts          # RequiredExcept, OptionalExcept, NullableExcept
types/graphql-schema.d.ts  # namespace schema.graphql (generated)
types/router.d.ts          # vue-router RouteMeta augmentation
types/robot-payment.d.ts   # third-party ambient consts
```

**Environment variables pointing at the backend** are typed here. `useRuntimeConfig().public.<VAR>` is typed by augmenting `nuxt/schema` in `furo-nuxt.d.ts`:

```ts
import '@openreachtech/furo-nuxt/types/furo-nuxt'

declare module 'nuxt/schema' {
  interface RuntimeConfig {
    ENDPOINT_URL: string
  }

  interface PublicRuntimeConfig {
    ENDPOINT_URL: string
  }
}
```

**When you add a `.furo-env` variable, add its field to both `RuntimeConfig` and `PublicRuntimeConfig`** so consumers are typed. (The `.furo-env` mechanism itself is the separate `hof-furo-env` skill — not covered by this skill.)

Other conventions: shared/ambient types → `types/`, kebab-case `.d.ts` named by domain. Global helpers via `declare global`; framework config via `declare module 'nuxt/schema'`; router meta via `declare module 'vue-router'` (`interface RouteMeta { headerTitle?: string }`, which types the custom `headerTitle` set via `definePageMeta`); generated GraphQL under `namespace schema.graphql`, referenced unqualified as `schema.graphql.<Type>`. Add `export {}` at the top so augmentations apply as a module (unless the file leads with an `import`, as `furo-nuxt.d.ts` does). **Per-file / local types stay as inline JSDoc `@typedef` in the `.js` / `.vue` file** — only cross-cutting / ambient types belong in `types/*.d.ts`.

`global.d.ts` exposes `RequiredExcept<T, K>`, `OptionalExcept<T, K>`, `NullableExcept<T, K>` for unqualified use in JSDoc; `RequiredExcept` is the standard way to type a context/module `FactoryParams` that has DI-defaulted keys.

## Components (for checkpoints 12/15)

Each component is `Xxx.vue` + `XxxContext.js`, both PascalCase, in the same directory. Own folder by default; flat siblings for simple `units/` primitives and all of `composites/`. No separate `.css` file — always `<style scoped>` inside the `.vue`.

| Folder | Role |
| --- | --- |
| `components/units/` | Design-system primitives, domain-agnostic, all prefixed `App` (`AppButton`, `AppInput`, `AppDialog`, …) |
| `components/atoms/` | Small app-specific presentational pieces not in the generic App kit |
| `components/molecules/` | Composed, domain-aware widgets built from units/atoms |
| `components/organisms/` | Large composed sections and dialogs; typically declare `emits` |
| `components/composites/` | Standalone flow dialogs; files are flat here |
| `components/pages/` | Feature/route-specific sub-components; the folder tree **mirrors the route tree** |
| `components/layouts/` | App chrome — `AppHeader`, `AppSidebar`, `AppSidebarOverlay` |
| `components/toast/` | `AppToast`, `AppToastContainer` |

There is **no `templates/` tier** — do not create one. `components/pages/wallet/WalletHistory.vue` is a feature sub-component; `/pages/wallet/index.vue` is the actual Nuxt route that imports it.

**Every referenced component must appear in the `components: {}` block of `defineComponent`.** Props use object form with `type` / `required` / `default` / often a `validator`, typed by a JSDoc `PropType` cast on the line above `type`:

```js
orderSummary: {
  /** @type {import('vue').PropType<schema.graphql.OrderSummary>} */
  type: Object,
  required: true,
},
```

**Emits reference a static map on the context — never raw strings:**

```js
// Component
emits: [
  CreditCardSelectDialogContext.EMIT_EVENT_NAME.DISMISS,
  CreditCardSelectDialogContext.EMIT_EVENT_NAME.SELECT_CREDIT_CARD,
],
```

```js
// Context
static get EMIT_EVENT_NAME () {
  return {
    DISMISS: 'dismiss',
    SELECT_CREDIT_CARD: 'selectCreditCard',
  }
}
```

The context emits via `this.emit(this.EMIT_EVENT_NAME.SELECT_CREDIT_CARD, payload)`. Root element class is `unit-<name>`; lifecycle hooks are registered inside `setupComponent()`; cross-component overrides use `:deep(...)`.

full text: `references/components-tier-taxonomy.md`, `references/components-props-emits.md`, `references/components-reactivity-styling.md`

## Composables (read the conflict note below first)

The skill places reusable stateful logic in the **repo-root `composables/`** directory (not `app/composables/`), all named `use*`, taking a single destructured params object and returning an object of functions/refs. Two export styles coexist: `export default function useX ()` and `export const useX = ({ ... }) => ...`.

- **`useApp*` is reserved for thin wrappers over `@openreachtech/furo-nuxt` base composables** — spread the base result and add/override behavior. `useAppGraphqlClient` wraps `useGraphqlClient(Launcher)` and adds `invokeRequestOnEvent`, `invokeRequestOnMounted`, `invokeRequestWithFormValueHash`, plus a cross-cutting `afterRequest` hook that detects `capsule.isUnauthenticated()`, clears the access token via `StorageClerk.createAsLocal()`, and redirects to `/login?redirect=...`.
- **Setup-only rule (critical):** composables call setup-only APIs (`useRoute()`, `useRouter()`, `onMounted()`, `onUnmounted()`, injection), so they **must be called in `setup`** and their results passed into contexts. **Never call a composable inside a Context class.**

full text: `references/composables-conventions.md`, `references/composables-useapp-wrappers.md`, `references/composables-debounce.md`

## Conflicts with the project rules in `D:\ORT\rules\` — the rules win

Do not resolve these yourself; follow `D:\ORT\rules\` and raise the conflict.

| Skill says | Project rule says | Where in the skill |
| --- | --- | --- |
| Reusable stateful logic goes in `composables/` as `use*` **functions**; global state is a `use*Store` **function** with inner named `function` declarations | This repository's tree doc: shared logic is **a class under the app's own folders — never a composable and never a bare function** | the whole Composables section, plus `state-store-pattern.md` |
| `const args = { ... }`; `$md` for markdown-it; `NNN` prefixes | `naming.md` bans abbreviated names (`md` is one); `args` is not on the whitelist | `pages-component.md`, `plugins-*.md` |
| `<Icon :name="link.iconName" size="1.5rem" />` — two attributes on one line | `05-frontend.md` 6-9: 2+ attributes must be chopped down, one per line | `layouts-nesting-and-slots.md` |
| `setup (props, componentContext)` written inline in several examples | `javascript-style.md`: parameter lists and object arguments are always chopped down, one binding per line with trailing comma | `pages-component.md`, `components-reactivity-styling.md` |
| `static create (args) { return new this(args) }` with no JSDoc | `jsdoc.md`: a factory method carries `@template` / `@this` / `@returns {InstanceType<T>}`, and every method has a short description | `pages-context.md` |
| `v-for="(link, index) of context.settingNavigationLinks"` in a template | `javascript-style.md` bans imperative loops in **JS**; a template `v-for` is not JS control flow — confirm against the rule before treating it as a violation | `layouts-nesting-and-slots.md` |

Already aligned, not conflicts: the skill's `ref(null)` initial value matches `javascript-style.md`; no `else if` / `switch` appears anywhere in the skill; typedefs at the bottom of the file match `jsdoc.md`; the `static create()` factory itself matches `architecture.md`.
