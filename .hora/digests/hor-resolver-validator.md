# hor-resolver-validator
<!-- hora-skills-ort-renchan 0.1.0 -->
<!-- source: .claude/skills/hor-resolver-validator/ -->

**Read the source above whenever this leaves a question open.**

> **Nothing this skill extends exists in this repository.** There is no `BaseInputValidator`, no
> `NormalPaginationInputValidator`, no inspector anywhere in `expense-note-backend/app/` (only
> `constants/`, `globals/`, `session/`, `tools/rateLimit/`), `@openreachtech/renchan` exports none,
> and **`@openreachtech/mentsu-value-inspector` is not installed** (`node_modules/@openreachtech/`
> holds `renchan`, `renchan-env`, `renchan-sequelize`, `mentsu-logger`, `mentsu-mixin-builder`,
> `mentsu-random-text-generator`, three eslint packages, `jest-constructor-spy`). The skill says so
> itself: *"Directory layout differs from the current repo on purpose — this is the target
> convention."* **The base class is written fresh.** Where the skill defers to "the actual base API
> in your framework", that framework is you — decide and record it.

## Grand principle: declare `[predicate, error]` entries, let the base run them

A validator is a `BaseInputValidator` subclass whose **only required override is
`generateValidationEntries()`** — it returns `Array<[() => boolean, RenchanGraphqlErrorCtor]>`. Each
entry pairs a **predicate** (returns `true` when that rule passes) with the **error constructor** to
raise when it fails. The base holds `this.input` and `this.errorHash`, iterates the entries, and
**throws the error of the first failing predicate**.

- **Predicates are small `isValidX()` methods** that read `this.input` and return a boolean. "Never
  throw inside a predicate — returning `false` is how a rule fails; the base turns that into the
  error."
- **Errors come from `this.errorHash`** (the resolver's error hash, injected into the validator), so
  each `InvalidXxx` maps to the resolver's `errorCodeHash` code.
- **Reuse shared validators** (e.g. pagination) by composing them inside entries, not by
  re-implementing limit/offset/sort.

## 1. `BaseInputValidator` — the contract to write

Per `references/validator-pattern.md` ("`BaseInputValidator` contract"), the base lives at
`app/validator/forResolver/BaseInputValidator.js` and provides:

- **Instance state:** `this.input` (the resolver input) and `this.errorHash` (the resolver's error
  constructors), **supplied through the factory** — `create({ input, errorHash })`.
- **`generateValidationEntries()`** — abstract; the subclass returns
  `Array<[() => boolean, RenchanGraphqlErrorCtor]>`.
- **Run entrypoint** — "iterates the entries and, for the first entry whose predicate returns
  `false`, **raises** that `ErrorCtor`. (Shown as `validate()` in the wiring below; use whatever the
  actual base names it.)"
- It exports the `RenchanGraphqlErrorCtor` type that validators import:
  `import('../../BaseInputValidator.js').RenchanGraphqlErrorCtor`.

**Correction to the always-on rule.** `D:/ORT/rules/input-validators.md` says the base provides
`validateInput()`, which *returns* the first failing entry's `.create()`ed error or `null`, and that
the resolver then `throw`s it. **The skill says the opposite twice** — the base "throws the error of
the first failing predicate" / "raises that `ErrorCtor`" — and names the method `validate()`, called
for its effect with no return used. Two incompatible contracts for a class that does not yet exist:
see *Not settled* below. The parts both agree on: constructor state is `{ input, errorHash }`, the
factory is `create({ input, errorHash })`, entries run **in declared order**, and the **first**
failure decides.

## 2. `generateValidationEntries()` — shape and order

Return shape is fixed: one two-element tuple per rule, predicate first as an arrow, error
constructor second, read off `this.errorHash`. Composed sub-validators are built **before** the
`return`, into a named const.

```js
/**
 * Generate validation entries
 *
 * @override
 * @returns {Array<[() => boolean, RenchanGraphqlErrorCtor]>}
 */
generateValidationEntries () {
  const paginationInputValidator = this.createPaginationInputValidator()

  return [
    [
      () => paginationInputValidator.isValidLimit(),
      this.errorHash.InvalidPaginationLimit,
    ],
    [
      () => this.isValidKeyword(),
      this.errorHash.InvalidKeyword,
    ],
    [
      () => this.isValidOriginObjectCategoryId(),
      this.errorHash.InvalidOriginObjectCategoryId,
    ],
  ]
}
```

**Entry order: the skill states no rule.** Its only example orders pagination entries (limit →
offset → sort) ahead of the per-field entries. The always-on `D:/ORT/rules/input-validators.md` does
prescribe an order, and nothing in the skill contradicts it, so follow it: **required/presence →
basic type → inspector checks → format/range → composite (cross-field)**. Order is load-bearing,
because only the first failing entry's error is raised.

## 3. Per-field predicates

- **One `isValidX()` per rule**, returning a `boolean`, reading `this.input`. PascalCase suffix
  matching the field (`isValidKeyword`, `isValidOriginObjectCategoryId`).
- **`areValid***()` for an array field** comes from the always-on rule; **the skill never mentions
  `areValid`** and shows no array-field predicate at all.
- **Optional field idiom** — destructure with a default and short-circuit `true` when absent, then
  check the shape:

```js
isValidKeyword () {
  const {
    keyword = null,
  } = this.input

  if (!keyword) {
    return true
  }

  return typeof keyword === 'string'
}
```

- **Value-shape checks go through an inspector**, built by a small `create*ValueInspector({ value })`
  helper on the validator (the seam; place it below its caller):

```js
isValidOriginObjectCategoryId () {
  const inspector = this.createIntegerValueInspector({
    value: this.input.originObjectCategoryId,
  })

  // the module has no isPositiveInteger — compose it (string-tolerant *Like)
  return inspector.isIntegerLike()
    && inspector.isPositiveNumberLike()
}

createIntegerValueInspector ({
  value,
}) {
  return IntegerValueInspector.create({
    value,
  })
}
```

## 4. Resolver wiring

The resolver declares the codes, then runs the validator with its own `input` and `errorHash`:

```js
/** @override */
async resolve ({
  variables: {
    input,
  },
  context,
}) {
  EmailInsertableVariablesInputValidator
    .create({
      input,
      errorHash: this.errorHash,
    })
    .validate()

  // ... input is now trusted ...
}
```

- **The skill chains `.create({...}).validate()` and uses no return value** — the base raises. (Note
  this collides with `D:/ORT/rules/javascript-style.md` "assign a factory instance to a variable
  before calling its methods"; the always-on rule's `const validationError = ...validateInput()` +
  `if (validationError) { throw validationError }` form satisfies both. Resolve with §1.)
- **"The error names in the validator's `ErrorHash` typedef must match keys in the resolver's
  `errorCodeHash`."**
- **Error-code prefix: use this repository's, not the skill's.** The skill's examples read
  `'400.C001.001'`. ORT's always-on convention and this repository's allocated ids
  (`server/graphql/resolver-id-hash-staff.js`: `signIn: 'M001'`, ...) give **`203` =
  input-validator** → `'203.M001.001'`.

## 5. Directory & naming

| | |
| :-- | :-- |
| Validator | `app/validator/forResolver/<endpoint>/{queries,mutations}/<Operation>InputValidator.js` |
| `<endpoint>` | the GraphQL endpoint the resolver belongs to (`user`, `customer`, `admin`, ... — here `staff`) |
| Base + shared | `app/validator/forResolver/BaseInputValidator.js`, `NormalPaginationInputValidator.js` |
| Class name | `<Operation>InputValidator`, PascalCase, **matching the resolver's `schema`**; one class per file, `export default` |
| Inspectors | imported directly from `@openreachtech/mentsu-value-inspector` — "it is a published module — use it from the package, not a local wrapper" |

`NormalPaginationInputValidator.js` is **default-exported and commonly imported as
`PaginationInputValidator`**.

**Base placement satisfies `npm-package.md`.** The rule "a `Base~` class never sits in the same
folder as its concrete subclasses" holds: the base sits at `app/validator/forResolver/`, two levels
above the concretes in `<endpoint>/{queries,mutations}/`. The skill's own import paths confirm it —
`import BaseInputValidator from '../../BaseInputValidator.js'`.

**Path conflict with the always-on rule — unresolved.** `D:/ORT/rules/input-validators.md` puts
validators at `app/tools/validator/resolvers/<audience>/{mutations|queries}/`; the skill puts them at
`app/validator/forResolver/<endpoint>/{queries,mutations}/` and calls that "the target convention".
Neither path exists in this repository. Pick one in the main session and record it; the `Base~`
placement above works unchanged under either.

## 6. The inspector API (`@openreachtech/mentsu-value-inspector@1.1.0`)

Three classes, each a named root export; each subclass inherits its parents' methods, so
**instantiate the most specific one you need**:

```
ValueInspector            // presence checks
└── NumberValueInspector  // + number checks + normalizeValue()
    └── IntegerValueInspector  // + integer / safe-integer checks
```

There is **no `IntegerNumberValueInspector`** (the always-on rule's name for it is wrong) and **no
`PaginationInputValidator` in the package** — that one is a shared local class.

| Flavor | Reads | Accepts numeric strings |
| :-- | :-- | :-- |
| **strict** (`isNumber`, `isInteger`, `isPositiveNumber`, ...) | the raw value | No — `isNumber('3.14')` → `false` |
| **`*Like`** (`isNumberLike`, `isIntegerLike`, ...) | the normalized value | Yes — `isNumberLike('3.14')` → `true` |

**At the GraphQL/REST boundary prefer the `*Like` checks.** `*Like` normalization: `'100'` → `100`,
while `true` / `1000n` / `{}` / `'abc'` / `null` / `undefined` → `null` and every `*Like` returns
`false`. **`''` normalizes to `0`, so `isIntegerLike('')` → `true`** — guard empty string explicitly
for required numeric fields.

- **Presence (`ValueInspector`):** `isNull()`, `isUndefined()`, `isNullish()`, `isDefined()` (not
  `undefined`), `isPresent()` (neither `null` nor `undefined`).
- **Numbers (`NumberValueInspector`):** `isNumber()/isNumberLike()` (finite),
  `isPositiveNumber()/...Like()` (`> 0`), `isNegativeNumber()/...Like()` (`< 0`), `isZero()` (incl.
  `-0`), `isNaN()`, `isFinite()`, `isInfinite()`, `normalizeValue()` → normalized `number` or `null`
  (lazy + memoized).
- **Integers (`IntegerValueInspector`):** `isInteger()/isIntegerLike()`,
  `isSafeInteger()/isSafeIntegerLike()` (safe integer, `±(2^53 − 1)`).
- **Positive integer must be composed:** `isIntegerLike() && isPositiveNumberLike()`; **for an id use
  `isSafeIntegerLike() && isPositiveNumberLike()`**.
- **Read the coerced number after validating with `normalizeValue()` — don't re-parse.**

full text: `.claude/skills/hor-resolver-validator/references/inspector-api.md#behavior-cheat-sheet`
(the per-input behavior table is dropped here).

## 7. Validating a string — what the skill actually covers

This is thin in the source, so take it literally rather than by extrapolation.

| Check | What the skill gives |
| :-- | :-- |
| presence | `ValueInspector#isPresent()` — "**Required present:** `inspector.isPresent()`" |
| absent-and-optional | the `if (!keyword) { return true }` idiom (§3) |
| is-a-string | plain `typeof keyword === 'string'` — no inspector method for it |
| emptiness | **nothing**, for a string. The only empty-string guidance is the numeric trap (`''` → `0`) |
| format (email, pattern) | **nothing.** No regex, no format method, no example anywhere in the skill or the inspector API |
| maximum length | **nothing.** No length method, no example |

So for `signIn(input: { email: String!, password: String! })`: presence and `typeof ... === 'string'`
are covered conventions; an **email-format check and a password byte-length cap are yours to write as
plain `isValidX()` predicates**, which the pattern fully permits (a predicate is any boolean method
reading `this.input`) but does not demonstrate. Q32's 72-byte bcrypt cap belongs in exactly such a
predicate — note it is a **byte** limit, so measure bytes, not `String#length`; the skill offers no
helper either way.

## 8. Distinct malformed values get distinct checks — identical wording is not licence to merge

Spec §10's requirement that "an address with no account and a correct address with the wrong password
are refused identically" is about **the outcome of a lookup**, and about **what the caller learns**.
It is not about input validation, and it is **never** a reason to collapse validation branches.

- **A malformed input may legitimately be refused with its own distinct code** — a missing email, a
  non-string email, a malformed email and an over-long password are **four rules, so four
  `isValidX()` predicates and four entries**, each with its own `InvalidXxx`.
- The skill pushes the same way: one predicate per rule, one entry per predicate. Its "first failing
  predicate wins" behavior controls only *which* error surfaces, not how many you declare.
- If sameness of the client-visible refusal is required, that is a property of the codes/messages
  chosen for a given outcome — decided per operation, never a reason to reduce the check count.

## 9. Testing

"Predicates are pure booleans, so unit-test them (and `generateValidationEntries`) ... no resolver,
no DB. **Instantiate the validator with a stub `errorHash`** (`errorHash: {}`) and assert each
`isValidX()`, and assert that invalid input raises the matching `InvalidXxx` through the run
entrypoint."

- The skill defers to a **`hoc-jest`** skill; its digest is at `.hora/digests/hoc-jest.md`, alongside
  `.hora/digests/hor-backend-testing.md`. A validator writes nothing, so tests go under
  `tests/__tests__/**` mirroring the source path.
- **The skill's test snippet is corrupted in the source file** (the `cases` array is textually
  scrambled mid-literal and wrapped in `/* eslint-disable */`). Do not copy it. Its visible style also
  violates the always-on testing rule (`'input $input'` as a title, `.toBe(expected)` on a boolean):
  follow `D:/ORT/rules/testing.md` — `toBeTruthy()`/`toBeFalsy()`, one input field in the title, AAA
  with one Act.

## Settled by the main session — read the answers, do not re-open the questions

**Every item below is kept as the digest raised it, with the ruling under it.** All six follow one
principle already recorded as Q10: **where an equipped skill and an always-on rule in
`D:/ORT/rules/` disagree, the always-on rule wins.**

**RULING 1 — the entrypoint is `validateInput()` and it RETURNS the error or `null`.** Not
`validate()`, and it does not throw. The always-on rule states both the name and the contract, and
the skill explicitly punts ("use whatever the actual base names it"), so there is no real conflict:
a rule that decides beats a skill that declines to. The resolver side reads

```js
const validationError = validator.validateInput()

if (validationError) {
  throw validationError
}
```

which is the always-on rule's own form, and it also satisfies the "name the instance before calling
its methods" rule that the skill's chained `.create(…).validate()` breaks.

**RULING 2 — the base `.create()`s the error; entries carry the error CLASS.** This falls out of
ruling 1, and the always-on rule says it directly: `validateInput()` "returns the first failing
entry's `.create()`ed error". So an entry is `[() => this.isValidX(), this.errorHash.InvalidX]` — a
constructor — and the base is what calls `.create()` on it.

**RULING 3 — the path is `app/tools/validator/`.** The always-on rule's
`app/tools/validator/resolvers/<audience>/{mutations|queries}/<Schema>InputValidator.js` wins over
the skill's `app/validator/forResolver/<endpoint>/`. Two reasons beyond Q10: `directory-structure.md`
makes `app/tools/<Concept>/` the home for a reusable domain concept, and checkpoint 5 already
created `app/tools/` for the rate limits — so this keeps one tools tree rather than opening a second
top-level directory. **The base goes at `app/tools/validator/BaseInputValidator.js`**, three levels
above the concretes, which satisfies `npm-package.md`'s rule that a `Base~` never sits beside its
subclasses.

**RULING 4 — only `signIn` gets a validator.** Your reading is right and is hereby confirmed.
`signOut`, `renewAccessToken` and `signedInStaffMember` take **no argument at all** — the convention
gives an operation with no input no argument, so no empty input type exists in the SDL and GraphQL
rejects any argument passed to one. There is no field to declare a rule for, and a validator holding
zero entries is machinery for nothing.

**This is not scope being trimmed, and the distinction matters**: checkpoint 6's exit condition says
"its input is validated", and an operation with no input satisfies that vacuously rather than
incompletely. What those three operations must still do is **authenticate** — by the refresh-token
cookie — and that is not input validation and does not belong in a validator.

**RULING 5 — a string format uses a module-level regex constant, and the length cap counts BYTES.**
No inspector exists for either, in the package or the skill, and none is to be invented as a shared
helper for one caller. So: a `SCREAMING_SNAKE_CASE` regex const declared right after the imports
(`javascript-style.md`), and a predicate reading it. **For the password cap, measure bytes, not
`String#length`** — `Buffer.byteLength(password, 'utf8')` — because bcrypt's limit is 72 bytes and a
multi-byte character makes those differ. Q32 records why the cap exists at all.

**RULING 6 — `areValid***()` is not needed here and is not to be written.** `SignInInput` has no
array field. The name stays reserved for the always-on rule's shape when `#expense-entry` needs it.

---

### The original questions, as the digest raised them

- **The run entrypoint's name and contract.** `validate()` that **throws** (skill) vs
  `validateInput()` that **returns the error or `null`** (always-on rule). The skill explicitly
  punts: "use whatever the actual base names it" / "align with the actual base API". Since the base
  is being written here, this is a decision, not a lookup. Whichever is chosen, the resolver side
  (§4) must match it.
- **The validator directory** — `app/validator/forResolver/` (skill) vs
  `app/tools/validator/resolvers/` (always-on rule). See §5.
- **Whether an operation with no input gets a validator at all.** The skill does not address it.
  Every mechanism it defines is input-shaped: `create({ input, errorHash })`, `this.input`, one
  predicate per input field. For `signOut`, `renewAccessToken` and `signedInStaffMember` — which take
  no argument (no empty input type in the SDL) — there is no field to declare a rule for, so a
  validator would hold zero entries. **The reading the skill supports is: no validator for those
  three; only `signIn` gets one.** It does not say so, so confirm rather than assume.
- **Whether `errorHash` entries are `.create()`d by the base or thrown as constructors.** The skill
  puts `this.errorHash.InvalidKeyword` (a constructor, per `RenchanGraphqlErrorCtor`) into the entry
  and says the base "raises" it; the always-on rule says the base `.create()`s it. Falls out of the
  §1 decision.
- **A string-format or length inspector.** None exists in the package or the skill (§7) — decide
  whether such predicates use bare JS, a module-level regex const, or a new shared helper.
- **`areValid***()` for array fields** — named only by the always-on rule, absent from the skill.

## Finishing checklist

- [ ] Class at the agreed path, named `<Operation>InputValidator` matching the resolver's `schema`,
      one class per file, `export default`, extends `BaseInputValidator`?
- [ ] `generateValidationEntries()` marked `@override`, returning
      `Array<[() => boolean, RenchanGraphqlErrorCtor]>`, entries ordered presence → type → inspector
      → format/range → composite?
- [ ] One `isValidX()` per rule, boolean, reading `this.input`, **never throwing**?
- [ ] Every distinct malformed value its own rule — no merged checks (§8)?
- [ ] Optional fields short-circuit `true` when absent; required numeric fields guard `''`?
- [ ] Inspector reached through a `create*ValueInspector({ value })` helper, `*Like` methods at the
      boundary, positive-integer composed?
- [ ] Every `InvalidXxx` used has a matching key in the resolver's `errorCodeHash`, prefixed `203`
      with this repository's resolver id?
- [ ] `ErrorHash` and input `@typedef`s at the end of the file;
      `@extends {BaseInputValidator<ErrorHash, XxxInput>}` on the class?
- [ ] Predicates unit-tested with a stub `errorHash`, under `tests/__tests__/**`?
