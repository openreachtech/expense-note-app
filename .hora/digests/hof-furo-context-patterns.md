# hof-furo-context-patterns
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-furo-context-patterns/ -->

**Read the source above whenever this leaves a question open.**

Furo apps keep Vue components thin: **all logic lives in `*Context.js` classes**, and a template
reads only `context.*`. This skill is the architectural foundation; it is short (221 lines) and is
**silent on several things checkpoint 16 needs** — those gaps are named below rather than invented.

## Base class

Every context extends `BaseAppContext` (`app/vue/contexts/BaseAppContext.js`), which extends
`BaseFuroContext` from `@openreachtech/furo-nuxt`. Put commonly reused methods on `BaseAppContext`.

`BaseAppContext<A, P, EE>` generics: `A` — ContextAccessor class or `null`; `P` — component Props
typedef or `{}`; `EE` — union of `emit()` event names or `null`.
(`ErrorPageContext extends BaseAppContext<null, ComponentProps, null>`.)

Inherited from `BaseFuroContext`: `this.props`, `this.componentContext`, `this.attrs`, `this.slots`,
`this.emit`, `this.expose`, `this.watch` (Vue's `watch`), `this.$` (ContextAccessor),
`this.EMIT_EVENT_NAME`, `this.Ctor`.

## Lifecycle contract — constructor + `create()` + `setupComponent()`

All three take **identical single-object destructured params**.

```js
export default class AnnouncementsPageContext extends BaseAppContext {
  constructor ({
    props,
    componentContext,
    route,
    router,
    statusReactive,
    errorMessageHashReactive,
    fetcher,
  }) {
    super({
      props,
      componentContext,
    })

    this.route = route
    this.router = router
    this.statusReactive = statusReactive
    this.errorMessageHashReactive = errorMessageHashReactive
    this.fetcher = fetcher
  }

  static create (args) {
    return new this(args)
  }

  setupComponent () {
    this.watch(
      [
        () => this.route.query.page,
      ],
      async () => {
        await this.fetcher.fetchAnnouncementsOnEvent({
          input: this.buildAnnouncementsInput(),
        })
      },
      {
        immediate: true,
      }
    )

    return this
  }

  get announcements () {
    return this.fetcher.announcementsCapsule
      .announcements
  }
}
```

- **`super({ props, componentContext })` is always called.** Base params are always exactly those
  two; everything else is an injected dependency assigned to `this.*`.
- **Instantiate only via `create()`** — a page calls `SomeContext.create(args).setupComponent()` and
  returns the instance. Never `new SomeContext(...)` from a page.
- **`setupComponent()` is the lifecycle hook** (overrides the base no-op). It registers
  `this.watch(...)` — often route-query driven with `{ immediate: true }` so a fetch runs on mount —
  and `onBeforeUnmount(...)`, then **returns `this`** for chaining.

> `javascript-style.md` bans chaining an instance method onto a `.create()`. The repo's existing
> `pages/sign-in/index.vue` already splits it: `const signInPageContext = SignInPageContext.create({…})`
> then `signInPageContext.setupComponent()`. Keep that split. (Conflict 1 below.)

## Dependency injection — created in `setup`, injected into `create()`

All reactive state, composables, clients, stores, routers, fetchers and form clerks are created in
the component/page `setup` and passed into `create({...})`.
**Never create refs/computed/reactive inside a context, and never call a composable inside a context.**

```js
setup (props, componentContext) {
  const route = useRoute()
  const router = useRouter()
  const statusReactive = reactive({
    isFetchingAnnouncements: true,
  })
  const errorMessageHashReactive = reactive({
    announcements: null,
  })

  const announcementsGraphqlClient = useGraphqlClient({
    Launcher: AnnouncementsQueryGraphqlLauncher,
  })
  const fetcher = AnnouncementsFetcher.create({
    statusReactive,
    errorMessageHashReactive,
    graphqlClientHash: {
      announcements: announcementsGraphqlClient,
    },
  })

  const args = {
    props,
    componentContext,
    route,
    router,
    statusReactive,
    errorMessageHashReactive,
    fetcher,
  }
  const context = AnnouncementsPageContext.create(args)

  context.setupComponent()

  return {
    context,
  }
}
```

An **optional** dependency gets a default in `create()`, typed via `RequiredExcept`, and the default
reads a **static factory on the class** — never an inline `SomeClass.create()`:

```js
static create ({
  props,
  componentContext,
  // ...
  localStorageClerk = this.createLocalStorageClerk(),
}) {
  return new this({
    props,
    componentContext,
    /* ... */
    localStorageClerk,
  })
}

static createLocalStorageClerk () {
  return StorageClerk.createAsLocal()
}
```

**Templates reach everything through the context**: getters for computed values
(`context.announcements`), methods for logic (`context.shouldHideAnnouncementsPagination()`),
reactive hashes exposed as fields (`context.statusReactive.isFetchingAnnouncements`).
A page may return several contexts and route each event to the right one:

```vue
@submit.prevent="signInSubmitterContext.submitForm({
  formElement: /** @type {HTMLFormElement} */ ($event.target),
})"
```

## Taxonomy, naming, typedefs

| Suffix | Role | Placement |
| --- | --- | --- |
| `*PageContext` | Page orchestration, getters, watchers | Next to `index.vue` in the page folder |
| `*SubmitterContext` | Form/mutation submission | Page folder; shared ones in `app/vue/contexts/submitters/` |
| `*LayoutContext` | Layout logic | `layouts/` |
| `*Context` | Component logic | Next to the component under `components/` |

Page-specific **fetchers / submitters / form-clerks are colocated with the page**; cross-page ones
go under `app/vue/contexts/` (e.g. `ErrorPageContext`, `submitters/SignOutSubmitterContext`).

Method naming:

| kind | form |
| --- | --- |
| event handler bound in a template | `...OnEvent` (`signOutOnEvent`, `fetchAnnouncementsOnEvent`) |
| fetch at mount | `...OnMounted` |
| predicate | `should...` / `has...` / `is...` |
| builder | `generate...` / `compose...` / `build...` |
| extractor | `extract...` |
| launcher hooks getter | `get <operation>LauncherHooks ()` |

JSDoc: at the bottom of the file define `<Name>ContextParams` (extends `BaseFuroContextParams`) and
`<Name>ContextFactoryParams` (an alias of Params, or `RequiredExcept<Params, 'optionalField'>`).
`@import` blocks pull types from `#app`, `vue`, `vue-router`, the sibling `./index.vue`, fetchers and
GraphQL payload/capsule modules. See `hoc-jsdoc`.

## Answers to the five checkpoint-16 questions

Verified against installed `@openreachtech/furo` (`lib/client/graphql/`), `@openreachtech/furo-nuxt`
2.x and the repo's `app/vue/contexts/BaseAppContext.js`.

### 1. How a context invokes a GraphQL operation — CORRECTED

The skill writes `useAppGraphqlClient(AnnouncementsQueryGraphqlLauncher)` (positional). **That
composable does not exist**; `composables/` holds only `useRedirect.js`. The real export, confirmed
in `furo-nuxt/index.js`, is `useGraphqlClient`, and it takes an **object**:

```js
// in setup() only — never inside a context
const signInGraphqlClient = useGraphqlClient({
  Launcher: SignInMutationGraphqlLauncher,
})
```

It returns `{ capsuleRef, invokeRequestOnEvent, invokeRequestOnMounted, invokeRequestWithFormValueHash }`.
Each invocation internally does `Launcher.create()` → `launchRequest({ payload, hooks })` and then
**assigns the resulting Capsule to `capsuleRef.value`**. `capsuleRef` starts as
`Launcher.createCapsuleAsPending()`. The context reads it through a getter:

```js
get signInCapsule () {
  return this.graphqlClientHash.signIn
    .capsuleRef
    .value
}
```

Which call to use: see the `hof-graphql` digest's table. `signIn` takes `{ input: … }`, so either
`invokeRequestWithFormValueHash({ valueHash })` (furo wraps the whole value hash as
`{ input: valueHash }`) or an explicit
`invokeRequestOnEvent({ variables: { input: { email, password } }, hooks })`.

### 2. The Fetcher / SubmitterContext split

What the skill prescribes: a page delegates **data loading to a `<Feature>Fetcher`** and **mutation
submission to a `<Feature>SubmitterContext`**, both created in `setup` and injected — the Fetcher via
`Fetcher.create({ statusReactive, errorMessageHashReactive, graphqlClientHash: { <operation>: client } })`,
the SubmitterContext returned alongside the PageContext so the template can call
`signInSubmitterContext.submitForm({ formElement })`. The GraphQL clients are grouped into a
**`graphqlClientHash`** keyed by operation name; the Fetcher/Submitter owns the capsule getters and
the `...OnEvent` methods, and the PageContext's getters read through it
(`this.fetcher.announcementsCapsule`).

**The skill does not say when a page context may do the work itself instead of delegating**, and
never shows a `*SubmitterContext` body. `hof-graphql`'s `operations-consuming.md` defers the full
flow to `[[fetcher-operation]]` and `[[mutation-operation]]` — **neither skill is equipped here**. A
one-operation page whose PageContext holds the client directly is covered by neither.
full text: `.claude/skills/hof-furo-context-patterns/references/dependency-injection.md`,
`.claude/skills/hof-graphql/references/operations-consuming.md`

### 3. Pending/loading driven by a real response — hook names CORRECTED

There is **no `onLoading` and no `onResolved`** anywhere in furo / furo-nuxt / furo-vue (grepped).
`launchRequest({ payload, hooks })` destructures exactly four:

| hook | signature | meaning |
| --- | --- | --- |
| `beforeRequest` | `async (payload) => boolean` | runs before the fetch; **returning `true` aborts**, and the launcher answers `Capsule.createAsAbortedByHooks({ payload })` |
| `afterRequest` | `async (capsule) => void` | runs after the Capsule is built, before it is assigned to `capsuleRef.value` |
| `onUploadProgress` | `({ request, progressEvent }) => void` | |
| `onDownloadProgress` | `({ request, progressEvent }) => void` | |

So a pending flag is raised in `beforeRequest` (returning `false` to proceed) and lowered in
`afterRequest` — both driven by the real request, never set by hand around the call. The hooks object
is exposed by the skill's naming convention as `get <operation>LauncherHooks ()` on the
Fetcher/Submitter and passed as `invokeRequestOnEvent({ hooks: this.signInLauncherHooks })`.
Note `beforeRequest` runs **after** `payload.isInvalidVariables()` short-circuits, so an
invalid-variables capsule never raises the flag.

**The skill shows no hooks body at all** — the `get …LauncherHooks()` name in `conventions.md` is its
only mention. The status-flag-in-hooks shape above is read off the furo source, not the skill.

### 4. How an error reaches the template

The skill's shape: a page creates `const errorMessageHashReactive = reactive({ <operation>: null })`
in `setup` and injects the **same object** into the PageContext and into the Fetcher/Submitter. The
Fetcher/Submitter writes the message into it (keyed by operation) when a response comes back; the
PageContext only reads it through a getter, and the template reads that getter.

**Project constraint governing the write**: `BaseAppGraphqlCapsule.extractResolvedErrorMessage()` is
this project's **single** resolution point from error code to user-facing message. A context — page,
fetcher or submitter — **must never map a code itself**. The write is therefore
`errorMessageHashReactive.signIn = capsule.extractResolvedErrorMessage()` (which answers `null` when
there is no error), and `SignInPageContext.refusalMessage` keeps returning
`this.errorMessageHashReactive.signIn ?? null`. The inherited `capsule.getErrorMessage()` is
misleadingly named — **it returns a CODE** — and must not reach the template.

**The skill never shows the populate step**; it shows only the reactive hash created in `setup` and
injected. The sentence above is the repo's `BaseAppGraphqlCapsule` doc-comment plus the existing
`SignInPageContext` getters, not the skill.

### 5. How a context is unit-tested

**The skill says nothing about testing.** What holds in this repo (see the `hoc-jest` digest and
`tests/__tests__/node/pages/sign-in/SignInPageContext.js`, which already covers this class):

- Context tests live under `tests/__tests__/node/**` mirroring the source path, in the **node** jest
  project — no browser, no `@vue/test-utils`. `props` / `componentContext` are plain literals
  (`{ attrs: { class: 'unit-page' } }`), and the reactive hashes are **plain objects**, not `reactive()`.
- Structure per the testing rule: `super class` → `.create()` (`toBeInstanceOf` plus
  `globalThis.constructorSpy.spyOn` delegation) → `constructor` → `to keep properties` → one describe
  per member.
- **Injecting a fake launcher/client**: the client is an injected dependency, so a test passes a
  duck-typed stand-in — `{ capsuleRef: { value: <capsule> }, invokeRequestOnEvent: jest.fn() }` —
  inside the case's `params` / `factoryParams`. Nothing is hoisted to describe scope.
- **Asserting a state transition**: Act on the handler (`await context.onSubmitForm()`), then assert
  the injected `statusReactive` / `errorMessageHashReactive` object, and/or
  `expect(invokeRequestOnEvent).toHaveBeenCalledWith(...)`. A hooks-driven flag is tested by taking
  the `…LauncherHooks` getter and calling `beforeRequest` / `afterRequest` directly with a fixture
  capsule.

## The two open questions — what the skill offers

**Q43 (where the access token is held).** The skill says nothing about session credentials, tokens,
cookies or memory. Its only adjacent content is the optional-dependency pattern, whose worked example
happens to be storage: `localStorageClerk = this.createLocalStorageClerk()` defaulting to
`StorageClerk.createAsLocal()`. Read as guidance it says only **inject the clerk through `create()`
with a static factory default** — that is the seam; it takes no position on localStorage vs memory.
Verified seams: `AccessTokenClerk.create({ storage, key })` (defaults `storage = this.createStorageClerk()`
→ `StorageClerk.createAsLocal()`, `key = 'access_token'`), and `StorageClerk.create({ storage })`
accepts any object exposing `getItem` / `setItem` / `removeItem`. **Not decided here.**

**Q40 (a context reading route state and navigating).** The skill's position is direct and twice
stated: `route` and `router` are obtained in `setup` (`useRoute()` / `useRouter()`) and **injected
into `create()`**, assigned to `this.route` / `this.router`; and
**"never call a composable inside a context"**. A context reads route state as `this.route.query.<key>`
— the worked `setupComponent()` watches `() => this.route.query.page` — and navigates through the
injected `this.router`. That means `useRedirect()` may only ever be called in `setup`, with its
`redirectTo` injected as a dependency. The skill offers nothing on whether the bare exported function
itself should become a class. **Not decided here.**

## Conflicts — named, not resolved

**`D:\ORT\rules\` wins. Each item is raised, not silently fixed.**

1. **`SomeContext.create(args).setupComponent()`.** `javascript-style.md` forbids chaining an
   instance method onto a `.create()` — name the instance, then call the method. The rule wins; the
   repo's `pages/sign-in/index.vue` already splits it. The skill's one-liner is the conflicting form.
2. **`static create (args) { return new this(args) }` with an opaque `args`.** `architecture.md`'s
   Factory Method plus the repo's own `SignInPageContext` destructure every field explicitly. The
   rule wins — keep the explicit destructured `create()` already in the file.
3. **`useAppGraphqlClient(<Launcher>)` — positional, and non-existent.** The installed export is
   `useGraphqlClient({ Launcher })`, an object argument, which `javascript-style.md` requires chopped
   down one property per line with a trailing comma. Whether to add an app wrapper named
   `useAppGraphqlClient` is a decision to raise, not to make in passing. (Same conflict as recorded
   in the `hof-graphql` digest.)
4. **`Ctor` — an abbreviation.** `naming.md` bans abbreviated names. `get Ctor ()` is inherited from
   `BaseFuroContext` and is used by the skill's own prose. The rule formally wins; the name belongs to
   the base class and cannot be changed here.
