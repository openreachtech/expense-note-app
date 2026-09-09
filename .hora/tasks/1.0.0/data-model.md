# #data-model  The nine tables, and the four seeded categories
<!-- spec: data-model @ sha256:621aa4f9caa62742 -->
<!-- repositories: backend -->

Scope: **nine tables, and all nine are this checkpoint's to write.** Three are the domain's
       (§9.1 `staff_members`, §9.2 `expense_categories`, §9.3 `expenses`) and six are the
       credential cluster (§9.4 `staff_member_secrets`, §9.5 `staff_member_password_hashes`,
       §9.6 `staff_member_access_tokens`, §9.7 `staff_member_refresh_tokens`, §9.8
       `staff_member_secrets_bk`, §9.9 `staff_member_password_hashes_bk`).

       **Nothing is inherited.** The backend row ships the session orchestration —
       `SessionClerk`, the cookie clerk, the engine cookie config — and `sequelize/models/`
       ships EMPTY. `SessionClerk` taking its models injected reads as though its tables
       exist; they do not (`.hora/tree/expense-note-backend.md`).

Constraint: **§9.4 and §9.5 are stricter than the running implementations, deliberately.**
            §9.4's foreign key is a UNIQUE index and its `email` is UNIQUE. The reference
            projects use a plain FK index and no index on `email` at all. Both are decisions
            written into §9 with their reasons — do not reconcile them toward the other
            projects, and do not copy the unique constraints into §9.8, where a plain 1:N
            index is what lets a history table hold more than one change (Q8)

Constraint: **the equipped `hor-cookie-authentication` 0.1.0 is known incomplete here.** Its
            `migrations.md` omits the two secrets tables entirely. **The spec is the authority;
            a disagreement with the skill must not rewrite §9** (Q8,
            `../../questions/1.0.0/open.md`). Fixed upstream, unpublished

Constraint: approval by a manager is out of scope **for now** (1.1.0, spec §4). `expenses`
            carries `status` from this first migration, holding `recorded` and nothing else.
            Leave that column in place — it is the seam, not dead weight

Constraint: a receipt photograph is out of scope **for now** (spec §4). An attachment arrives
            later as a **child table**. Add no column to `expenses` for it, and assume nowhere
            that an expense has exactly one representation

Constraint: categories an operator can edit are out of scope **for now** (spec §4). A category
            is already a row with an id, seeded from here. Never an enum written into the code

Constraint: signing up and resetting a password by mail are out of scope **for now** (spec §4).
            The password is a one-way hash on the member of staff's own row, so a later reset
            changes that row and touches nothing else

Constraint: more than one currency is **permanently** out of scope (spec §4). `amount` is an
            integer number of yen. Design no currency column, no exchange rate and no minor
            unit. Do not abstract money

Constraint: more than one company in one deployment is **permanently** out of scope (spec §4).
            No row carries a tenant. Do not add one "just in case"

Note: the index on `(staff_member_id, spent_on)` is spec §9.3's own, and it serves the
      heaviest read this version has (§7). It is a requirement, not an optimization to defer

Constraint: **where an equipped skill and an always-on ORT rule disagree, the rule wins** (Q10).
            Three cases reach this checkpoint: indexes are added with **sequential `await`s,
            never `Promise.all`**; index names are built from an **abbreviated
            `SHORT_COLUMN_NAME`**, always, not from the full column name; and a subclass's
            JSDoc uses **`@augments`**, not `@extends`. The migration skill's own examples show
            the opposite of the first two — do not follow them, and do not "fix" these back

Note: `saved_at` on §9.4, §9.5, §9.8 and §9.9 sits BESIDE `created_at` / `updated_at`, never
      instead of them — `created_at` is when the row appeared, `saved_at` is when that address
      or digest was set. The convention's column lists name auth-specific columns only, which
      is why `id` never appears in them either

Note: the foreign-key attribute is PascalCase over a snake_case field — `StaffMemberId` →
      `staff_member_id`. That is this project's foreign-key rule, and the reference migrations
      carry a comment saying so

Note: **the automated suites run on SQLite, not on the MariaDB the spec declares.**
      `sequelize/config.cjs` is the boilerplate's own: `development` is SQLite at
      `sequelize/storage/development.sqlite3`, and the `DATABASE_*` values filled into
      `.env.development` are read only by its `production` block. That matches the house
      convention — SQLite for dev and test, and the `mariadb:10.5.12` container for the manual
      verification spec §8 declares — so it is not a defect. The consequence is real, though:
      §9's types are MariaDB-shaped (`datetime(3)`, `varchar(191)`, `bigint` keys), and SQLite
      is lax about every one of them. Verify the migrations against the container before
      checkpoint 3 is called passed, rather than against SQLite alone

Note: this feature has no `<!-- usecases -->` block, and that is correct — a data model has no
      user-facing use case of its own, and the features built on it carry them. Its
      `<!-- acceptance -->` block is what checkpoint 18 reads

## Spec gate
- [x] 1. Draft or confirm the specification  <!-- skills: hoc-requirement-definition; digests: none taken — an interactive checkpoint, run by the main session, hands no agent a digest. Matched against hora-skills-ort-core 0.2.0 -->
- [x] 2. Verify the use cases can be met  <!-- skills: hoc-requirement-definition; digests: none taken — interactive, no agent. Matched against hora-skills-ort-core 0.2.0. GAP: the checkpoint delegates to "the shared UI/UX project context", and the only equipped skill covering that is frontend-surface (hof-), out of surface for a backend-only feature. Ran without it -->

## Backend gate
- [x] 3. DB and API schemas  <!-- skills: hor-database-design, hor-sequelize-migration, hor-sequelize-model, hor-type-interface, hor-constant-definition, hor-cookie-authentication, hoc-naming, hoc-jsdoc; digests: hora-skills-ort-renchan 0.1.0 and hora-skills-ort-core 0.2.0 -->
- [x] 4. Stub API  <!-- n/a: this feature adds no API operation at all. §9 declares nine tables and zero operations — verified, no operation table and no schema/input/result header anywhere in the section. §10.1, §11.1 and §12.1 hold this version's ten operations and belong to #sign-in, #expense-entry and #monthly-summary, each of which will stub its own -->
- [x] 5. The modules the implementation needs  <!-- skills: hor-sequelize-seeder; digests: hora-skills-ort-renchan 0.1.0. Catalog checked first, once, for the whole feature -->
- [x] 6. Actual API  <!-- n/a: this feature adds no API operation, the same reason as checkpoint 4. Its half of the exit condition that COULD apply — the unit tests covering this feature's acceptance criteria — was satisfied anyway: tests/__tests__/sequelize/ is 4 suites / 32 tests green and tests/_orders/ is 2 suites / 21 tests green, covering four of §9's nine criteria -->
- [x] 7. Worker  <!-- n/a: decided with the placement skill, not by eye. Everything this feature contributes is light, synchronous and in the request path or outside it entirely -->
- [x] 8. Security audit  <!-- skills: hor-security-audit, invoked IN FULL rather than through a digest, both passes. Two passes: the first over all 42 files, the second scoped to the fix -->
- [ ] 9. Verify the use cases again, against the built API

## Frontend gate
- [ ] 10. Open the frontend
- [ ] 11. Reconfirm UI/UX and the use cases
- [ ] 12. Component design
- [ ] 13. The frontend modules the implementation needs
- [ ] 14. API client
- [ ] 15. UI
- [ ] 16. Wire the data-fetching logic in
- [ ] 17. Local test environment

## Acceptance gate
- [ ] 18. Acceptance (E2E and unit both)

## Checkpoint 2 — the walk

`#data-model` states no use cases of its own, and that is correct: a data model has none, and
the features built on it carry them (`../../../.claude/skills/hora/references/spec-format.md`).
**Passing on that alone would have been vacuous**, so what was walked instead is the one thing
this feature can still get wrong — whether §9's nine tables can represent every state the
seven dependent use cases need.

| Use case | What it needs of §9 | Held by |
|---|---|---|
| signs in Monday morning, and every screen after knows who they are | a name to show, an address to look up by, a digest to verify, a token pair to issue | §9.1, §9.4, §9.5, §9.6, §9.7 |
| signs out on a shared machine; the next person is asked to sign in | a way to revoke a whole series, not one token | §9.7's `revoked_at` + `session_key`; §9.6 deleted |
| records a 1,200 yen fare with date, amount, category, memo | all five fields, and an owner | §9.3 |
| corrects 12,000 to 1,200, and sees the corrected one from then on | update in place, with no history expected | §9.3. **No `_bk` for expenses, and none wanted** — §11's criterion says the entry count must not change, and §7 keeps what remains rather than what was |
| removes a duplicate lunch, and it is gone from every later read | a hard delete | §9.3. §7 deletes outright rather than archiving, so no `deleted_at` |
| picks a month, reads the total, writes it on a claim form | a month's rows for one owner, summable | §9.3, on the `(staff_member_id, spent_on)` index §7 calls the heaviest read |
| switches to the previous month and reads its entries | the same read, by a different month | as above |

**All seven are representable, and nothing in §9 is unused by them** except the two `_bk` tables
and `status`, each of which §9 already states is there for a later version rather than this one.

**One thing the walk confirmed rather than assumed:** `expenses` has no backup table, and must
not gain one. Two of the three expense use cases turn on a correction or a removal being
final — §11's criterion that the entry count does not change, and §7's rule that what is
retained is what remains. Adding a history table for expenses out of symmetry with §9.8 and
§9.9 would contradict both.

## Checkpoint 3 — what was built, and what verified it

Nine tables, nine migrations, nine models, nine type declarations under `types/models/`, the
expense-status constant pair, the shared `BaseAppRenchanModel`, and seven test files. Seven
implementer agents, one per table except the two backup pairs, which stayed whole because the
`_bk` half must declare `tableName` explicitly and the body half must keep
`super.setupHooks?.()` — both failures are silent, and splitting the pair puts the link between
them in two prompts.

**No API surface.** §9 declares no operations, so that half of the exit condition is not
applicable here; §10.1's operations are `#sign-in`'s.

**Verified against real MariaDB 10.5.12, not SQLite.** Re-run in this session rather than taken
from another session's report, because the evidence had been cleaned up and could not be
reproduced from the tree:

- all nine migrated clean, then `db:migrate:undo:all` reverted all ten and left zero tables
- types are what §9 declares: `bigint(20)` keys, `int(11)` on `expense_categories`,
  `datetime(3)` throughout, `varchar(191)` for name / email / memo / the digests, `date` for
  `spent_on`, `varchar(32)` for `status`, `used_at` nullable and `expired_at` not
- **Q8's two strictnesses hold**: `staff_member_secrets` carries UNIQUE on `staff_member_id`
  AND UNIQUE on `email`, while both `_bk` tables carry a plain FK index — which is what lets a
  history table hold more than one row per person
- **Q10's amended naming came out as intended**: composites abbreviated (`smi`, `eci`,
  `smi_so`, `at`, `sk`, `th`), single words left whole (`name`, `email`)
- `information_schema.key_column_usage` reports **zero** DB-level foreign-key constraints

**Why that check mattered.** The suites run on SQLite, which reads a `bigint` key back as
`INTEGER` and every `datetime(3)` as bare `DATETIME`. Every one of the types above would have
passed locally whatever the migration said.

**One thing about this machine, not about the code.** Port 3306 is held by a MariaDB inside
WSL, relayed by `wslrelay.exe`, so the committed compose file cannot bind it. The verification
ran on 3307 through an override kept outside the repository, with a throwaway config that is
the repo's own `live` block with the port changed. **The committed 3306 is correct and was not
touched** — CI publishes 3306, and the repository has to match CI.

**Tests exist but were not run, and that is per the checkpoint.** Only checkpoints 6, 16 and 18
name tests in their exit conditions. Units 3 and 5 wrote seven test files covering four of
§9's criteria; they first run at checkpoint 6.

## Checkpoint 5 — one module, and what the catalog check found

**The catalog check ran first, once for the feature, as the rule requires.**
`@openreachtech/hora-ecosystem` tracks 33 packages; the only candidate touching this work is
`renchan-sequelize`, already a dependency. **Nothing was installed and nothing was
reinvented** — the whole feature runs on catalog packages already present:
`TimestampSeedsSupplier` and `MigrationAttributeFactory` for the migrations and the seeder,
`ModelAttributeFactory` and `RenchanModel` for the models, `SequelizeActivator` for the
bootstrap.

**One module: the category seeder**, which §9's criterion "the four categories exist after the
seeder runs, in their display order" requires. Written as **two files**, and that is the part
worth knowing:

`package.json`'s `db:seed:master` points at `sequelize/seeders/**dev-master**`, and **nothing in
this repository loads `sequelize/seeders/master/` at all** — there is no `db:seed:prod`. So the
canonical file sits in `master/` and a re-export under the identical filename sits in
`dev-master/`, which is what dev and CI actually read. Placed in `master/` alone the seeder
would never run, and §9's criterion would fail at acceptance with a correct seeder sitting in
plain sight.

**Row ids use the allocated `100` prefix** — `10000001` to `10000004` — and the seeder skill's
**master-data exemption was deliberately not taken.** That exemption would have used small
sequential ids or ids from `app/constants`; `/hora-build`'s rule says a seeder's explicit ids
come from the allocated prefix "in any table" with no exemption, and §9.2 forbids these four
categories ever becoming a code enum, so nothing will reference them by literal id. The four
strings now live in exactly one place in the repository.

**Verified against real MariaDB, then reverted.** The four rows read back in `display_order`
1–4 with ids `10000001`–`10000004`; `AUTO_INCREMENT` advanced to `10000005`, so `int(11)`
holds the prefix comfortably. `db:seed:undo:all` removed exactly those four. The database was
left as found.

**`PasswordEncipher` is not this feature's module.** Unit 5 of checkpoint 3 asked checkpoint 5
for it, but §9's criteria never mention password verification and §10's do — the model takes
it injected and its tests stub it, so `#data-model` needs nothing. It belongs to `#sign-in`'s
checkpoint 5.

### Two environment facts this checkpoint established

**The npm database scripts cannot run on Windows as written.** `db:refresh`, `dev`, `test` and
`test:live` all begin with `export`, a POSIX shell builtin, and npm on Windows spawns
`cmd.exe`: `'export' is not recognized as an internal or external command`. The local SQLite
database was initialized by running the underlying `sequelize-cli` commands through bash
instead. **Same family as Q13** — the row was created on Windows and its scripts assume a
POSIX shell (Q15).

**The local test picture, run after initializing the database:** `tests/_orders/` is 2 suites
and 21 tests, all passing. `tests/__tests__/` is 18 suites and 171 tests with **6 failing**,
and all six are the known `AUTH_COOKIE_SECURE` failures — this branch still carries `=false`,
and the fix is backend PR #5, unmerged. **Nothing this checkpoint wrote is implicated**, and
the four suites covering this feature's own models and seeder are 32 tests, all green.

## Checkpoint 7 — the placement decision, made with the skill

The checkpoint says to decide this **with the placement skill, not by eye**, so its decision
flow was walked over everything this feature contributes:

| What this feature contributes | Where the flow puts it |
|---|---|
| the nine migrations | **not request processing at all** — invoked by `db:migrate` from the CLI |
| the category seeder | the same — invoked by `db:seed` |
| the nine models | passive definitions; they have no placement of their own |
| `Expense`'s `beforeSave` existence check | **request path.** Two indexed primary-key lookups, and it has to be synchronous because it gates the write |
| the backup mixin's `afterSave` on the two credential tables | **request path.** One insert, synchronous, because the history is the point |

Nothing here is heavy, time-consuming or dependent on anything outside the process, so step 3
of the flow never fires; there is no side effect to defer past a response, and nothing
time-triggered. **The spec says the same independently:** §8 declares no Redis "because this
version runs no background job — every write finishes inside its own request, and nothing here
leaves the process."

So the checkpoint is not applicable, and the reason is a decision rather than an absence of
one.

## Checkpoint 8 — two audit passes, and what each found

**The audit skill was invoked in full both times, never through a digest** — a step whose skill
*is* the criteria runs the skill whole, because the missing check is the one nobody thinks to
ask about.

### First pass: 0 HIGH, 0 MEDIUM, 3 LOW, 2 INFO — verdict `met`

**All three LOW findings were fixed anyway, and the reason is the same in each case: the §9
criterion they bear on is worded absolutely, and the code did not deliver that absolute.**

| Finding | The word that forced it |
|---|---|
| `Expense`'s referential hook missed `bulkCreate` and static `update` | §9: an expense **always** names an owner and category that exist — and this hook is the entire control, since the project declares no DB foreign key |
| `email` uniqueness depended on a collation nobody pinned | §9: two members of staff can share a current address **in no circumstance** — and on SQLite, the dialect the suites run on, `UNIQUE` on `TEXT` is case-sensitive, so it was **false where it is tested** |
| `hashToken` digested an empty string | §9.7: **no** refresh token is stored in a form that could be presented as one |

**The second was fixed by normalizing rather than by pinning a collation**, deliberately:
pinning corrects MariaDB and leaves the tests running on a dialect where the criterion still
fails. Normalization holds underneath any dialect. Its stated limit: it holds for writes
through the model, so raw SQL or `queryInterface` would bypass it — the durable answer is a
collation **plus** normalization, not either alone.

### Second pass, scoped to the fix: all three resolved, one new LOW, verdict `met`

The re-audit confirmed each fix against the installed Sequelize with line numbers rather than
from memory — including that `options.attributes` **is** the caller's values object by
reference (`lib/model.js:1939-1943`), which is what makes the in-place rewrite work.

**The new LOW: `upsert()` runs `beforeUpsert` alone**, so it reaches both tables with no hook
and would bypass both guarantees. Zero callers in tracked source, and no operation exists to
add one.

**Decided: not fixed, but the docstrings that overclaimed were corrected.** Adding
`beforeUpsert` hooks would state the same rule in a third place for a path nothing can reach.
What was actually wrong is that both JSDoc blocks claimed exhaustiveness — "on every write
path", "a value reaches this table three ways" — and that claim was wider than the code. Both
now name `upsert` as unsupported and say what adding a caller would require first. **The same
class of sentence this spec has been bitten by five times: true when written, and nothing
points back at it when the design moves.**

**The backup mixin's `afterSave` has the identical bulk-path gap**, and it is accepted on the
same grounds: no operation in 1.0.0 reaches those paths, and 1.0.0 has no address-change
operation at all. Fixing it would mean reimplementing framework mixin behavior in our model.

### A pre-existing flake the fix had to clear first

`tests/_orders/` was **already failing 4 of 8 parallel runs before any of this work**, and the
cause was auto-increment fixture ids landing on explicit `100006xx` ids inserted by a parallel
worker. Nothing in `tests/_orders/` now relies on auto-increment for `staff_members` or
`expense_categories`, and 8 consecutive runs pass. **It would have surfaced as an intermittent
CI failure on the feature's own pull request and been read as a defect in the data model.**

### Verified independently before marking this passed

Lint clean across **62 files**, and **103 tests in 9 suites** green against a freshly migrated
and seeded database — re-run rather than assumed, because the lint fix below changed real
control flow.
