# hoc-jsdoc
<!-- hora-skills-ort-core 0.2.0 -->
<!-- source: .claude/skills/hoc-jsdoc/ -->

**Read the source above whenever this leaves a question open.**

All typing is JSDoc — no TypeScript syntax, no `.ts` files. Always annotate types with JSDoc.

## Blocks that are required

- `function` declarations and methods (`MethodDefinition`) **must** have JSDoc. **Empty constructors and empty functions are not exempt** (`exemptEmptyConstructors: false` / `exemptEmptyFunctions: false`). Arrow functions, function expressions, and class declarations themselves are not targeted.
- No empty JSDoc blocks, no empty descriptions.
- **Single-line blocks are prohibited** — always write them across multiple lines. The only exceptions are `lends` / `type`, plus `extends` / `inheritdoc` / `override`.
- A function that throws needs `@throws`; a generator needs `@yields`.

## `@param`

- `@param` is required for every argument, **with a type and a name**; destructured (named) parameters are targeted too, and the documented name must match the actual argument name.
- **Do not use `{object}`.** Write objects as an inline type literal, explicitly listing each property as a named parameter.
- **Name a single object argument `params`** unless there is a special reason not to.
- **Do not place a semicolon or comma after each chopped-down property.**
- If you write a `@param` description, prefix it with a hyphen `- `. (The description itself is not required by lint.)

```javascript
// NG
/**
 * @param {object} params - Parameters.
 * @param {string} params.alpha - Alpha.
 */

// OK
/**
 * @param {{
 *   alpha: number
 *   beta: number
 * }} params - Parameters.
 */
```

## `@returns`

- **Every function and method gets `@returns`, even one that returns nothing** — `@returns {void}`, or `@returns {Promise<void>}` when async. Never omit the tag. (Stricter than lint.)
- **Attach a description as well as a type.** Sole exception: `@returns {void}` / `@returns {Promise<void>}`.
- **No hyphen between the type and the `@returns` description** — the leading `- ` is a `@param` convention only.

```javascript
// NG: no description
/**
 * @returns {string}
 */

// NG: a hyphen before the description
/**
 * @returns {string} - Default characters to pick from.
 */

// OK
/**
 * @returns {string} Default characters to pick from.
 */
```

## Types

| Rule | Write | Never |
|---|---|---|
| Arrays | `Array<string>` | `string[]` (any trailing `[]`) |
| Any type | `*` | `any` |
| Unknown-key object | `Record<string, *>` (last resort) | `object` / `Object` — **prohibited regardless of the reason** |
| "No value" | `null`, e.g. `@returns {string \| null}` | `undefined` in a type |
| Returns nothing | `@returns {void}` | `@returns {undefined}` |

- **Avoid vague types; write the most specific type you can.** `object` / `Object` / `any` / `*` are not used unless clearly necessary. Investigate the type and write it as a type literal with the most detailed properties possible, or as a concrete type name.
- **Make generics concrete** — `Array<UserEntity>`, not `Array<object>`.
- A type literal with explicit properties beats `Record<string, *>` wherever possible.
- `undefined` exception: when a third-party module or the like requires `undefined`, you may write it in `@returns` and so on. If there is a way to avoid `undefined`, prefer that.

## Do not write undefined type names

Any type name other than built-ins (`string` / `number` / `boolean` / `Array` / `Object` / `*` / `null`, etc.) and TS utility types (`Record` / `Partial` / `Pick` / `Omit` / `ReturnType`, etc.) must be **defined before it is referenced**, via one of `@typedef` / `@class` / `@interface` / import. So `Array<UserEntity>` requires a `UserEntity` `@typedef` (or the like) in the same file or its import source. **Correction to an earlier version of this digest:** it said `jsdoc/no-undefined-types` was "off in the base default but overridden to `error` by this project's `eslint.config.js`". **That is false for this repository.** `expense-note-backend/eslint.config.js` overrides only `no-shadow` and, for three named files, the `eslint-comments` pair — so the rule stays `off`. Verified with `npx eslint --print-config`, which resolves it to `0`; reading a plugin's source instead is what produced three wrong readings of `id-denylist` earlier in this project.

So **nothing mechanically enforces this convention here** — follow it because the convention says so, not because a check would catch a miss. It matters most for ambient namespaces that resolve through `jsconfig.json` rather than through an import: `GraphqlType.*`, `model.*` and — since checkpoint 3 of `#sign-in` — `server.graphql.staff.*`. Lint will not tell you a name there is wrong; `npx tsc -p jsconfig.json --noEmit` will.

## `@typedef`

- Write `@typedef` as a block comment of **at least three lines**; a single-line `@typedef` errors under lint (`jsdoc/multiline-blocks`).
- **One `@typedef` per block.**
- **Separate `@typedef` block comments from one another with a blank line.**
- Fields are separated by a newline only — no trailing delimiter (see `@param` above).

```javascript
// OK: at least three lines, one per block, separated by a blank line
/**
 * @typedef {*} BooleanLike
 */

/**
 * @typedef {{
 *   id: number
 *   name: string
 * }} UserEntity
 */
```

Placement: **`@typedef` blocks go at the END of the file** (after the class/exports), each in its own block comment. Naming: params typedef `<ClassName>Params`; factory params `<ClassName>FactoryParams`; `FactoryParams` simply aliases `Params` when no keys are optional, and uses `RequiredExcept<Params, 'key'>` when `create()` defaults some keys via DI.

> Scope note: the source states placement and the `Params`/`FactoryParams` naming only in `references/placement.md` and `references/class-typing.md`, both marked **Frontend (Vue / Nuxt) only**. The skill body states no separate backend placement rule. full text: references/placement.md, references/class-typing.md

## Type-only imports

- Use a type-only import to reference a type declared in another module. **Never use the TypeScript `import type` statement** — it is not available in a JavaScript-only project.
- Two styles: the `@import` block tag and the inline `import('…')` expression. **Follow the style the repository has established, and do not mix both for the same type.** **renchan backends use the inline `import('…')` expression**; Furo / Nuxt apps use `@import`. If a repository has established neither, prefer `@import`.
- One `@import` tag per JSDoc block, one source module per block; blocks at the end of the file, separated by a blank line.

Inline `import('…')` form — valid in any type position (`@type`, `@typedef`, `@param`, `@returns`, `@extends`); generics and unions nest as usual:

```javascript
/**
 * @returns {import('./CustomerOrdersBk.js').default} Backup model declaration.
 */
```

- For a module's **default export**, use `.default`.
- **Always include the filename extension** (`.js` / `.vue`) in the module path — never omit it.
- An imported type can be aliased to a local name in one `@typedef`, then used bare:

```javascript
/**
 * @typedef {import('@openreachtech/furo-nuxt/lib/contexts/BaseFuroContext.js').BaseFuroContextParams} ComponentContextParams
 */
```

- **Ambient globals are used unqualified, never wrapped in `import('…')` and never `@import`ed**: `RequiredExcept`, `OptionalExcept`, `NullableExcept`, and the `schema.graphql.*`, `furo.*`, `GraphqlType.*` namespaces (declared under `declare global` in `types/*.d.ts`).
- When the repository's convention is `@import`, promote a recurring inline `import('…')` to a named block rather than repeating the path. full text: references/import-tag.md, references/import-expression.md

## Class-level annotations & the casting idiom

- **`@override`** on overridden statics/getters — notably `static create`, `static get EMIT_EVENT_NAME`, `static get document`.
- **`@extends`** on the class doc: `@extends {BaseAppContext<null, ComponentProps, null>}`, or `@extends {BaseGraphqlCapsule<D>}` with `@template D`.
- **`@template` with constraints** in generics: `@template {(...args: Array<unknown>) => void} T`, `@template {GraphqlType.LauncherCtor} L`.
- **Class field types** via `@property` in the class-level doc, or captured through the constructor's params typedef.
- The `create()` factory template idiom, with its cast:

```javascript
/**
 * @template {X extends typeof OrderDetailProductContext ? X : never} T, X
 * @override
 * @param {OrderDetailProductContextFactoryParams} params
 * @returns {InstanceType<T>} Instance of this class.
 * @this {T}
 */
static create ({
  // ...
}) {
  return /** @type {InstanceType<T>} */ (
    new this({
      // ...
    })
  )
}
```

> Scope notes: the source writes **`@extends`**, never `@augments`, and states these annotations in `references/class-typing.md`, marked **Frontend (Vue / Nuxt) only**. The parenthesized `/** @type {…} */ ( … )` cast is the only casting idiom the skill shows (here for `InstanceType<T>`; also for `String` in a Vue `PropType`). The skill states **no** Sequelize-specific `/** @type {*} */` cast for reconciling a library's return type. full text: references/class-typing.md

## `@public` and access

- **Add `@public` to the JSDoc of any method that serves as an entry point accessed from outside.**
- Write `@public` / `@private` / `@protected` / `@access` correctly and without duplicates (`jsdoc/check-access`).

```javascript
/**
 * Generate a random text of the given length.
 *
 * @param {{
 *   length: number
 * }} params - Parameters.
 * @returns {string | null} Generated random text, or null if it cannot be generated.
 * @public
 */
```

## Block layout (lint-enforced)

- **Every line must start with `*`**; keep the `*` column aligned, one space after the tag / type / name / hyphen.
- **Exactly one blank line between the description and the first tag; no blank lines between tags; no blank line before the closing.**
- **Order tags by the default `tagSequence`** — roughly `@param` → `@returns` → `@throws` → … → `@public`/`@access` → `@example`.
- No stray `*` in the middle or at the end of a line (leading whitespace allowed).
- Type syntax must not be broken (matched `{`–`}`).

## Relaxed by lint (do not "fix" these)

`jsdoc/no-types` off (type annotations ARE allowed — this skill depends on it), `require-returns-description` off, `require-description` off, `require-param-description` off, `require-example` off, `require-file-overview` off, `check-tag-names` off (custom tags such as `@note` allowed), `match-description` / `require-description-complete-sentence` off, `valid-types` / `imports-as-dependencies` / `informative-docs` / `text-escaping` off. full text: references/eslint-jsdoc-rules.md
