# hof-graphql
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-graphql/ -->

**Read the source above whenever this leaves a question open.**

GraphQL in a Furo app splits into two halves: the **schema types** (`types/graphql-schema.d.ts`)
and the **operations** (`app/graphql/client/`). Every operation is a **Launcher / Payload / Capsule
trio**.

## The contract is authoritative

`.hora/contracts/1.0.0/staff-graphql.graphql` is the pinned contract. A document, a variable, a
field name or a result shape that disagrees with it is **raised as a question — never silently
reconciled**, and never "fixed" by editing the frontend to match whatever the stub happens to
return.

## File set per operation

Each operation gets its own folder named after the GraphQL field (camelCase), holding exactly three
files, one `export default` class each:

```
app/graphql/client/queries/<fieldName>/
├── <Field>QueryGraphqlLauncher.js
├── <Field>QueryGraphqlPayload.js
└── <Field>QueryGraphqlCapsule.js

app/graphql/client/mutations/<fieldName>/
├── <Field>MutationGraphqlLauncher.js
├── <Field>MutationGraphqlPayload.js
└── <Field>MutationGraphqlCapsule.js
```

Naming: `<FieldPascalCase>{Query|Mutation}Graphql{Launcher|Payload|Capsule}.js`. For a **mutation**
the folder/class stem is the field with its **trailing token singularized** (generator's
`singularizeTrailingToken`); a query keeps the field verbatim. The **document's field selection and
the Capsule's root getter always use the original field name**, singularization or not.

The four operations of `#sign-in` (both `mutations/` and `queries/` are currently empty):

| contract field | kind | folder | class stem |
| --- | --- | --- | --- |
| `signIn(input: SignInInput!)` | mutation | `mutations/signIn/` | `SignInMutation` |
| `signOut` — no argument | mutation | `mutations/signOut/` | `SignOutMutation` |
| `renewAccessToken` — no argument | mutation | `mutations/renewAccessToken/` | `RenewAccessTokenMutation` |
| `signedInStaffMember` — no argument | query | `queries/signedInStaffMember/` | `SignedInStaffMemberQuery` |

All three extend the app bases in `app/graphql/client/` (`BaseAppGraphqlLauncher`,
`BaseAppGraphqlPayload`, `BaseAppGraphqlCapsule`), which extend `@openreachtech/furo` bases.
Subscriptions have parallel bases (`BaseAppGraphqlSubscriber`, `BaseAppSubscriptionGraphqlPayload`,
`BaseAppSubscriptionGraphqlCapsule`) — not used by this feature.

## Payload — document + variables

`static get document ()` returns a template literal tagged `/* GraphQL */` (editor highlighting).
The class generic is the `<Name>RequestVariables` typedef declared at the bottom of the same file.

**With an input object** (`signIn`):

```js
import BaseAppGraphqlPayload from '~/app/graphql/client/BaseAppGraphqlPayload.js'

/**
 * SignIn mutation payload.
 *
 * @extends {BaseAppGraphqlPayload<SignInMutationRequestVariables>}
 */
export default class SignInMutationGraphqlPayload extends BaseAppGraphqlPayload {
  /** @override */
  static get document () {
    return /* GraphQL */ `
      mutation SignInMutation ($input: SignInInput!) {
        signIn (input: $input) {
          staffMemberId
          accessToken
        }
      }
    `
  }
}

/**
 * @typedef {{
 *   input: schema.graphql.SignInInput
 * }} SignInMutationRequestVariables
 */
```

### What changes when there are NO variables

Three of this feature's four operations take no argument. The trio does **not** change shape — only
these three things do:

| | with variables | with none |
| --- | --- | --- |
| operation signature in the document | `mutation SignInMutation ($input: SignInInput!) {` | `mutation SignOutMutation {` — **no parentheses at all** |
| field selection | `signIn (input: $input) {` | `signOut {` — **no argument list** |
| the typedef | `@typedef {{ input: … }} …RequestVariables` (chopped, one field per line) | `@typedef {{}} SignOutMutationRequestVariables` — **still declared, empty object** |

The `@extends {BaseAppGraphqlPayload<…RequestVariables>}` line stays; the empty typedef is what it
points at. Nothing else is omitted — no Payload class, no `document`, no file is skipped.

```js
import BaseAppGraphqlPayload from '~/app/graphql/client/BaseAppGraphqlPayload.js'

/**
 * SignOut mutation payload.
 *
 * @extends {BaseAppGraphqlPayload<SignOutMutationRequestVariables>}
 */
export default class SignOutMutationGraphqlPayload extends BaseAppGraphqlPayload {
  /** @override */
  static get document () {
    return /* GraphQL */ `
      mutation SignOutMutation {
        signOut {
          signedOut
        }
      }
    `
  }
}

/**
 * @typedef {{}} SignOutMutationRequestVariables
 */
```

Variables are never supplied by the Payload itself: furo's `BaseGraphqlPayload.create()` signs
`({ variables = {}, options = {} } = {})`, so a no-variable operation simply sends `{}`. The caller
supplies them (see *Invoking from a context*).

## Capsule — normalize the response

A root `get <originalFieldName>ValueHash ()` reads `this.content?.<field> ?? null`; one further
getter per result field drills in. A **list** field falls back to `[]`, a scalar/object to `null`.
The `<Name>ResponseContent` typedef at the bottom shapes `this.content`, referencing generated
types as `schema.graphql.*` (no import — they are global).

```js
import BaseAppGraphqlCapsule from '~/app/graphql/client/BaseAppGraphqlCapsule.js'

/**
 * SignIn mutation graphql capsule.
 *
 * @extends {BaseAppGraphqlCapsule<SignInMutationResponseContent>}
 */
export default class SignInMutationGraphqlCapsule extends BaseAppGraphqlCapsule {
  /**
   * get: signInValueHash
   *
   * @returns {schema.graphql.SignInResult | null}
   */
  get signInValueHash () {
    return this.content
      ?.signIn
      ?? null
  }

  /**
   * get: accessToken
   *
   * @returns {string | null}
   */
  get accessToken () {
    return this.signInValueHash
      ?.accessToken
      ?? null
  }
}

/**
 * @typedef {{
 *   signIn: schema.graphql.SignInResult
 * }} SignInMutationResponseContent
 */
```

### What the base already gives you — do not re-implement

| member | where | note |
| --- | --- | --- |
| `content`, `errors`, `hasContent()`, `hasError()`, `isPending()`, `hasNetworkError()`, `hasQueryError()`, `getErrorMessage()` | furo `BaseGraphqlCapsule` | `getErrorMessage()` returns an error **CODE**, not a message |
| `extractResolvedErrorMessage()` | this repo's `BaseAppGraphqlCapsule` | **the single point where a code becomes a user-facing message** |

**A per-operation Capsule must never map an error code to a message itself.** Codes belong in
`app/constants-error.js` and are resolved only by `extractResolvedErrorMessage()`; a second mapping
site is how the "unknown address and wrong password refuse identically" guarantee quietly stops
holding. See the `hof-error-handling` digest.

## Launcher — wire Payload + Capsule

**One Launcher per operation, never shared.** furo's `BaseGraphqlLauncher.Payload` / `.Capsule` are
abstract and `throw new Error('this function must be inherited')`, so a missing pair fails at
runtime. Only static getters; no other logic:

```js
import BaseAppGraphqlLauncher from '~/app/graphql/client/BaseAppGraphqlLauncher.js'

import SignInMutationGraphqlPayload from './SignInMutationGraphqlPayload.js'
import SignInMutationGraphqlCapsule from './SignInMutationGraphqlCapsule.js'

/**
 * SignIn mutation graphql launcher.
 *
 * @extends {BaseAppGraphqlLauncher}
 */
export default class SignInMutationGraphqlLauncher extends BaseAppGraphqlLauncher {
  /** @override */
  static get Payload () {
    return SignInMutationGraphqlPayload
  }

  /** @override */
  static get Capsule () {
    return SignInMutationGraphqlCapsule
  }
}
```

`BaseAppGraphqlLauncher.graphqlConfig` already returns the singleton `graphqlConfig`
(`ENDPOINT_URL` / `WEBSOCKET_URL`), populated at runtime by `plugins/000.furo.js` from env. A
per-operation Launcher never touches the endpoint.

## Auth headers — already handled, and out of scope here

`BaseAppGraphqlPayload.collectBasedHeadersOptions()` (this repo) appends
`{ [HEADER_KEY.ACCESS_TOKEN]: accessToken }` — `'x-renchan-access-token'` from `app/constants.js` —
when a token is present, reading it through `loadAccessToken()` →
`StorageClerk.createAsLocal().get(STORAGE_KEY.ACCESS_TOKEN)`. **A per-operation Payload must not
override `collectBasedHeadersOptions()` and must not set the header itself.**

Where the token is stored and when it is written is **open question Q43, checkpoint 16** — do not
resolve it here, and do not edit the base while building the four clients.

## Invoking from a context

Logic lives on a context extending `BaseAppContext`; a template reads only `context.*`. The client
is created in `setup` (never inside a context class) and injected.

```js
// in setup()
const signInGraphqlClient = useGraphqlClient({
  Launcher: SignInMutationGraphqlLauncher,
})
```

The composable exposes `capsuleRef` plus three invocations, each of which replaces
`capsuleRef.value` with the new Capsule:

| call | use for | variables sent |
| --- | --- | --- |
| `invokeRequestOnEvent(args?)` | a user-triggered operation | `args.variables ?? {}` |
| `invokeRequestOnMounted(args?)` | a fetch at mount | `args.variables ?? {}` |
| `invokeRequestWithFormValueHash({ valueHash, extraValueHash?, options?, hooks? })` | a `<form>` submission | furo's `generateVariables()` wraps the whole value hash as **`{ input: valueHash }`** |

- `signIn` — the `{ input: … }` wrapping is exactly its contract, so a form submits with
  `invokeRequestWithFormValueHash`, or explicitly with
  `invokeRequestOnEvent({ variables: { input: { email, password } }, hooks })` (chopped down).
- `signOut` / `renewAccessToken` / `signedInStaffMember` — **never use
  `invokeRequestWithFormValueHash`**: it would send an `input` variable the document never declares.
  Call `invokeRequestOnEvent({ hooks })` (or `invokeRequestOnMounted({ hooks })`), letting variables
  default to `{}`.

Hooks are `GraphqlType.LauncherHooks<Payload, Capsule>` with `beforeRequest` (returning `true`
aborts) / `afterRequest`. Reading the result in a context:

```js
get signInCapsule () {
  return this.graphqlClientHash.signIn
    .capsuleRef
    .value
}
```

Full Fetcher / SubmitterContext flow: `hof-nuxt` and `hof-furo-context-patterns` digests.
full text: `.claude/skills/hof-graphql/references/operations-consuming.md`

## `types/graphql-schema.d.ts`

**Operation result types are declared there, not as JSDoc typedefs.** One ambient file, real
TypeScript, `export {}` first (makes it a module so `declare global` applies), then:

```ts
export {}

declare global {
  namespace schema.graphql {
    type DateTime = string

    interface SignInInput { email: string; password: string }
    interface SignInResult { staffMemberId: number; accessToken: string }
  }
}
```

Every `.js` file references these as `schema.graphql.<TypeName>` **without any import**.

- Naming mirrors the backend SDL: entities as-is, inputs `<Name>Input`, results `<Name>Result`,
  scalars aliased (`BigNumber` / `DateTime` → `string`, `DateOnly` → `string // YYYY-MM-DD`,
  `Upload` → `File`; an unknown custom scalar falls back to `unknown` and must be reviewed).
  **Copy each field name verbatim from the SDL — never re-spell it on the frontend.** A
  classification field is `xxxCategory`, never `xxxType`.
- The root `Query` / `Mutation` / `Subscription` types are **dropped** — they are expressed as
  operation clients.
- The per-operation `*RequestVariables` / `*ResponseContent` typedefs **stay in their Payload /
  Capsule files, never in this `.d.ts`**, because the `.d.ts` is regenerated whole.

**This file does not exist yet in `expense-note-frontend-staff/types/` (only `furo-nuxt.d.ts` and
`jest.d.ts`).** Checkpoint 14 must create it, or the four Payloads/Capsules cannot reference
`schema.graphql.SignInInput` / `…Result`.

## Generating rather than hand-writing

Both generators read backend `.graphql` schema files; here they are already local at
`expense-note-backend/server/graphql/schemas/staff/`. Run from the frontend project root; the
scripts live at `.claude/skills/hof-graphql/references/` in this repository (the skill's own command
writes a `lib/skills/frontend/hof-graphql/references/` path).

| script | output | overwrite behavior |
| --- | --- | --- |
| `generate-graphql-types.js … --out types/graphql-schema.d.ts` | the whole ambient `.d.ts` | **no `--force` flag** — always rewrites the single file entirely |
| `generate-graphql-clients.js … --out app/graphql/client` | `queries/<field>/` + `mutations/<field>/` trios | **do not pass `--force`** — by default it **skips** an operation whose files already exist, preserving hand-tuned clients |

Narrow with `--target signIn signOut renewAccessToken signedInStaffMember`, or point the path at a
single `.graphql` file / subfolder. `--depth <n>` controls nested selection-set expansion (default
10). Generated output is a starting point, not an exemption: it must still be read against the
contract and the always-on rules before it is committed.
full text: `.claude/skills/hof-graphql/references/generate-graphql-clients.js`,
`.claude/skills/hof-graphql/references/generate-graphql-types.js`

## Conflicts — named, not resolved

**The always-on rules in `D:\ORT\rules\` win.** Each item below is a conflict to raise, not to fix
silently.

1. **Trio classes hold no property and define no `static create()`.** `architecture.md` requires
   every class to hold at least one property, forbids static-only classes, and requires a Factory
   Method. A Payload / Capsule / Launcher subclass as the skill (and the generator) writes it is
   static getters only. The rule wins on paper; the shape is imposed by the furo base classes,
   which supply the constructor and `static create()`. Unresolved.
2. **`useAppGraphqlClient(<Launcher>)` — positional, and absent.** The skill and the `hof-nuxt`
   digest both write `useAppGraphqlClient(SomeLauncher)`. No such composable exists in this
   repository (`composables/` holds only `useRedirect.js`); the installed
   `@openreachtech/furo-nuxt` exports `useGraphqlClient({ Launcher })`, taking an **object**. Under
   `javascript-style.md` an object-literal argument is chopped down, one property per line, trailing
   comma. Whether to add an app wrapper named `useAppGraphqlClient` is a decision to raise, not one
   to make in passing.
3. **`isUnauthenticated()` is claimed but does not exist.** `operations-base-classes.md` lists it on
   `BaseAppGraphqlCapsule`. This repository's `BaseAppGraphqlCapsule` defines only
   `extractResolvedErrorMessage()`, and furo 1.11.0's `BaseGraphqlCapsule` has no such member.
   Do not call it; do not add it while building the four clients.
4. **`valueHash` / `Ctor` naming.** The root Capsule getter is `<field>ValueHash` and furo exposes
   `get Ctor ()`. `naming.md` bans abbreviations. The convention is furo's and is followed for
   uniformity; the rule is the one that formally wins.
