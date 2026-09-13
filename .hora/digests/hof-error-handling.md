# hof-error-handling
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-error-handling/ -->

**Read the source above whenever this leaves a question open.**

## The model

Backend returns a dotted **error code** (`'203.M001.001'`). The frontend turns that code into a
user-facing message at **a single resolution point** on `BaseAppGraphqlCapsule`. Every code is its
own entry — **no array grouping** and **no reverse map**.

Typical file locations:

| File | Holds |
| --- | --- |
| `app/constants-error.js` (or `app/constants/errors.js`) | `ERROR_CODE_HASH` + `ERROR_LOCALE_HASH` — separate from `app/constants.js` |
| `app/graphql/client/BaseAppGraphqlCapsule.js` | the resolver method + `isUnauthenticated()` |
| `i18n/locales/<lang>.json` | the per-language text the locale paths point at |
| `error.vue` (repo root) + `app/vue/contexts/ErrorPageContext.js` | the Nuxt error page |

## Two variants — an app uses one or the other

The resolver's shape depends on how the app stores messages. **The skill states these as
alternatives ("An app uses one or the other"), not as a sequence.**

| Variant | Dictionary | Resolver | i18n needed |
| --- | --- | --- | --- |
| locale-path | `ERROR_LOCALE_HASH` (code → i18n key) | `extractResolvedErrorLocalePath()` | yes — locale files, `const { t } = useI18n()` in `setup`, `this.t` wired into the context |
| static-string | `ERROR_MESSAGE_HASH` + `ERROR_CODE_MAP` (single language) | `extractResolvedErrorMessage()` | **no** — "the static variant needs neither" |

**The locale-path machinery named in the skill's own description requires an i18n layer even for one
language**: the map value is a locale **path, never a message string**, and rendering happens via
`t()` at the surfacing point. The static-string variant reaches final text with no i18n at all.
Which one this feature installs is a fork — see Conflicts & forks below.

## `ERROR_CODE_HASH` / `ERROR_LOCALE_HASH` — one entry, end to end

```js
// app/constants-error.js
export const ERROR_CODE_HASH = /** @type {const} */ ({
  // <SemanticName><CodeSuffix>: '<code>'
  Unauthenticated102X000001: '102.X000.001',
  InvalidEmail203M001001: '203.M001.001',
})

export const ERROR_LOCALE_HASH = /** @type {const} */ ({
  // keyed by the CODE value → a locale PATH (i18n key), never a message string
  [ERROR_CODE_HASH.Unauthenticated102X000001]: 'errors.unauthenticated',
  [ERROR_CODE_HASH.InvalidEmail203M001001]: 'errors.invalidEmail',
})
```

```jsonc
// i18n/locales/en.json
{ "errors": { "invalidEmail": "The email address is invalid." } }
```

- Identifier = **PascalCase `<SemanticName><CodeSuffix>`**, suffix = the code with dots removed
  (`203.M041.001` → `203M041001`), so a code seen in logs maps to exactly one entry.
- Locale paths are **camelCase, dot-namespaced under `errors.`**.
- Keep entries **grouped by code range with consistent section comments**: Standard Client,
  Standard Server, Invalid Input, Not Found, Business Logic, Server Errors.
- Two codes sharing a semantic name may carry different text — point them at different paths.
  **If several codes should read identically, point them at the same path — do not collapse them
  into one `ERROR_CODE_HASH` entry.**
- **Hazard:** a duplicate code value silently overwrites a path, because `ERROR_LOCALE_HASH` is
  keyed by the code value. Guard it with the maintenance tests.

**Adding a new error** — three steps, in order:
1. Add the code to `ERROR_CODE_HASH` with a `<SemanticName><CodeSuffix>` identifier.
2. Add its locale path to `ERROR_LOCALE_HASH`, keyed by the code **via `ERROR_CODE_HASH`**.
3. Add the text for that path to **every** locale file.

## Resolution — the single point

Defined on `BaseAppGraphqlCapsule`. **Always resolve via the capsule method — never hand-map codes
in a context.** `getErrorMessage()` is the furo base method and returns the error **code**.

```js
extractResolvedErrorLocalePath () {
  const errorCode = this.getErrorMessage() // furo base returns the error *code*

  return ERROR_LOCALE_HASH[/** @type {keyof typeof ERROR_LOCALE_HASH} */ (errorCode)]
    ?? errorCode // unmapped → raw code; t() renders an unknown key as the key itself
}

isUnauthenticated () {
  return this.getErrorMessage() === ERROR_CODE_HASH.Unauthenticated102X000001
}
```

Static-string form, for contrast — resolves to **final text in two hops**, `code → semantic name`
(`ERROR_CODE_MAP`) → `message` (`ERROR_MESSAGE_HASH`); a **null code falls back to
`ERROR_MESSAGE_HASH.Unknown`**; an unmapped code, or a name with no message, **falls back to the
raw code**. Its `isUnauthenticated()` compares against `ERROR_CODE_HASH.Unauthenticated`.
full text: references/resolving-and-surfacing.md#static-messages

**Unknown / unmapped code — what the user sees.** The skill *does* specify a fallback: the resolved
value is **the raw dotted code**, displayed as-is (`t()` renders an unknown key as the key itself).
The skill specifies **no `Unknown` fallback for a null code in the locale-path variant** — only the
static variant names one. Do not invent one; read the source if this case must be handled.

## Surfacing — `errorMessageHashReactive`

A `reactive()` hash **created in the `.vue` `setup`**, keyed by **camelCase operation names that
match the mutation names**, typed via a **local typedef at the end of `<script>`**, and **passed
into the relevant context(s)**. It holds the **rendered** message (final text), not a path.

```js
/** @type {Reactive<ErrorMessageHash>} */
const errorMessageHashReactive = reactive({
  updateEmail: null,
  verifyEmail: null,
})

/**
 * @typedef {{
 *   updateEmail: string | null
 *   verifyEmail: string | null
 * }} ErrorMessageHash
 */
```

**One reactive object is shared across the contexts that need it** (submitters + section context).
In the submitter's launcher `afterRequest` hook: resolve and set on error; **clear it before
submitting**.

```js
// submitForm(): clear first
this.errorMessageHashReactive.updateEmail = null

// afterRequest hook — locale paths: render the resolved path with t():
afterRequest: async capsule => {
  if (capsule.hasError()) {
    this.errorMessageHashReactive.updateEmail = this.t(capsule.extractResolvedErrorLocalePath())

    return
  }

  this.successMessageHashReactive.updateEmail = this.t('messages.emailUpdated')
}
```

**The template holds no logic** — it binds the hash off the context as a whole and hands it down as
a prop; resolution and rendering both happen in the context's `afterRequest` hook:

```
:error-message-hash="context.errorMessageHashReactive"
```

Parallel `successMessageHashReactive` (success text) and `statusReactive` (loading flags like
`isInvokingUpdateEmail`) follow the same shape.

## The Nuxt error page — `error.vue`

`error.vue` (repo root) receives the `NuxtError` prop and **delegates to `ErrorPageContext`**
(`app/vue/contexts/ErrorPageContext.js`). It renders `context.errorStatusCode` +
`context.errorMessage` with a "return home" link, its label a locale path rendered with
`t('actions.returnHome')`.

**In-page vs error page.** Operation errors — anything a launcher's `afterRequest` sees — are
surfaced **in place** through `errorMessageHashReactive`; that path never navigates. `error.vue` is
Nuxt's own page for a `NuxtError`. **The skill does not state the boundary condition that routes an
error to `error.vue` rather than in-page**, so a refusal must not be assumed to reach it.
full text: references/error-page.md (13 lines — the whole reference)

## Maintenance tests

Guard three things — **duplicate code values, path coverage, translation coverage**. The duplicate
guard compares `Object.values(ERROR_CODE_HASH)` against itself by index; the other two filter codes absent from
`ERROR_LOCALE_HASH`, and every `ERROR_LOCALE_HASH` path against each locale file.
full text: references/error-handling-structure.md#4-maintenance-tests

## Product constraint for this feature (not from the skill)

**#sign-in §10 — an unknown email address and a correct address with the wrong password must be
refused identically.** The backend already enforces this: both outcomes carry the single code
`204.M001.001` (`InvalidCredentials`). **The frontend hash must not reintroduce the distinction —
one code, one entry, one message.** Never add a second entry, a second path, or any branch that
could let the two cases read differently.

The 13 codes this feature must map: `203.M001.001`–`004`, `204.M001.001`–`003`, `204.M002.001`–`002`,
`204.M003.001`–`002`, `204.Q001.001`–`002`.

## Conflicts & forks — named, not resolved

**`D:\ORT\rules\` wins over this skill in every row below. Flagged, not resolved.**

| # | Conflict | Rule that wins |
| --- | --- | --- |
| 1 | Skill uses bare `t()` / `this.t(...)` / `const { t } = useI18n()`. | `charter.md` (4) names this exact case: "`i18n.t()`, not `t()`". |
| 2 | `error.vue` snippet chains `.setupComponent()` onto `ErrorPageContext.create({ … })`. | `javascript-style.md` — "Assign a factory instance to a variable before calling its methods". |
| 3 | Maintenance-test snippets use `for (const … of Object.entries(LOCALES))`, a second `expect()` argument, bare `test()` with no class/member `describe`, no `test.each`, no AAA blank lines. | `javascript-style.md` (loop ban) and `testing.md` (describe structure, `test.each`, AAA, allowed matchers). |
| 4 | Skill snippets are not house-formatted — object arguments not chopped down, no blank line before an `if` following an assignment. | `javascript-style.md` chop-down rules, `charter.md` (6). |
| 5 | `getErrorMessage()` carries the banned `get~` prefix. | `naming.md` bans `get~`. **It is a furo base-class method, not ours** — it cannot be renamed; do not "fix" the call site. |

**The i18n fork — do not guess.** The skill's headline machinery (`ERROR_LOCALE_HASH` +
`extractResolvedErrorLocalePath` + `t()`) **requires an i18n layer even for a single language**. The
project has decided **English, single language, "no localization layer"**
(`expense-note-frontend-staff/ai/contexts/uiux-context.md` §7, grounded on spec §6), and i18n is not
currently installed in `expense-note-frontend-staff`. The skill supplies a static-string variant that
needs no i18n. Choosing between installing i18n and taking the static variant is a decision for the
build, not for the implementer to assume.
