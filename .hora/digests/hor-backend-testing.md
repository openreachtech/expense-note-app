# hor-backend-testing
<!-- hora-skills-ort-renchan 0.1.0 -->
<!-- source: .claude/skills/hor-backend-testing/ -->

**Read the source above whenever this leaves a question open.**

> **Scope.** This skill governs test **organization and discipline** — placement, run order,
> running, purity. The **writing style** of an individual test (describe/test structure, AAA,
> `cases` shape, title interpolation, matchers) is explicitly **out of scope here** and is set by
> the always-on rule `D:/ORT/rules/testing.md`. Where the two touch the same question, the
> always-on rule wins (Q10).

## Core principle

A test must fail **only** because the code it exercises is wrong — not because another test ran
first, and not because the test itself contains untested logic. Everything below follows from that.

## 1. Placement — `tests/__tests__/` vs `tests/_orders/`

| Location | Contents | Ordering |
| --- | --- | --- |
| `tests/__tests__/` | tests that **do not** write to the DB — pure units, read-only queries, formatters, validators, getters, builders | none needed; any order |
| `tests/_orders/<Category>/` | tests that **do** write to the DB — anything that inserts/updates/deletes rows | guaranteed **within a category** by a `_.test.js` barrel |

- **Classify by what the method _does_, not by whether the test mocks the write away.** A method
  that writes directly or **transitively** (calls `create` / `update` / `destroy` /
  `beginTransaction`, or orchestrates sub-methods that do) belongs in `_orders` **even if the test
  stubs the persist call** — mocking the write does **not** move it to `__tests__`.
- **Placement is per-method, so one class usually splits across both trees.** `Expense` is the
  live precedent: `tests/_orders/Expense/Expense.js` (writes) **and**
  `tests/__tests__/sequelize/models/Expense.js` (`.createAttributes()`, superclass). Do not dump a
  read-only `find~` into `_orders` just to keep it beside the class's other tests — co-location is
  not a placement reason.
- `tests/__tests__/` **mirrors the source tree** (`sequelize/models/Expense.js` →
  `tests/__tests__/sequelize/models/Expense.js`).
- `tests/_orders/` is grouped **by domain / category**, not by source path — one folder per
  cohesive area of write behavior (`Expense/`, `SignInAttempt/`, `StaffMemberSecret/`,
  `StaffMemberPasswordHash/`).
- Test files under `_orders` are named **plainly** (`Expense.js`, never `Expense.test.js`) so Jest
  does not discover them independently; the barrel is the category's single entry point.

## 2. The `_.test.js` barrel — a file not listed there never executes

Each category folder holds a `_.test.js` that **imports its test files in the order they must
run**. Jest discovers `_.test.js`; its import order *is* the execution order for the category.

```js
// tests/_orders/Expense/_.test.js
import './Expense.js'
```

- **Adding a `_orders` test means adding one `import` line to its category's barrel**, at the
  position where it must run (e.g. after the test that creates the row it depends on). Omit it and
  the file silently never runs — no failure, no output.
- A **new category** needs a new folder **and** its own `_.test.js`.
- **Cross-category order is NOT guaranteed.** `_orders` fixes order within a category only. Two
  interfering categories are isolated into **separate CI jobs**, each against a fresh database — in
  this repository `expense-note-backend/.github/workflows/test-with-sqlite.yml` re-invokes
  `npm test` once per phase, and each invocation rebuilds, so each phase starts clean.

## 3. Run order and isolation in this repository

- **`_orders` tests here insert fixed explicit row ids, so they are not idempotent across bare
  jest runs.** A second run without a database rebuild fails on primary-key collisions.
  `tests/_orders/StaffMemberSecret/StaffMemberSecret.js` and
  `tests/_orders/SignInAttempt/SignInAttempt.js` have the identical trait. **This is the house
  pattern, not a defect to fix** — fixed ids are what let the expectation be a literal.
- **`npm test` rebuilds first**, which is what makes the pattern safe: `test.sh` runs teardown →
  migrate → `db:seed:master`, then `db:seed:dev` before the seeded phase (the same steps as
  `npm run db:refresh` / `npm run r`).
- **Sanctioned command, from the repository root:**

  ```
  npm_config_script_shell=bash npm test
  ```

  The `test` script uses `export` and `./test.sh`, so it needs bash; without the override it fails
  under PowerShell.
- **A bare `npx jest` fails every suite with a module error** — it misses
  `--experimental-vm-modules`, which the `test` script exports and which
  `tests/setup-after-env.js` (ESM, top-level `await`) requires.
- Fast single-file loop (no rebuild; runs against the DB **as it currently is**, so prepare it once
  first). Add `--runInBand` for a DB-writing or order-sensitive file:

  ```bash
  NODE_OPTIONS="--experimental-vm-modules" NODE_ENV=development npx jest --runInBand <path>
  ```

  After running a `_orders` file this way the database no longer matches its baseline — restore it
  (`npm run r`, or undo → re-apply the one seeder set) before trusting a later run.
- Through the runner, one path with a chosen seed set: `./test.sh --seeded <path>` (development
  fixtures applied) / `./test.sh --empty <path>` (master seeds only). `test.sh` also references
  `tests/empty/**`, which does not exist here — `--passWithNoTests` covers it.
- Live (real-dialect) run: `NODE_ENV=live` selects MariaDB and **requires a MariaDB running
  locally**; the same test files run unchanged. `npm run test:live` → `./test-live.sh`.
- **Parallelism × per-worker heap must fit in real memory.** `--max-old-space-size` is *not* a
  reservation — it is how far the GC may be deferred, so a value the machine cannot give lets a
  worker grow until the OS kills it (silently, taking the run with it). Invariant:
  `workers × (heap cap + overhead) ≤ memory actually free`. Derive the cap from a measured peak
  (`--logHeapUsage`), lower `--maxWorkers` first under pressure, and go `--runInBand` when one
  worker's peak already fills what is free. `test.sh` defaults to `--maxWorkers=5`; CI uses `3`.
  full text: references/running-tests.md#parallelism-and-memory-the-worker-budget

## 4. Seeder rows vs rows the test creates

The always-on rule: DB-touching tests use **real seeded data** from
`sequelize/seeders/development/*` or `dev-master/*`; if the data is missing, **add a seeder — never
mock a row**. It also permits "a real created record". Which applies:

| Situation | Use |
| --- | --- |
| The method reads pre-existing rows (a `find~`, a read-only query, a resolver reading masters) | **seeded rows**, referenced by their real seeded ids |
| The method under test **is** the write (a model hook, `.create()`, `.bulkCreate()`, a save orchestrator) | **rows the test creates**, with explicit ids from the feature's prefix |
| A row the write needs as a parent (an FK target — this project has no DB-level FK constraints) | create it in the Arrange phase, as `Expense.js` creates its `StaffMember` and `ExpenseCategory` |

- **Trap: `queryInterface.bulkInsert` bypasses Sequelize model hooks entirely.** A seeder therefore
  **cannot exercise a hook**, and must write **already-transformed** values itself (which is why a
  seeder needing a computed value — a password hash — builds a fulfilled array before
  `bulkInsert`). A hook's behavior is proved only by a `_orders` test going through the model:
  `StaffMemberSecret.create({ email: 'Carla@Example.COM' })` asserted as `carla@example.com`.
- **Do not hand-query the DB inside a test to verify.** Assert the return value of the method under
  test; never call `Model.findOne` / `findAll` / `update` in the body to fetch or check state — that
  tests the ORM. (The one place a `findAll` *is* the Act: a **seeder's own** test, whose contract is
  the seeded rows — `tests/__tests__/sequelize/seeders/master/expense_categories.js`.)

## 5. Explicit row ids come from the feature's allocated prefix

- Every explicit id a test writes uses the **id prefix allocated to the feature**, handed to the
  agent in its assignment — **never invented, never taken from another feature's rows**.
- Registry: `expense-note-backend/.hora/id-bank.json` (currently `data-model` → `100`,
  `sign-in` → `101`). Live shape: `#data-model` rows read `10000201`, `10000310`; `#sign-in` rows
  read `10100101`, `10100112`.

## 6. Mocking policy — default to real

- **A method that *can* run for real must not be mocked.** The local DB runs for real against
  seeders, and your own domain methods run for real on the happy path. Mocking the behavior the
  test exists to verify lets a hard-coded or regressed implementation still pass.
- **The only two sanctioned cases:**
  1. **External systems** — third-party APIs that must never be hit, in success *and* failure tests.
  2. **Steering an otherwise-unreachable branch** — e.g. `mockResolvedValue(null)` to force a
     not-found guard that seeded data cannot naturally trigger.
- **Over-mocking is a violation**: stubbing a call that could have run for real, or mocking DB rows
  instead of adding a seeder.
- **A spy whose interaction is the point of the test must be asserted** —
  `toHaveBeenCalledWith(...)` or `.not.toHaveBeenCalled()`. With no assertion it is a violation. A
  spy used *purely* to steer a branch is verified by the resulting `rejects.toThrow(...)`; adding
  the call assertion is still better.
- Mocks are set up **inline in the test body** (`jest.spyOn(...)`). The repo uses **no**
  `beforeEach` / `afterEach` / `beforeAll` for mock setup — `tests/setup-after-env.js` registers
  the single global `afterEach(() => jest.restoreAllMocks())` and sets `globalThis.constructorSpy`,
  `globalThis.sequelizeActivator`, `globalThis.jest`.
- A double that is *only ever* borrowed as a stub must still be exercised **for real** in its own
  test.

## 7. Test purity & doubles

- **No new logic in a test.** Never define a function or class inside a test to compute an expected
  value or reproduce production behavior — untested logic makes the test lie. Assert against
  **literals**.
- **Everything a test leans on must already be tested.** Exercising production code A that calls B
  is fine (B has its own tests); what is banned is **new** code that exists only to serve the test
  and has no tests of its own.
- **Shared doubles have a home and are themselves tested:** `tests/mocks/` (mock/stub classes
  standing in for a collaborator) and `tests/tools/` (shared factories, builders, fixtures).
  **Each mock class has its own test file.** Prefer explicit fakes — obviously-fake names and
  values. A double used by exactly one test may sit beside it; the moment a second test needs it,
  move it (with its own test) rather than copy it. *Neither directory exists in this repository
  yet — creating one means creating its test file too.*

> ⚠ **PARTLY OVERRIDDEN — a small stub stays inline in the case.** The always-on
> `D:/ORT/rules/testing.md` requires that **nothing be hoisted to describe scope**: an injected
> collaborator stub goes into each case's `params` (`japaneseHolidayCalculator: { isHoliday: () =>
> false }`), duplicated per case, never a shared `const`. The always-on rule wins (Q10), so
> `tests/mocks/` / `tests/tools/` is for a **genuine shared mock class or factory** — not the
> default home for a one-line stub.

## 8. Resolver vs tool/domain class

**The skill is silent on resolvers** — it governs organization only. What follows is the always-on
`D:/ORT/rules/graphql-resolvers.md` + `testing.md`, which is what this project follows:

| Under test | A thrown failure is asserted as |
| --- | --- |
| a resolver (`server/graphql/resolvers/**`) | the **error-code string** — `toThrow('204.M038.001')`, never an `Error` class |
| a tool / domain class (`app/**`), a model | a real `Error` — `toThrow(SomeError)` / `toThrow('message')` / `toThrow(UniqueConstraintError)` (as `StaffMemberSecret.js` does) |

- Error-code prefixes: **`203`** = input-validator error, **`204`** = database error, **`205`** =
  external error; the `M###` / `Q###` segment is the resolver's id from
  `server/graphql/resolver-id-hash-<audience>.js`.
- Throw cases capture the un-awaited call as an arrow —
  `const actual = () => target.method(params)` — then
  `await expect(actual).rejects.toThrow(expected)`. Never `try`/`catch` in a test body.
- A resolver's **placement still follows §1 per-method**: a mutation's `resolve()` goes to
  `_orders/<Domain>/`; its `schema` / `errorCodeHash` getters, `validateInput`, `formatResponse`
  and any `find~` go to the sibling `__tests__` file mirroring the resolver's source path.

## Finishing checklist

- [ ] In `tests/__tests__/` (mirroring the source path) if the method does no DB write; in
      `tests/_orders/<Category>/` if it writes directly or transitively.
- [ ] The `_orders` file is **imported by its category's `_.test.js`**, in the right position.
- [ ] Explicit ids come from the feature's prefix in `.hora/id-bank.json`.
- [ ] No new logic in the test; every function/class/mock it uses is already tested.
- [ ] Any shared mock/tool lives under `tests/mocks/` or `tests/tools/` and has its **own** test.
- [ ] Verified with a single-file run, then with `npm_config_script_shell=bash npm test`.

## What the skill does not settle

- **Nothing about resolver-specific assertions** (error-code strings, the `203`/`204`/`205`
  prefixes) — see §8; that comes wholly from the always-on rules.
- **Nothing about test-authoring style** — describe titles and member notation, AAA phases, allowed
  `cases` fields, one-input title interpolation, the allowed matcher set, `constructorSpy`,
  inheritance describes. All of that is `D:/ORT/rules/testing.md`.
- **No naming convention for a `_orders` category folder** beyond "a cohesive area of write
  behavior"; this repository names each folder after the **model** under test.
- **No rule on whether a `_orders` test must clean up after itself.** It does not here — the
  rebuild before each run is the reset, and fixed ids depend on that.
- **No guidance on where a test for a `dev-master` / `master` seeder goes.** This repository puts
  it in `tests/__tests__/sequelize/seeders/<set>/<table>.js` (the seeder does not write during the
  test; it already ran).
