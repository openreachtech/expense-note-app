# hoc-jest
<!-- hora-skills-ort-core 0.2.0 -->
<!-- source: .claude/skills/hoc-jest/ -->

**Read the source above whenever this leaves a question open.**

> ⚠ **This skill is heavily overridden in this project.** The always-on rule
> `D:/ORT/rules/testing.md` is long, strict and **wins wherever the two disagree** (Q10).
> Every `⚠ OVERRIDDEN` block below marks a place where following the skill's text would be
> wrong here. Where no block appears, the skill's rule stands and the rule file is silent.

## Core principles

**QA stance — never cut coverage using implementation knowledge.** A test must fail if the
implementation is wrong. Reading the implementation and concluding "this is always X" or
"this member is trivial", then omitting a case/member or collapsing a `test.each()` into one
`test()`, is a QA mistake — a hard-coded implementation would still pass. Drop a case only
when the input **cannot exist under the contract**, never because you know today's output.

**No logic in test files.** A test is assertions on simple input/output values only. Only
things **already tested elsewhere** may be used inside a test (already-tested classes,
factory methods, literal data, tested helpers under `tests/tools/`). Untested transformations,
branches and helper definitions are banned — test code is not itself verified, so wrong logic
in it goes unnoticed.

**Index levels 1 and 2 by definition name.** `describe(class) > describe(member) >
describe(behavior) > test()`. Never put behavior (`should …` / `when …`) at level 1 or 2.
The class-name `describe()` is **repeated per member** — one member per class describe — so
the class name always sits directly above the member describe (a syntactic sticky header).
`jest/no-identical-title` is off, so this passes lint.

**Comments inside test code are written in English** (`// same reference`,
`// neutral value; not under test`).

## describe structure and member notation

| notation | member |
| :-- | :-- |
| `#instanceProperty` | instance property |
| `#instanceMethod()` | instance method |
| `#get:instanceGetter` | instance getter |
| `#set:instanceSetter` | instance setter |
| `.staticProperty` | static property |
| `.staticMethod()` | static method (incl. `.create()`) |
| `.get:staticGetter` | static getter |
| `.set:staticSetter` | static setter |
| `constructor` | constructor |

`#` = instance, `.` = static. Methods keep `()`; getters/setters take the `get:` / `set:`
prefix instead of `()`. Referring to a member **in behavior-layer prose** keeps the `()`:
`describe('should call JSON.stringify() with value and replacer')`, not `JSON.stringify`.

Correct shape — repeat the class describe, one member each:

```js
describe('StaffMemberRefreshToken', () => {
  describe('.isGeneratedToken()', () => {
    describe('should accept a token of the shape the generator mints', () => {
      // cases + test.each
    })
  })
})

describe('StaffMemberRefreshToken', () => {
  describe('.get:tokenPattern', () => {
    // ...
  })
})
```

Incorrect: one `describe('StaffMemberRefreshToken')` wrapping `constructor`, `.create()` and
`#buildX()` together.

### Behavior-layer naming

Name the behavior describe **`should [verb]`** (`should keep property`, `should call
constructor`). When the behavior depends on a precondition, insert `describe('when
[precondition]')` and put `test('should [verb]')` inside it.

> ⚠ **OVERRIDDEN IN THIS PROJECT — `to [verb]` is not banned.** The skill states "Do not use
> `to [verb]`". The always-on rule `D:/ORT/rules/testing.md` states, for the inheritance test:
> "**Both title styles are accepted:** `describe('inheritance')` → `test('should be correct
> class')`, or `describe('super class')` → `test('to be instance of <BaseClass>')` (the latter
> is more common in the codebase)." Always-on rules win here (Q10), and the tree is unanimous:
> **all 19** existing test files use `describe('super class')` → `test('to be instance of
> <Base>')`, and the engine tests use `test('to be fixed value')`. **Follow the tree.**

> ⚠ **OVERRIDDEN IN THIS PROJECT — the constructor behavior describe is `to keep properties`.**
> The skill writes `describe('should keep property')`. The always-on rule states:
> "**`constructor`** → `describe('to keep properties')` → one `describe('#<prop>')` per stored
> property, each asserting `toHaveProperty('<prop>', params.<prop>)`." Always-on wins (Q10).

### Fixed forms for members whose behavior takes no input

| Situation | Form |
| :-- | :-- |
| Class `extends` a base | `describe('super class')` → single `test('to be instance of <Base>')`, asserting `<Sub>.prototype` with `toBeInstanceOf(<Base>)`. Import the base class. Place it **first**, before the member describes |
| Abstract member throws when not overridden | `describe(member) > describe('when not inherited') > test('should throw error')`, single `test()`, `toThrow('<Class>.get:schema must be inherited')` |
| …and the message varies by derived class | Turn the derived classes into `cases`; title `'Builder: $params.Builder.name'` |
| Abstract **instance** member throws | Instantiate first, filling unrelated constructor args with neutral values (`replacer: null, // neutral value`) |
| Static getter / static property holding a **fixed primitive** | `describe(member) > describe('when called as is') > test('to be fixed value')`, no `cases` |
| Fixed value is an **object** (`WeakMap` / `Map` / `Set`) | `test('should be a WeakMap')` + `toBeInstanceOf(WeakMap)` — never `toBe(<literal>)` |

```js
describe('StaffMemberRefreshToken', () => {
  describe('super class', () => {
    test('to be instance of BaseAppRenchanModel', () => {
      const actual = StaffMemberRefreshToken.prototype

      expect(actual)
        .toBeInstanceOf(BaseAppRenchanModel)
    })
  })
})
```

### When `test.each()` is required, and when a single `test()` is allowed

| Member | Form |
| :-- | :-- |
| Method taking **arguments** (static or instance) | `test.each()` with **≥ 2 inputs**. One input would pass against an implementation that ignores the argument. This holds inside every `describe('when …')` branch too |
| **Instance getter** (`get Ctor ()`), or any getter varying with instance state/type | `test.each()` — the variable element is **which instance**. Case the base class plus several derived classes |
| Static getter / static property with a constant value | single `test()` under `when called as is` |
| Inheritance / abstract-throw / default-filling with no arguments | single `test()` |
| One axis is a **neutral collaborator** not involved in the behavior | drive the involved axis with `test.each()`, fix the neutral one |

> ⚠ **A no-arg *instance* member still needs `test.each()`.** The always-on rule states:
> "**Use `test.each()` by default.** Only exceptions: an arg-less static method, and a static
> getter. Even when an instance method takes no args, vary the instance's property patterns
> across cases." Skill and rule agree; the tree does **not** — the boilerplate
> `SessionCredentialGenerator` tests `#generateToken()` with plain `test()`. That is
> pre-existing boilerplate, not precedent. **Case the instance's `factoryParams`.**

### `constructor` and `.create()`

- **`constructor`** → `describe('to keep properties')` → one `describe('#<prop>')` per stored
  property → `cases` / `test.each()`. Assert with `toHaveProperty('<prop>', expected)`, not
  `toBe()`.
- **Isolate the property under test.** Put **only** that property into the case; fill the
  other required constructor args with a **neutral value** when assembling `args` in the body.
- **`.create()`** → `describe('should be instance of own class')` (`toBeInstanceOf`) **and**
  `describe('should be call by constructor')` (constructor-spy delegation):

```js
const SpyClass = globalThis.constructorSpy.spyOn(TargetClass)

SpyClass.create(params)

expect(SpyClass.__spy__)
  .toHaveBeenCalledWith(params)
```

- When `.create()` **derives** what it hands the constructor, assert the spy with the
  **transformed** object, not the raw case input.
- **Defaults filled by `.create()`** → `describe('should fill default <name>')`. With **one**
  omittable argument, a single `test('with no arguments')` suffices. With **two or more**,
  enumerate **every combination in which at least one is omitted** = **2ⁿ − 1** cases
  (all-specified fills no default, so it is excluded). Comment out the omitted keys
  (`// gamma: omitted → default null`); write `expected` as the complete shape reaching the
  constructor; give specified values that **differ from the default**; order cases from most
  specified to fewest, with the all-omitted `params: {}` case **last**.

> ⚠ **OVERRIDDEN — a `.create()` default may be asserted on the instance.** The skill states
> "do not look at the return value's properties" in `.create()` tests. The always-on rule
> allows either: "assert the default was applied — either via the constructor-spy
> (`toHaveBeenCalledWith(expect.objectContaining({ <param>: <default> }))`) or
> `toHaveProperty('<param>', <default>)`. If the default comes from a static supplier
> (`= this.generateCurrentDateTime()`), `jest.spyOn` that supplier, and also assert it was
> called." Always-on wins (Q10). The skill's "do not test property retention in `.create()`"
> (it is the constructor's job) is otherwise kept.

## Case data

### Case field names

> ⚠ **OVERRIDDEN IN THIS PROJECT — do NOT use `input` or `override`.** The skill reserves four
> top-level case properties: `override` / `input` / `tally` / `expected`, and says "Do not
> invent names like `params` / `args`." The always-on rule inverts exactly this. It states:
> "Allowed case fields — **only these**: `params` … `expected` … `factoryParams` … `mock*` …
> `tally` … `label` … `expectedPattern` / `expectedTotalLength`", and "Any other key, inputs
> not under `params`/`factoryParams`, or a `name` field → violation." Always-on wins (Q10),
> and the tree follows it: **every project-written test file** uses `params` /
> `factoryParams` / `expected` / `label`, and only the four untouched boilerplate files under
> `tests/__tests__/app/session/` use `input`. **Write `params`, not `input`.**

The field set to use here:

| field | purpose |
| :-- | :-- |
| `params` | arguments passed to the member under test |
| `factoryParams` | arguments to `.create()`; kept separate from `params` |
| `expected` | the asserted value. Omit where the assertion needs none (`toBeNull` / `toHaveLength(0)` / `toBeInstanceOf` / throw cases) |
| `mock*` | fixtures for stubbed return values (`mockTokenResponse`) |
| `tally` | a value that is **both** what is passed in and what is expected |
| `label` | the title string, only when no field path can identify the case |
| `expectedPattern` / `expectedTotalLength` | non-deterministic output only (id-hash / token generators) |

The skill's `override` role — supplying a stub for an abstract member — has no rule-sanctioned
field name. Carry it as `mockSchema` / `mock*` (a `mock*` fixture is what the rule provides
for a stubbed return value), or fold the value into `factoryParams`. **Not settled** — see the
last section.

`tally` (both skill and rule) is only for when the value **passed to the subject** and the
value **expected in the assertion** are the same value or the same object reference.
Mechanical check: `tally` must appear **both** in Act (`member(tally)`) **and** in Assert
(`expect(...).toXxx(tally)`). Appearing in only one is a misuse — Assert-only means it is
`expected`; Act-only means it is `params`. If the subject wraps (`{ prop: tally }`), extracts,
renames or transforms the value, it is **not** `tally`. If even one case in the array
undergoes a real transformation, treat the whole array as `params` / `expected`.
Reference `.toBe()` on an object carries a trailing `// same reference` comment.

### Case data conventions

- Every element of `cases` is **an object** — never a bare primitive.
- **≥ 2 elements** in principle. One is acceptable only with a specific reason (e.g. only one
  case can be truthy), or where the **collection** fixes the count.
- Primitive values are **unique across elements** — duplicated values let a test pass even
  when the implementation returns a different case's value.
- Numeric ids: **at least 6 digits** (`100001`, `100002`, …) so the trailing index reads off
  the Jest log. Never `id: 1`.
- Strings: **base + index suffix** (`'source-0001'`, `'source-0002'`), base kept identical
  across elements, separator always `-`. Do not switch to a domain-plausible value that breaks
  the format.
- When `expected` is **opaque** (Base64, hash, signature), the index goes on the `params` side
  and `expected` is written as the raw result.
- **A run of words** uses `alpha`, `beta`, `gamma`, … `omega` **mechanically in order**,
  consumed across the whole `cases` array. A single representative token is `omega` — never
  `foo` / `bar`.
- **A string carrying meaning** shows its intent either in the value (`'not-a-function'`,
  `'unparseable-date'`) or via an English inline comment.
- **Derived / magic values** are written as literals with the derivation as a comment
  (`value: '9007199254740993', // Number.MAX_SAFE_INTEGER + 2`) — never computed inline.
  `Number.MAX_SAFE_INTEGER` itself is written as the expression.
- **Integers**: always include a case at `Number.MAX_SAFE_INTEGER` (valid side) and
  `Number.MAX_SAFE_INTEGER + 1` (invalid side), **in addition to** typical values.
- **Finite and small value sets** (≤ ~100: enums, weekdays, HTTP methods, regex-escape
  characters) are **enumerated exhaustively**, not sampled.
- **Internal transformations** (escape / trim / case / encode) need at least one case where
  the output changes if the transformation is missing — never fill `cases` with no-op values.
  The **terminal** method implementing the transformation writes all cases; a **caller**
  samples at least one transforming and one no-op value.
- **Variable-count elements**: cover counts down to 3, 2, **1**, **0**, laid out
  **symmetrically** as a grid against the existing axis — never carved into their own
  describe. Order counts largest → smallest.
- **Typical values first, edges later.** Never fill `cases` with edge cases only;
  `null` / `undefined` / `[]` go at the end.
- **A structural marker at a special position** (start / end / both ends / marker-only) is a
  qualitatively different path → its own behavior describe.
- **Formatting: at most one property key per line.** Structural containers (`params:` holding
  an object) are always broken onto new lines. A single-key leaf payload may be inline
  (`valueHash: { id: 100001 }`); multi-key must expand. Array elements each get their own line
  (single-key elements may be inline). **Exception:** where elements are **flat and the array
  is long** (dozens of two-field rows), write one element per line as a table.
- **Sample callback functions carry no branches** — `(key, value) => '*' + value`, not a
  ternary on the key.

### Splitting axes into describes

| Axis | Form |
| :-- | :-- |
| Valid vs invalid argument values | `describe('with valid values')` / `describe('with invalid values')` — never mixed in one `cases`. A missing property is left **commented out** to make the omission explicit |
| Recursive / nested structure | its own `describe('with nested values')`, nested **≥ 2 levels**, with `expected` also nested so a shallow flattening fails |
| Boolean return | `describe('should be truthy')` / `describe('should be falsy')`, `expected` **omitted**, assertion fixed to `toBeTruthy()` / `toBeFalsy()` |
| Memoization / same reference | `describe('should be memoized')`; bind the **first** call to `expected` in Arrange, the second to `actual`, assert `toBe(expected) // same reference`. Only `params` needed. Where the pool is a **static** property, use a **fresh object key per case** — `jest.restoreAllMocks()` does not reset a static `WeakMap` |
| A mode / flag argument that changes how another is interpreted | `describe('with <arg>: <value>')`; the mode is **omitted from the cases** and written directly into `args` in the body |
| A boolean argument or property | `describe('when #isEnabled:false')` / `describe('when #isEnabled:true')`. `#` prefix only when it is an instance property; a bare method-argument flag is `when mode:true`. Coexisting with a valid/invalid split, the boolean describe goes **inside** |
| A member that **may** depend on a property | still split by that property axis, even when it does **not** depend on it — verifying both modes give the same result is what guarantees independence |
| Call-count patterns | one `describe()` per count (`when called once` / `when called twice` / `when not called`), each pinning every call's arguments with `toHaveBeenNthCalledWith(n, …)` |

### Double loop — property × argument, property × property

When a constructor property **and** a method argument both affect the output, or two
properties do, **default to a double loop**: `describe.each()` (outer) × `test.each()`
(inner), laid out **at least 2 × 2**. A single loop pairing one value with one value cannot
show that each axis independently affects the behavior. Single loop is allowed only when one
axis is a neutral collaborator whose non-involvement the **contract** guarantees.

- Inner cases variable is a prefixed `~Cases` (`betaCases`, `credentialCases`, `bindingsCases`).
- Outer element carries `params` / `tally` / `factoryParams`; `expected` sits **inner** when it
  varies per inner argument, **outer** when it is constant per outer property. `expected` is
  the only reserved property allowed inside the inner `~Cases`.
- Inside a `describe.each()` callback, inner cases identical across every outer case may be
  defined once right before `test.each()`, separated from it by a **blank line**.

```js
describe.each(cases)('alpha: $params.alpha', ({ params, betaCases }) => {
  test.each(betaCases)('beta: $beta', ({ beta, expected }) => {
    const instance = SomeClass.create(params)

    const actual = instance.someMethod({ beta })

    expect(actual)
      .toBe(expected)
  })
})
```

> ⚠ **The instance may NOT be hoisted out of the test body here.** The always-on rule states:
> "The instance under test, every stub/fixture it needs, and `factoryParams` all belong **in
> the case** … and are built **inside the test body** (the Arrange phase) — never declared as a
> shared `const` above or inside a `describe`. This holds for **every** describe", and
> "**Duplicating a value across cases is accepted and preferred**." This contradicts the
> skill's "Define shared fixtures directly under the member describe", its placement rule (1)
> (`const registry = new BoundCtorRegistry(params)` before `test.each()`), and its "pull a
> case-invariant `args` outside `test.each()`". Always-on wins (Q10) — **build the instance
> and its fixtures inside the body.** The skill's ban on **file-scope** shared variables
> (imports only) survives and is the stricter of the two, so keep it.

> ⚠ **One `cases` array feeds exactly one `test.each`.** The always-on rule's corollary:
> "Don't share a single `cases` const across sibling `test.each` calls — give each leaf
> describe its own array (duplication accepted)." The skill's reconciliation section argues
> the opposite ("share or verify nothing") for a cases array a sibling describe audits.
> Always-on wins (Q10); no reconciliation test exists in this tree.

## Test titles

Interpolate **one** `params` field whose value is unique across every case. Never interpolate
`expected`, never two fields, never `.length`.

```js
test.each(cases)('token: $params.token', ({ params }) => { /* ... */ })
test.each(cases)('amount: $params.amount', ({ params, expected }) => { /* ... */ })
```

- Dot as deep as needed (`$params.setting.timezone`); an array element's field uses jest's
  `.0` path on the right, while the label text may read `[0]`:
  `'closedDates[0].closedFrom: $params.closedDates.0.closedFrom'`.
- A whole object or `Date` may be interpolated when it renders uniquely and readably.
- For **opaque** values, first look for a distinguishing sub-field (`$params.<field>.<prop>`).
- `label: $label` is the last resort, only when no field path can identify the case. The field
  must be `label`, never `name`.

> ⚠ **OVERRIDDEN — never title by `expected`, and never interpolate more than one field.** The
> skill's 2ⁿ − 1 default-filling example titles with
> `'alpha: $expected.alpha, beta: $expected.beta, gamma: $expected.gamma'`, on the grounds that
> `params` is sparse there. The always-on rule states: "The `test.each` title interpolates
> **one** field, and its value must differ across every case so names are unique. Never embed
> the **expected** value — failure logs already show Expected/Received; the name must reveal the
> **input**", and marks `'a: $params.a, b: $params.b'` a violation. Always-on wins (Q10). For a
> sparse combination case, use **`label: $label`** — the tree already does this in
> `tests/_orders/Expense/Expense.js` (`label: 'no StaffMemberId'`).

> ⚠ **`$#` is not the sanctioned last resort here.** The skill permits `$#` (jest's row index)
> where no readable identifier exists. The always-on rule names `label` for that role and lists
> `label` among the allowed case fields; `$#` appears nowhere in it. Prefer `label`. Keep the
> skill's stronger point either way: **do not fabricate a readable index by distorting the
> type** (no `{ id: 1 }` standing in for a real `WeakMap`).

## AAA — Arrange, Act, Assert

Exactly three phases, in order, **each separated by one blank line**, and nothing else.

```js
test.each(cases)('tokenByteSize: $factoryParams.tokenByteSize', ({
  factoryParams,
  expected,
}) => {
  const generator = SessionCredentialGenerator.create(factoryParams) // Arrange

  const actual = generator.generateToken() // Act

  expect(actual) // Assert
    .toHaveLength(expected)
})
```

- **Blank lines mark phase boundaries only.** Do not split the interior of one phase — "build
  `args`" and "create the instance" sit adjacent with no blank line between them. The one
  exception: **groups with genuinely different concerns** inside Arrange may be separated
  (stubbing return values vs. creating a spy; creating the subject vs. the
  `args` + `jest.spyOn(args, key)` group — subject first, then the args group).
- **Act is a single call**, captured in a variable. For a throw case, capture the un-awaited
  call as an arrow: `const actual = () => target.method(params)`.
- **Pass a variable to `expect()`, never an expression.** To assert a property of the return
  value, receive the return value in a descriptively named variable (PascalCase for a class),
  then `const actual = <Var>.<prop>` with no blank line between the two.
- **Assemble the argument object in Arrange as `const args = { … }`** and pass the variable in
  Act — do not build an object literal inline in the call. If you are passing an
  already-existing variable unchanged, inline it (`method(params)`).
- **`args` naming:** one build → `args`. **Two or more builds in one test** (constructor share
  + method share) → **destination member name + `Args`**: `constructorArgs`,
  `buildPathnameArgs`. Never role-based names like `propertyArgs` / `methodArgs`.
- **Do not create an `args` that just copies the case 1:1.** If the argument object matches
  `params` exactly, pass `params` directly. Build `args` only when there is something to fill
  in (neutral values) or something to split.
- **If one `params` mixes a constructor property and a method argument, splitting into
  `constructorArgs` / `<methodName>Args` is mandatory** — never pass the whole thing to both.
- **Only `expected` and `tally` may be passed to the matcher.** `params` must not reach the
  matcher. Where the matcher takes multiple arguments, make `expected` an **array** of the
  argument list and spread it: `toHaveBeenCalledWith(...expected)`; for multiple calls, an
  array of per-call arrays spread as `...expected[0]`, `...expected[1]`.
- **Expected-value literals in a single `test()`**: single-line (`'…'` / number / boolean /
  `{ id: 100001 }` / `[Alpha, Beta]`) → write **directly in the matcher**; multi-line
  (multi-key, nested, array of objects) → **bind to `const expected`** and keep the matcher
  line as `.toEqual(expected)`.

> ⚠ **OVERRIDDEN IN THIS PROJECT — the Act variable is `actual`, not `received`.** The skill
> states: "The variable that receives the return value in the Act step … should be named
> `received`, not `actual`." The always-on rule states: "**Act** — a **single** call to the
> member under test, captured in `const actual`", and its examples read `const actual = …` /
> `expect(actual)`. Always-on wins (Q10), and the tree is decisive: **all 21 project-written
> test files** use `const actual`; only the three untouched boilerplate files under
> `tests/__tests__/app/session/` use `received`. **Write `actual`.**

## Mocks and spies

- **Override with `jest.spyOn()` — do not define a test-only derived class.**
  `jest.spyOn(TargetClass, '<name>', 'get').mockReturnValue(…)` for a getter,
  `jest.spyOn(TargetClass, '<method>').mockReturnValue(…)` for a method
  (`.mockImplementation(…)` when the value must vary by argument). Then run and assert against
  the **subject class itself** (`toBeInstanceOf(TargetClass)`).
- **A derived class is acceptable only** when the substitution goes beyond a member to the
  shape of the `class`, or when one `describe()`'s `cases` needs **several** derived classes
  simultaneously (a `#get:Ctor` variable element, per-class error messages).
- **Never `jest.fn()` inline inside `test()`.** Define `args` with a **real** function, then
  `jest.spyOn(args, '<key>')` — `spyOn` calls through to the real implementation by default,
  so no stub implementation is written, and it is restored automatically.
- **A getter that returns an existing function:** spy on the **real function** it returns
  (`jest.spyOn(globalThis, 'btoa')`), not the getter — spying the getter fails to type-check
  (`'get'` collapses to `never`). Only when the real function has no independently spyable
  location may you subclass, override the getter and plant a `jest.fn()`.
- **Combine with `constructorSpy`**: a member replaced by `spyOn` is reached through the
  prototype chain, so the class from `globalThis.constructorSpy.spyOn(Target)` picks up the
  same return value.
- **Spy variable naming:** any variable holding a `jest.fn()` gets a `~Spy` suffix, without
  exception (`btoaSpy`, `fetchSpy`, `normalizeSpy`); as a constant, `BTOA_SPY`.
  `constructorSpy.spyOn()` returns a **class**, so it stays `SpyClass`; binding its
  `SpyClass.__spy__` to a variable does earn the `~Spy` suffix.
- **Pin arguments**: `toHaveBeenCalledWith(…)`, not a bare `toHaveBeenCalled()`
  (`jest/prefer-called-with`). `toHaveBeenCalledTimes(n)` is allowed **only** alongside
  `toHaveBeenNthCalledWith(1..n, …)` pinning every call's arguments.
- **No manual restore** — `tests/setup-after-env.js` registers the single
  `afterEach(() => jest.restoreAllMocks())`.

The always-on rule adds constraints the skill does not carry, and they apply:

- **Mock only when necessary — default to real.** The local DB runs for real against seeders,
  and your own domain methods run for real on the happy path. Mock in only two cases:
  **external systems** (third-party APIs, in success *and* failure tests), and **steering an
  otherwise-unreachable branch** (`mockResolvedValue(null)` to force a not-found guard).
  Stubbing something that could have run for real is a **violation**; if data is missing,
  **add a seeder** — never mock a DB row.
- A method borrowed only as a stub elsewhere must still be exercised **for real** in its own
  `describe`.
- **Seeder data is real and unique.** Reference real seeded ids/tokens from
  `sequelize/seeders/development/*` or `dev-master/*`; never call `Model.findOne` / `update` /
  `findAll` inside a test to fetch or verify.
- A spy whose interaction **is** the point of the test must carry
  `expect(spy).toHaveBeenCalledWith(…)` or `.not.toHaveBeenCalled()`; with no assertion it is
  a violation.

## Prohibitions

**Syntax inside `describe()` / `test()`** — banned outright:

- `if`, ternary `? :`, `??`, short-circuit `||` / `&&`
- higher-order functions (`map` / `filter` / `reduce`) — the moment a **function** is the
  argument, unverified logic can live inside it
- `forEach` and all loops — iterate with `test.each()` / `describe.each()` /
  `expect.each()` from `@openreachtech/jest-expect-each`
- helper function definitions inside a test file (a violation of file responsibility even if
  tested in the same file) — define under `tests/tools/`, test at the mirrored
  `tests/__tests__/tests/tools/…`, then it may be used freely
- `it()` / `it.each()` (also lint-enforced by `jest/consistent-test-it`; both skill and
  always-on rule ban it — the rule states "**No `it()` / `it.each()`** — use `test()` /
  `test.each()` only")
- `test.skip` / `test.only` / `xtest` / `ftest` / `xdescribe` / `fdescribe`
- commented-out tests
- a `describe()` / `test()` title ending in `.`
- a `done` callback (use `async` / `await`)
- `return` from a `test()`, or `export` from a test file
- jasmine globals (`spyOn` / `fail` / `jasmine.*`), `__mocks__` imports
- shared variables at **file scope** — only imports live there. DRY does not apply to test
  code; define fixtures fresh in each `describe()` that uses them

**Matchers** — banned:

- `expect.anything()` and `expect.any(Object)` — they constrain nothing and pass against a
  wrong implementation. Pin a concrete value with `toBe()` / `toEqual()`, or a concrete type
  with `toBeInstanceOf(WeakMap)`. `expect.any(<concrete type>)` is allowed **only** where the
  type is itself the discrimination, it cannot go through `toBeInstanceOf()` (nested inside
  `toHaveProperty()`), and the value is pinned by the assertion beside it — never as the whole
  assertion.
- alias matchers (`toBeCalled`, `toThrowError`) — use `toHaveBeenCalled`, `toThrow`
- comparisons built as expressions — `expect(a).toBe(b)`, not `expect(a === b).toBe(true)`;
  `toBeGreaterThan(b)`, not `expect(a > b).toBe(true)`
- `expect(arr.length).toBe(n)` — use `toHaveLength()`
- a bare `toThrow()` — always pass a message argument
- `mockImplementation(() => Promise.resolve())` — use `mockResolvedValue()` /
  `mockRejectedValue()`
- snapshots (this convention pins concrete values instead)

**Matcher rules from the always-on rule, which apply on top:**

- **One `toEqual`** for the whole returned object — do not pick sub-fields with many separate
  `expect`s. Build the full `expected` (Sequelize `Op` symbols included) and compare once.
- **Allowed matchers:** `toEqual`, `toBe` (primitive / same-reference), `toBeNull`,
  `toBeInstanceOf`, `toHaveProperty`, `toHaveLength`, `toMatch`, `rejects.toThrow`,
  `toHaveBeenCalledWith`.
- **`null` result** → its own `describe`, `.toBeNull()` (never `toEqual(null)`).
- **Empty array** → its own `describe`, `.toHaveLength(0)` (never `toEqual([])`).
- **Arrays of `objectContaining`** → write each element out; no `.map(it => …)`.
- **Never write the literal `undefined`.** Omit the key and leave a `// field: undefined`
  comment. (Note: lint's `no-undefined` is off under `tests/**`, so this is the rule's
  constraint, not lint's.)
- **Non-deterministic output** (generated id-hashes, tokens) is asserted by shape —
  `toMatch(expectedPattern)` plus `toHaveLength(expectedTotalLength)`. One-way hash output
  likewise: `toMatch(/^\$2[aby]\$/u)`, or better, assert the delegation with a spy.
- **Order-independent row sets**: `expect.arrayContaining([...])` in `expected`; a paired
  `expect(actual).toHaveLength(<n>)` is allowed here.

> ⚠ **OVERRIDDEN IN THIS PROJECT — do NOT use `toStrictEqual` (or `toMatchObject`).** The skill
> notes only that `jest/prefer-strict-equal` is off, so "`toEqual()` may be used
> (`toStrictEqual()` is not forced)" — it does not forbid it, and four assertions in
> `tests/__tests__/server/graphql/AdminGraphqlServerEngine.js` and
> `CustomerGraphqlServerEngine.js` use it. The always-on rule states: "**Do NOT use
> `toStrictEqual` or `toMatchObject`** — one `toEqual` with the full `expected` object is
> enough." Always-on wins (Q10). Those four are **boilerplate from the initial
> renchan-boilerplate commit**, not project precedent; the project-written
> `StaffGraphqlServerEngine.js` asserts the same kind of config object with `toEqual`.
> **Write `toEqual`.**

> ⚠ **`toBe(true)` / `toBe(false)` are banned.** Neither skill file states this. The always-on
> rule does: "**Booleans** use `toBeTruthy()` / `toBeFalsy()`, never `toBe(true)` /
> `toBe(false)`. (A boolean inside a full `expected` object is fine.)" It agrees with the
> skill's truthy/falsy describe split, and the whole tree follows it.

> ⚠ **`expect.objectContaining` goes INSIDE the case's `expected`, never inlined into the
> assertion.** Neither skill file settles where it may appear. The always-on rule does:
> "**`expect.objectContaining` / `arrayContaining` / `any` go INSIDE the `expected` value —
> never inlined into the assertion.** In a `test.each`, the matcher is the **value of the
> case's `expected` field**; in a plain `test()`, it's the `const expected` declared in
> Arrange. Either way the assert line stays clean: `expect(actual).toEqual(expected)`."
> `tests/_orders/Expense/Expense.js` is the tree precedent:

```js
const cases = [
  {
    params: {
      StaffMemberId: 10000310,
      amount: 1200,
    },
    expected: expect.objectContaining({ // matcher lives in the case's expected
      StaffMemberId: 10000310,
      amount: 1200,
    }),
  },
]
```

## Errors

Wrap the call in an arrow; never `try` / `catch` in the body.

```js
const actual = () => Expense.create(params)

await expect(actual)
  .rejects
  .toThrow(expected)
```

- **Resolvers** (`server/graphql/resolvers/**`) throw an **error-code string** →
  `toThrow('204.M038.001')`, not an `Error` class.
- **Tool / domain classes** (`app/**`) throw real `Error` objects → `toThrow(SomeError)` /
  `toThrow('message')`.

## Directory layout and imports

The skill mirrors the source tree under `tests/__tests__/`, stripping the source root
(`lib/`) — and it argues at length against co-location, which does not arise here.

> ⚠ **The placement rule in force is the always-on one, and it adds a second tree the skill
> does not know about.** `D:/ORT/rules/testing.md`: "`tests/__tests__/**` → method does **not**
> write … `tests/_orders/**` → method writes to the DB directly or transitively … Mocking the
> persist sub-calls does **not** move it to `__tests__`." Placement is **per method**, so one
> class is usually split across both trees. `tests/_orders/` is grouped **by domain**, its
> files are named plainly (`Expense.js`, not `Expense.test.js`), and **a new `_orders` test
> must be added to its folder's `_.test.js` barrel to run** (`import './Expense.js'`).
> Always-on wins (Q10); the tree already has `tests/_orders/{Expense,SignInAttempt,
> StaffMemberPasswordHash,StaffMemberSecret}/` each with its barrel.

What survives from the skill:

- `tests/__tests__/**` mirrors the source path (`app/session/BaseSessionResult.js` ↔
  `tests/__tests__/app/session/BaseSessionResult.js`).
- A large file **may** be split by method, turning the class name into a directory
  (`tests/__tests__/tools/PathnameBuilder/{constructor,create,buildPathname}.js`). Optional.
- Helpers live under `tests/tools/`, with their tests at the path carried **whole**:
  `tests/tools/makeSample.js` ↔ `tests/__tests__/tests/tools/makeSample.js`.
- **Import the subject with a relative path**, never the `~` alias.
- **Import order: the subject under test first**, then a **blank line**, then base classes and
  collaborators. The tree follows this exactly.

## Types

- Attach `@type` to `cases` **only to resolve or suppress a type error**. If the literal infers
  correctly, it is redundant — omit it.
- **Normal-value series:** declaration only, never the `/** @type {Array<*>} */` **value cast**.
- **Abnormal-value series** (deliberately type-violating `null` / missing keys): declaration
  **plus** the `Array<*>` value cast. Because valid/invalid are already split into separate
  describes, only the abnormal series needs the cast.
- **Dynamic-key arguments** (`Record<string, *>`) are the exception where a normal-value series
  **does** carry a declaration — literals with differing keys otherwise infer to disagreeing
  concrete types.
- **A `@type` declaration on `cases` types every field precisely** — never `expected: *`.
  Derive a field's type from an existing value (`typeof SomeClass`,
  `(typeof ScalarHash)[keyof typeof ScalarHash]`).
- **For normal values, do not cast — pass a real value matching the declared type**
  (`new WeakMap()`, not `/** @type {*} */ ({ id: 100001 })`).
- **A dynamic key taken from a case is narrowed on the `cases` side**
  (`name: keyof typeof ScalarHash`), keeping the access site cast-free. Avoid a
  type-resolution-only intermediate variable and avoid a value-position cast at the access site.
- **A union passed to an overloaded function** (`JSON.stringify`) is cast **inline at the
  argument** with `Parameters<typeof Fn>[N]`.
- Write any as `*`, never `any`.
- **Implicit-any function arguments** are typed with `@param` placed directly above the
  property or `const`. Once `@param` is written, **`@returns` is mandatory**
  (`jsdoc/require-returns`) and the block **must be multi-line** (`jsdoc/multiline-blocks`).
- **The type after assignment goes on the line above** the statement; a temporary
  `/** @type {*} */` cast, when needed to bridge the value, goes on the value side.
- **A test-only derived class extending a generic base** takes
  `/** @extends {SomeBase<*, *>} */` directly above it, with as many `*` as the base has
  class-level `@template` parameters. (The always-on `jsdoc.md` prefers `@augments` over
  `@extends` for **implementation** code, per Q10; for the single-line test-vessel form the
  skill's `@extends` is what the tree would show, and no test file in this tree has one yet —
  see the last section.)
- A clean `eslint` run is **not** proof of type resolution. Confirm with
  `npx tsc -p jsconfig.json --noEmit`.

## Non-class subjects — compressed

A module, a data file and a reconciliation between two collections keep the index rule with a
different thing indexed. Level 1 is: **what the module default-exports** (the class's,
function's, `const`'s or same-named import binding's own name — not the filename; a value with
no name of its own gets a name chosen for what the file is about); **a data file's path** from
the mirroring base, extension included, varying segment written `*`
(`i18n/locales/*/message.json`); or **a noun phrase naming the relation**, for a reconciliation.
Below level 1: a module takes `default export` (stops there — no name to index) or
`named export` → `as <the exported name>`; a data file takes the asserted part's **key path**,
or a bare lower-case noun phrase naming the reading when the whole file is asserted (never
borrow `.` for a reading); a reconciliation takes one describe per collection, in the plural.
A reconciliation closes its chain end to end (case fields against each other, each case against
the declaration, the **unique** case count against the declaration count, the declaration total
against the catalogue), sums the parts rather than merging them, and leaves the describe out
entirely for an empty collection (`test.each([])` throws).

**This project has no such test yet — no module-of-constants, data-file or reconciliation test
exists under `expense-note-backend/tests/`.** Read the source before writing the first one.
full text: `.claude/skills/hoc-jest/references/structure.md#when-the-subject-is-not-a-class`
and `references/naming.md#notation-when-the-subject-is-not-a-class`

## The repo's Jest environment

- **ESM needs the node flag.** `package.json`'s `test` script exports
  `NODE_OPTIONS="$NODE_OPTIONS --experimental-vm-modules"` and `NODE_ENV=development`, then
  runs `./test.sh`.
- **The sanctioned run is `npm_config_script_shell=bash npm test`** from
  `expense-note-backend/`. The script is bash (`export …; ./test.sh`), so on Windows npm's
  default shell cannot run it. **A bare `npx jest` misses the node flags and fails every
  suite.**
- `test.sh` tears down and rebuilds the DB, seeds master, then runs
  `tests/empty/__tests__/` + `tests/empty/_orders/` (absent here; `--passWithNoTests` covers
  it), seeds dev, then `tests/__tests__/` and `tests/_orders/` (the latter with
  `--detectOpenHandles`). Default `--maxWorkers=5`. Narrow a run with
  `npm test -- --seeded <path>`.
- `tests/setup-after-env.js` (via `jest.config.js` `setupFilesAfterEnv`) sets up:
  - `globalThis.jest` — the `@jest/globals` `jest` object
  - `globalThis.constructorSpy` — `ConstructorSpy.create({ jest })` from
    `@openreachtech/jest-constructor-spy`; used as
    `globalThis.constructorSpy.spyOn(TargetClass)` → `SpyClass.__spy__`
  - `globalThis.sequelizeActivator` — the real local test DB, awaited at setup, which is why
    seeder-backed data is reachable
  - the **single global** `afterEach(() => jest.restoreAllMocks())` — **individual tests must
    not add their own `afterEach` restore**, and this does **not** reset static state such as
    a memoization `WeakMap`
- ORT jest extensions available: `expect.each(actual).toBe(expected)` / `.toBe.each(…)` and
  `expect.deepContaining(expected)`. `@types/jest` (dev) resolves `describe` / `test` typing.
- `moduleNameMapper` resolves `~/` to the repo root — but **tests import by relative path**
  (see Directory layout).
- Lint: `npm run lint` (`eslint .`). Under `tests/**/*.js`, `no-undefined`,
  `max-classes-per-file` and the no-static-class `no-restricted-syntax` selector are **off**;
  `jest/no-identical-title`, `jest/prefer-lowercase-title`, `jest/prefer-strict-equal`,
  `require-top-level-describe`, `require-hook`, `no-hooks`, `max-expects` and
  `prefer-expect-assertions` are **off** too.
- Running tests is a task of its own — the equipped `hoc-test-execution` skill covers it and
  is not digested here.

## Settled by the main session — the four the digest raised

**Each question below is kept as the digest asked it, with the answer under it.** Read the
answer; do not re-open the question.

1. **The allowed-matcher list vs. lint.** The always-on rule presents a closed list of nine
   matchers, which omits `toContain`, `toBeGreaterThan`, `toHaveBeenCalledTimes` and
   `toHaveBeenNthCalledWith` — yet `jest/prefer-to-contain` and `jest/prefer-comparison-matcher`
   **require** the first two where they apply, and the skill's call-count convention is built on
   the last two. `toBeTruthy` / `toBeFalsy` are likewise absent from the list while being
   mandated by the same rule file for booleans. Treat the list as "these plus what lint
   requires", and confirm.

   **SETTLED: the list is a preference list, not a closed set.** It cannot be closed, because the
   same rule file mandates `toBeTruthy()` / `toBeFalsy()` for booleans and neither appears on it —
   the rule contradicts a closed reading of itself. So: prefer the nine; a matcher outside them
   needs a reason, and "lint requires it here" or "the rule mandates it elsewhere in the same
   file" both count.

   **Checked against the resolved config rather than the plugin source**, because a plugin file's
   base can be empty with the real rules spread in elsewhere — that trap has already cost this
   project three wrong readings of `id-denylist`. `npx eslint --print-config` on a real test file
   returns **50** `jest/*` rules, and **there is no contradiction**: wherever the two overlap the
   always-on rule is strictly *stricter*. `jest/no-restricted-matchers` is configured `{}`, so
   `toStrictEqual` is lint-legal and rule-forbidden — stricter, not opposed.

   The pressure from `jest/prefer-to-contain` / `jest/prefer-comparison-matcher` mostly does not
   arise: both trigger on `expect(a.includes(b)).toBe(true)` / `expect(a > b).toBe(true)`, and the
   rule's own preferred form — one `toEqual` over the whole object or array — never writes those
   constructs in the first place.
2. **A replacement for the skill's `override` case field.** The skill's `override` (a stub
   value for an abstract member, consumed by `jest.spyOn`) is not in the always-on rule's
   allowed field set, and no tree precedent exists. `mock*` is the nearest sanctioned name.

   **SETTLED: use `mock*`.** The rule's allowed extras include `mock*` for "fixtures for stubbed
   return values", and a stub value for an abstract member consumed by `jest.spyOn` is exactly
   that. Do not introduce `override`: the extras list is the one part of the case shape the rule
   states as a fixed set, and "any other key … → violation".
3. **`@extends` vs `@augments` on a test-only derived class.** Q10 settled `@augments` for
   implementation JSDoc; the skill writes `@extends` for the single-line test-vessel form and no
   test file in this tree has one yet.

   **SETTLED: `@augments`, in test files too.** Lint runs over `tests/`, the always-on `jsdoc.md`
   is not scoped to implementation code, and every file this project has *written* uses
   `@augments` while only inherited boilerplate uses `@extends`. A test-only vessel class is code
   in this repository like any other.
4. **`hoc-jest` has no notion of a DB-backed test.** Seeders, transactions, `_orders`
   placement and "mock only external systems" come entirely from the always-on rule; the skill
   mentions a seeder once, only as an argument against in-test setup logic. Read
   `D:/ORT/rules/testing.md` for anything DB-touching.


   **SETTLED: correct, and the gap is covered elsewhere.** For anything DB-touching read
   `.hora/digests/hor-backend-testing.md` first — it carries the `__tests__` versus `_orders`
   placement rule (per-method, not per-class), the `_.test.js` barrel without which a file
   silently never runs, the two-phase runner, the `bulkInsert`-bypasses-hooks trap, and the
   row-id prefix requirement — and `D:/ORT/rules/testing.md` behind it. **This digest governs
   test *shape*; that one governs test *placement and data*.**