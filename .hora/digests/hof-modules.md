# hof-modules
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-modules/ -->

**Read the source above whenever this leaves a question open.**

## When a module class is warranted

> "Use when reusing general logic across multiple files."

Reusable **general** logic. Furo follows an OOP structure, so utility classes are preferred
over utility functions or composables. The skill names no other trigger and no count
threshold beyond "across multiple files" — logic living in exactly one page or one
component is not a module.

## Placement

```
app/modules/TimerClerk.js         # class: manages a setTimeout lifecycle
```

- **All modules live flat in `app/modules/`.** There are no per-concept subfolders — the
  concept does not choose a directory, it chooses the class name.
- Modules are always classes, **one class per file**, and each class must have **at least
  one property**.

## Naming

| Suffix | Role |
| --- | --- |
| `*Clerk` | a class that manages/operates something. Used broadly across Furo apps (`StorageClerk`, `AccessTokenClerk`, `FormElementClerk`) |
| `*Detector` | a classification/lookup helper (`CardBrandDetector`) |

## Class shape

`export default class`, a `constructor ({ ... })` taking a single destructured object, and a
static `create()` factory that is **the intended construction path (constructors are not
called directly)**. The factory uses the self-typing idiom shared across Furo apps
(identical in Context classes).

```js
export default class TimerClerk {
  /**
   * Constructor.
   *
   * @param {{
   *   callback: Function
   *   timeInMilliseconds: number
   * }} params - Parameters
   */
  constructor ({
    callback,
    timeInMilliseconds,
  }) {
    this.callback = callback
    this.timeInMilliseconds = timeInMilliseconds
    this.lastTimer = null
  }

  /**
   * Factory method to create a new instance of this class.
   *
   * @template {X extends typeof TimerClerk ? X : never} T, X
   * @param {{
   *   callback: Function
   *   timeInMilliseconds: number
   * }} params - Parameters
   * @returns {InstanceType<T>}
   * @this {T}
   */
  static create ({
    callback,
    timeInMilliseconds,
  }) {
    return /** @type {InstanceType<T>} */ (
      new this({
        callback,
        timeInMilliseconds,
      })
    )
  }
}
```

A property not supplied by params is initialized to its empty value in the constructor
(`this.lastTimer = null`).

## Dependency injection with factory defaults

`create()` can default collaborators, making the class testable by injection. Type
`FactoryParams` as `Partial<...Params>` (or `RequiredExcept`) accordingly.

```js
static create ({
  resolveCardBrand = determineCardType,
  cardBrands = determineCardType.types,
} = {}) {
  return /** @type {InstanceType<T>} */ (
    new this({
      resolveCardBrand,
      cardBrands,
    })
  )
}
```

Typedefs sit **at the file bottom**, in the `Params` / `FactoryParams` pair:

```js
/**
 * @typedef {{
 *   resolveCardBrand: typeof determineCardType
 *   cardBrands: typeof determineCardType.types
 * }} CardBrandDetectorParams
 */

/**
 * @typedef {Partial<CardBrandDetectorParams>} CardBrandDetectorFactoryParams
 */
```

## Consumption

Instantiate anywhere via `X.create(...)`:

```js
const timerClerk = TimerClerk.create({
  callback,
  timeInMilliseconds: 3000,
})
```

## Not in the source — resolve elsewhere

- **Testing.** The skill says nothing about how a module class is tested or where its test
  file goes. Use `hoc-jest` and this repository's own layout: tests mirror the source path
  under `tests/__tests__/node/app/modules/<ClassName>.js`, or
  `tests/__tests__/jsdom/app/modules/<ClassName>.js` when the class touches the DOM
  (`window`, `document`, browser storage, navigation).
- **`app/modules/` does not exist yet** in `expense-note-frontend-staff`; the first module
  creates it.

## Conflicts with `D:\ORT\rules\` — the rule wins, unresolved here

1. **`get~` prefix.** The skill's DI example names an injected collaborator
   `getCardTypeDefinition` and reads `determineCardType.getTypeInfo`. `naming.md` bans the
   `get~` prefix outright and bans `info` as a name part. The rule wins; the skill's
   example names are not a licence.
2. **Constructor `@param`.** The skill inlines the object type in the constructor's
   `@param`. `jsdoc.md` says the constructor takes `@param {XxxParams} params` pointing at
   the end-of-file typedef, and requires a named typedef once an object type has ≥ 4
   fields. The rule wins.
3. **Trailing `- Parameters` description** on `@param` appears in the skill but not in
   `jsdoc.md`'s format. The rule wins.
4. **Collaborator defaults.** `architecture.md` requires a collaborator's `.create()` to be
   wrapped in a `static create<Collaborator>()` helper and defaulted as `this.createXxx()`,
   never called inline in the default. The skill's DI example defaults to imported values
   rather than `.create()` calls, so it does not demonstrate the required wrapper. The rule
   wins.

No conflict on: `static create()` required, at least one stored property, no static-only
classes, one class per file, `Params` / `FactoryParams` typedefs at end of file — the skill
and the rules agree.
