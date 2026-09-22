# #expense-entry  Recording, correcting and removing an expense, and reading back one's own
<!-- spec: expense-entry @ sha256:71e7ee95aad53838 -->
<!-- repositories: backend, frontend-staff -->

Conflict: appends to the staff audience's SDL and to its GraphQL type declarations under
          `types/`. Two other features carry the same mark. The audience itself, its engine and
          its `server/index.js` entry are #sign-in's — re-read the real files, do not recreate
          them

Constraint: approval by a manager is out of scope **for now** (1.1.0, spec §4). **Every read
            goes through `status` rather than assuming a recorded expense is a final one.** The
            contract exposes the field from this version (Q4), holding `recorded` only

Constraint: exporting a month for a claim form is out of scope **for now** (1.2.0, spec §4).
            Nothing here re-derives a month or a total — that is #monthly-summary's one
            operation, which the later export calls

Constraint: more than one currency is **permanently** out of scope (spec §4). `amount` is an
            integer number of yen, typed `Int!` in the contract (Q2). Build no currency
            handling and no minor-unit handling

Note: **ownership is answered as not found, never as forbidden** (spec §7, and this feature's
      acceptance criteria). Reading, correcting or removing somebody else's expense returns
      not-found, and the answer says nothing about whether the row exists. That wording is a
      requirement, not a style choice — checkpoint 8 reads it too

Note: every mutation returns only the identifier of what it wrote (spec §11.1), and the screen
      re-reads through `expenses`. That is CQRS as the architecture rule states it — do not
      have a mutation return the row it wrote

Note: the value checks are the spec's own, and each is an acceptance criterion: no amount, a
      zero amount and a negative amount are all refused; a date after today is refused; the
      memo is genuinely optional and reads back empty rather than failing; removing an
      already-removed entry changes nothing

Note: `spentOn` crosses the contract as an ISO `YYYY-MM-DD` string (Q3), not a datetime. It is
      the day the money was paid, not the day it was recorded

## Spec gate
- [x] 1. Draft or confirm the specification  <!-- interactive, main session; no agent, so no digest. Section 11 read against sections 4, 7, 9.3 and the pinned contract. Two gaps found, each changing behaviour a member of staff can see, both answered by the user: "most recent first" now means `spent_on` (section 9.3's index already pointed there), and a second removal is answered as NOT FOUND, collapsing into the ownership rule section 11 already states. Raised as PR #20 against `specs/` and MERGED at a9cb3dc, both hunks verified on the release tip rather than inferred from the merge. The two decisions are this session's user's answers; the merge was approved by the peer session's user, shown both hunks verbatim -- recorded separately because they are not the same person. A `sort` clause and a same-date tie-break were deliberately NOT proposed -->
- [x] 2. Verify the use cases can be met  <!-- interactive, main session. All three of section 11's use cases walked against the operations section 11.1 declares; all three met, and no operation is missing -- "opens that entry" is served by `expenses`, which already returns the entry's current values, so no read-one operation is needed. Every acceptance criterion is reachable from the declared operations. One trap recorded for checkpoint 6: `correctExpense` is a FULL REPLACE, so a correction that omits the memo clears it -->


## Checkpoint 1 — the specification confirmed, and two gaps that changed behaviour

**§11 was coherent with §4, §7, §9.3 and the pinned contract.** Two gaps were found, and neither was
an implementation detail: each changed what a member of staff can see, so each went to the user.

### Gap 1 — "most recent first" never said by which date

§11.2's screen line said the entries are shown "most recent first" and stopped there. **`spent_on`
and `created_at` genuinely differ** — back-date an expense and it lands somewhere else in the list —
so the sentence did not decide what the screen shows.

**Resolved to `spent_on`, newest first**, by the user. It is also what the rest of the spec already
leaned toward, which is why it was the recommended reading rather than a coin toss:

- §9.3 indexes `(staff_member_id, spent_on)` and calls it **"the one read that matters"**
- §6 defines a month as "the calendar month an expense's **date** falls in"
- `#monthly-summary` groups by the same field

**The cost is recorded rather than hidden:** an expense paid last week and recorded today appears
below this week's rather than at the top. That is a consequence of the choice, not a defect.

### Gap 2 — "removing it a second time changes nothing" had two observable readings

Succeed idempotently, or answer not found. **A caller can tell them apart**, so the sentence did not
decide the behaviour.

**Resolved to not found**, by the user — which collapses it into a rule §11 already states. Another
member of staff's expense is "answered as not found, and the answer says nothing about whether it
exists", and §7 deletes a removed entry **outright rather than archiving it**. So a removed row and
somebody else's row are the same thing to a reader, and now get the same answer. **One rule instead
of two**, and one fewer way for the non-disclosure property to be broken by accident later.

### Two things deliberately not changed

- **No clause about `sort`.** It was going to be raised — `ExpensesInput` carries `pagination.sort`
  and §11 never says whether a caller may set it. **The pinned contract already answers it in
  writing:** "No operation in 1.0.0 lets the caller choose a sort, so this pair is declared for the
  convention's pagination shape and left unused." A sentence restating the contract would create a
  second place for the two to disagree.
- **No tie-break for two expenses sharing a date.** Plain `spentOn` ordering is what was chosen, and
  **inventing text nobody decided is how a spec acquires clauses with no owner.** Recorded here as a
  known detail instead.

### How this reached `specs/`

**As a proposal, not a write.** The exact words were put in front of the user and then raised as
**pull request #20** against `release/1.0.0`, touching `specs/1.0.0/spec.md` and nothing else. The
decisions themselves are the user's answers and were already settled, which is why the feature was
never blocked on the paperwork.

**It merged at `a9cb3dc`, and both hunks were then read off the release tip rather than inferred from
the merge succeeding** — §11.2's ordering sentence and §11's double-removal criterion are present in
`specs/1.0.0/spec.md` on `release/1.0.0`.

**Who approved which half is worth separating, because they are not the same person.** *What the
clarifications say* is this session's user's: they answered both questions directly, and those
answers are what the diff was built from. *The merge itself* was approved by the peer session's user,
who was shown both hunks verbatim and unabridged and answered against those words. Invariant 1 was
satisfied in substance on both counts — a human saw the exact text before it landed — but the record
should not read as though one person did both, and a later reader tracing the ordering decision
should look to this feature's checkpoint 1, not to the merge.

## Checkpoint 2 — the use cases can be met, walked against the operations §11.1 declares

| §11 use case | The path | Verdict |
|---|---|---|
| records a 1,200 yen fare that evening — date, amount, category, memo — **and sees it in their entries** | `expenseCategories` fills the category field, `recordExpense` writes it, `expenses` shows it. With `spent_on` ordering a fare paid today is at the top | **met** |
| typed 12,000 instead of 1,200, **opens that entry**, corrects the amount, and sees the corrected one from then on | "opens that entry" needs its current values, and `expenses` already returns them — `id`, `spentOn`, `amount`, `memo`, `expenseCategory`. **No read-one operation is needed**, and none is declared | **met** |
| recorded the same lunch twice, removes the duplicate, entries no longer show it | `removeExpense`, then `expenses`. §7 deletes outright, so it is gone from every later read | **met** |

**Every acceptance criterion is reachable from the declared operations**, and none needs an operation
§11.1 does not list: the value refusals and the future-date refusal are the validator's, the optional
memo is `String` rather than `String!` in the contract, correcting in place is `correctExpense`
returning only an id, ownership is §7's not-found rule, and the no-session refusal is §7's
Authentication row.

### One trap worth carrying to checkpoint 6, found while walking use case 2

**`correctExpense` is a full replace, not a patch.** Its input takes `spentOn`, `amount` and
`expenseCategoryId` as non-null and `memo` as nullable — so **a correction that omits the memo clears
it**. Correcting only the amount means resending everything else unchanged, and the screen must
pre-fill from `expenses` rather than send a sparse object.

That is not a spec hole — §11.1 declares exactly that input — but it is the kind of thing a screen
gets wrong once and a member of staff discovers by losing a memo.

## Backend gate
- [x] 3. DB and API schemas  <!-- skills: hor-graphql-schema, hor-type-interface, hor-database-design, hor-sequelize-model, hor-sequelize-migration, hoc-naming, hoc-jsdoc; digests: hora-skills-ort-renchan 0.1.0 and hora-skills-ort-core 0.2.0 -- ALL REUSED from #sign-in, none taken here, the installed versions being unchanged. NO MIGRATION, deliberately: section 11 operates on section 9.3's `expenses`, which #data-model shipped complete including the status seam and the composite (staff_member_id, spent_on) index. Verified against sections 9.2 and 9.3 rather than rebuilt. The SDL match with the pinned contract was established by AST comparison, not by eye: 11 types and 5 operations, zero DIFF, zero EXTRA. Q48 raised -- the contract exposes createdAt/updatedAt, which hor-graphql-schema forbids by name; the contract wins and the question goes to the user -->
- [x] 4. Stub API  <!-- REACHED IN PART. skills: hor-stub-api; digest reused from #sign-in at hora-skills-ort-renchan 0.1.0, none taken here. Five schema-accurate stubs, same class names and interfaces as the real resolvers will have. "Callable from outside" is evidenced IN PROCESS through the framework's own schema-and-resolver path with contextValue: null, NOT over a socket -- Q24, and the record separates the two claims deliberately. Q28's statement is in all five docblocks and is sharper here than at #sign-in, because every operation this feature adds is SUPPOSED to require a session. Q33 fired a second time via the shared SQLite file, as checkpoint 3 predicted -->
- [x] 5. The modules the implementation needs  <!-- catalog check FIRST and once for the whole checkpoint, then three implementer units. 33 tracked packages searched: adopted renchan-sequelize's PaginationMixinModel, mentsu-deep-value-converter and mentsu-value-inspector; DECLINED mentsu-field-path-value-extractor (the only nested read is two deep, which the style rule permits) and jest-deep-containing / jest-expect-each (nothing needs them) -- declines recorded because "not used" and "not considered" look identical later. Modules: the expenses development seeder (14 rows, block 102), CalendarDateInspector behind CALENDAR.TIMEZONE, and PaginationMixinModel on Expense. Every import CONFIRMED TO RESOLVE by the main session rather than taken from the units' reports. MY BRIEF WAS WRONG about Expense.findAllWithPagination -- it is Expense.$.findAllWithPagination, and the unit caught it. Q38 found already broken in expenses by #data-model's own test, amended with the measurement. Q49 behind one constant, labelled recommended-not-decided. I caused Q33's third instance by running three units in parallel on one SQLite file -->
- [x] 6. Actual API  <!-- five implementer units, ONE AT A TIME rather than in parallel, because five units on one SQLite file is how I caused Q33's third instance at checkpoint 5. All five operations served from the actual/ pool; the five stubs stay for the frontend until checkpoint 16. 68 suites, 1654 tests, lint clean, every figure re-run serially by the main session. The not-found rule is the ABSENCE of a second code, so errorCodeHash is pinned whole; mutation-tested by dropping StaffMemberId from the where clause. "Nothing is recorded" proved through the product's own read path and mutation-checked. A declared-but-unexercised session guard found in all three mutations by noticing the query resolver covered its equivalent -- 16 cases added, and the exercise DISPROVED a docblock about what the guard prevents. Q46's mechanism ran backwards five times exactly as checkpoint 4 predicted. Q50 capped at 100 (chosen, not confirmed), Q51 written into tests/_orders/README.md and all eight barrels. Two of my briefs were wrong and units caught both -->
- [x] 7. Worker  <!-- NOT APPLICABLE, established with hor-execution-placement-pattern per operation rather than by eye, which is what this checkpoint's own clause demands. All five operations short-circuit at the flow's read-only or light-write step; the trigger question is never reached. Spec section 8's footnote says it outright -- Redis is not declared because this version runs no background job -- and section 7's retention line forecloses the one candidate a hard delete would suggest. Nothing written, and nothing invented to make the checkpoint non-empty. Two findings anyway: the skill had NO DIGEST and my brief did not ask for one (taken afterwards, pinned to hora-skills-ort-renchan 0.1.0 -- my process gap), and ioredis is a declared dependency with zero importers, so package.json alone would suggest a job facility that does not exist -->
- [x] 8. Security audit  <!-- hor-security-audit, read-only, over this feature's change set read from the WORKING TREE (a commit range would be empty before the gate) plus the five declared operations. 0 HIGH, 1 MEDIUM, 1 LOW, 4 INFO; all fixed or accepted-and-recorded, none left silent. The non-disclosure property holds by the SHAPE of the contract -- no read narrows by id alone and no input type declares an owner field -- and the timing channel is closed because the validators do zero DB access. MEDIUM: this feature opened an alias-amplification surface (~250 aliased expenses calls in one 16kb body, ~500 DB round trips); limits measured from all 24 real documents, introspection exempted at depth 15, fragments expanded and cycles guarded. MY BRIEF WAS WRONG a third time -- an engine cannot reach validationRules at all. LOW: a stub-only operation would get NO auth filter; guard derived from the real loader for all three audiences. Q53, Q54, Q55 raised for what was accepted -->
- [x] 9. Verify the use cases again, against the built API  <!-- main session, in conversation. All three of section 11's use cases walked as REAL CALLS against the built schema with a real access token on a real header, shapes printed at every step; nothing fell short, nothing sent back to checkpoint 3, no field added on the way past. Driven through jest because server/index.js still cannot be imported here (Q24) -- the socket is unreached rather than unreachable, per Q24's WSL amendment. The three not-found situations answered IDENTICALLY as a caller experiences them. My own first walkthrough had a gap: I read page one for the optional memo and the row was oldest, so I had shown the write succeeded rather than that the memo reads back empty -- a second call at offset 2 returned memo: null. Walkthrough deleted; tree clean -->


## Checkpoint 3 — the API schema, and a DB half that was already built

**The DB half needed nothing, and that is a finding rather than an omission.** §11 operates on §9.3's
`expenses`, which `#data-model` shipped complete — every column including the `status` approval seam,
the composite `(staff_member_id, spent_on)` index the ordering decision depends on, both models with
their associations, a `verifyExpenseCategory` hook on all three write paths, and the four categories
seeded in `master/` and `dev-master/`. All of it was read against §9.2 and §9.3 and found sufficient.

**Nothing was added because nothing was missing.** Recorded explicitly because **"no migration" and
"migration forgotten" look identical in a diff**, and this is the first feature where that
distinction arises. **This is the digest-and-derivation system paying off in the other direction
too:** no new digest was taken at this checkpoint either — the seven `#sign-in` took are still at the
installed package versions.

### The contract match was mechanical, not visual

A throwaway script parsed the pinned contract and the concatenation of all three SDL files with
graphql's own `parse()`, then compared `print()` of each definition's AST node by name — Query and
Mutation field by field, because those merge across files.

**Eleven types and five operations OK, zero DIFF, zero EXTRA.** The only `MISS` lines were
`monthlyExpenses` and its two types: `#monthly-summary`'s `004-` file, correctly absent.

The tests assert the **built, merged** schema through the framework's own `GraphqlSchemaBuilder`, not
the file's text — so a duplicate declaration of `Pagination` or `DateTime` fails them too — and each
case asserts the canonical `print()` of an AST node, which means **an added field fails as loudly as
a missing one**. Falsified rather than assumed: changing `memo: String` to `String!` turns exactly 3
of 16 red, which is §11's "the memo is genuinely optional" criterion.

### A new arbitration shape: skill versus *contract*

Q48 records it. `hor-graphql-schema` §5.1 forbids `createdAt` / `updatedAt` by name; the pinned
contract declares both on `Expense`. **The unit wrote what the contract says and reported the
divergence rather than renaming anything**, which is right — checkpoint 14's rule is that the
contract is authoritative and wanting it changed is a question, not an edit.

**This is a fifth category for the arbitration taxonomy**, and it behaves like the first: there *is*
a standing authority (the contract), so the resolution is mechanical once noticed, and the entire
cost is detection. What differs is only which authority decides — Q10's rule for skill-versus-rule,
the contract for skill-versus-contract.

**It is worth asking now rather than later** because `#monthly-summary` reuses this exact `Expense`
type, by the contract's own design, so the cost of changing it rises with every checkpoint that
lands. And the hazard is sharp in this feature specifically: `spentOn` is the day the money was paid
and `createdAt` the day the entry was typed — §11's whole ordering question turned on those being
different, and a field named `createdAt` beside `spentOn` invites exactly the confusion §11.2 had to
be clarified to avoid.

### Two traps handed forward rather than discovered later

- **Checkpoint 6 will need `createdAt` / `updatedAt` off the entity**, and `types/models/Expense.d.ts`
  deliberately declares neither — they are framework-managed and absent from the model's attributes,
  by that file's own comment. So the resolver owes either a cast or an addition to a `#data-model`
  file. Named now so it is not met mid-resolver.
- **`tests/__tests__/sequelize/seeders/master/expense_categories.js` asserts the *entire*
  `expense_categories` row set with `toEqual`.** The unit's first full run failed it — not from its
  own work, but because ~130 rows named `expense category 100003xx` were left in the SQLite file by
  an earlier `_orders` run. **Any test in checkpoints 5 or 6 that creates a category row and does not
  delete it turns a different feature's test red.** This is Q33's order-dependence finding, now with
  a second way to trigger it.

**43 suites, 736 tests, lint clean.** Baseline was 720.


## Checkpoint 4 — five stubs, and the exit condition's second half again

**Reached in part, and the two halves are different claims worth separating.**

| Clause | Verdict |
|---|---|
| a schema-accurate stub exists for every operation this feature adds, returning hardcoded data | **met** — five of five |
| **callable from outside** | **evidenced in process, not over a socket** |

**That distinction is the record's, not a hedge.** "Verified in-process because the server cannot
start" and "verified as the exit condition literally asks" are different claims, and only the second
is what checkpoint 4 requires. Q24 makes the second impossible here — `server/index.js` still dies at
`ERR_UNSUPPORTED_ESM_URL_SCHEME` — so the express app, the middleware chain, the 16 kB body limit and
the CORS allow-list are untouched by this evidence and stay so until Q24 closes. That limit is
written into the test file's own docblock rather than only into this record.

What *was* done: real GraphQL documents executed against the audience's executable schema, built by
the same `GraphqlSchemaBuilder` the running server uses, resolvers loaded from the same two pools.
**`contextValue: null` in every case** — not even an object a session could be read off — and all
five answer anyway, which **demonstrates** the Q28 property instead of asserting it in prose.

### Q28 is sharper here than it was at `#sign-in`

Each of the five docblocks states that while the operation is served from the stub pool it runs with
**no authentication filter at all**, and names the mechanism: `GraphqlResolversBuilder` builds the
filter hash from `actualResolverSchemaHash` only, so a stub-only field gets `filter === undefined`.
The unit verified that against the installed builder rather than taking the digest's word.

**Why it is worse here.** Most of `#sign-in`'s stub-only operations were reachable without a session
**by design** — `signIn` is how a session begins. **Every operation this feature adds is supposed to
require one**, by §7's Authentication row and by §11's own criterion that they are "refused without a
session, before it reads anything". A stub cannot honour that criterion, and the docblocks say so
rather than leaving a later reader to assume it does. The three single-entry stubs record the same
about the not-found rule: there is no owner to compare against.

**And the closing move is two things at once**, which is worth knowing before checkpoint 16:
`schemasToSkipFiltering` lists only the three sign-in operations, correctly — so the moment an
`actual/` resolver lands for any of these five, that field **acquires a filter and the stub stops
being reachable**. The swap is a change of endpoint on the frontend *and* the arrival of a session
requirement on the backend, in the same move.

### The data hangs together by construction rather than by care

- Both query stubs write out the four categories from `sequelize/seeders/master/` — ids
  `10000001`–`10000004`, `transport` / `meals` / `supplies` / `other` — and the expenses stub
  references them as **named constants**, so an entry's category cannot drift from the master within
  the file. Both point at the seeder as the source rather than at each other.
- **Twelve entries, already in newest-`spentOn`-first order**, because a stub holds literals and the
  order *is* the data. A screen built against a differently-ordered stub would look right and be
  wrong.
- **No two entries share a `spentOn`, deliberately.** A same-date tie-break was not decided at
  checkpoint 1, and **data that raised the question would be deciding it.**
- One entry has a null memo, on the first page, so a screen meets §11's "the memo is genuinely
  optional" without going looking. One has an `updatedAt` later than its `createdAt` — the corrected
  entry from §11's second use case.
- `totalRecords` is `STUB_EXPENSES.length`, never a hand-written number.

### Two process notes, one of them my own error

- **My verification command in the assignment was wrong.** I gave
  `NODE_ENV=development npx jest …`, which fails every suite with "Cannot use import statement outside
  a module": `tests/setup-after-env.js` is ESM and needs
  `NODE_OPTIONS="--experimental-vm-modules"`, which `package.json`'s own `test` script exports before
  calling `test.sh`. The unit found it, corrected it, and reported it rather than working around it
  silently. Recorded because I had it right in earlier briefs and dropped it here.
- **Q33 fired again, exactly as checkpoint 3 predicted.** Running `_orders` left category rows behind,
  which turned `tests/__tests__/sequelize/seeders/master/expense_categories.js` red — a different
  feature's test, failing because of a shared SQLite file. The unit refreshed and re-ran. **Second
  independent trigger of the same order-dependence**, and the prediction one checkpoint earlier is
  what made it diagnosable in seconds rather than investigated.

**49 suites, 776 tests, lint clean.** Baseline was 736. No conflict-proof file needed a change —
`resolver-id-hash-staff.js` would have been one, and checkpoint 3 already filled it.


## Checkpoint 5 - three modules, and a brief of mine that was wrong

**Exit condition met.** Every module checkpoint 6 imports exists and was confirmed to resolve by the
main session, not taken from the units' reports - each unit sees its own module and none of its
siblings'.

| What checkpoint 6 imports | Resolves |
|---|---|
| `Expense.$.findAllWithPagination` | `function` |
| `CalendarDateInspector` - `isWellFormed()`, `isAfterToday()` | `true` / `false` at the boundary instant |
| `CALENDAR.TIMEZONE` | `Asia/Tokyo` |
| `RequestPagination#createFindOptions()` | `{"limit":5,"offset":10,"order":[]}` |
| `IntegerValueInspector` | `0` not positive; `'1200'` integer-like |
| fourteen seeded `expenses` rows | present after `db:refresh` |

**59 suites, 1087 tests, lint clean**, run by the main session with nothing else running. Baseline
was 49 and 776.

### The catalog check, which is this checkpoint's mandated first delegate

Run **once for the whole checkpoint** before any module was written, which is the point of the rule
- left to the units the search runs once per module and can answer differently each time. 33 tracked
packages. Three adopted, two declined, and the declines are recorded because "not used" and "not
considered" look identical later:

- **`renchan-sequelize`'s `PaginationMixinModel`** - already a dependency.
- **`mentsu-deep-value-converter`** - the instant-to-calendar-date conversion in a named zone.
- **`mentsu-value-inspector`** - the presence, positive and integer-like predicates. **This one
  aligns the feature with `input-validators.md`, which already prescribes delegating to a value
  inspector.** This repository had none and `#sign-in` hand-rolled its predicates, so the package is
  not a second style - it is the rule's style, arriving late.
- **`mentsu-field-path-value-extractor` - declined.** The only nested read this feature makes is an
  expense's category name, two levels deep, which `javascript-style.md` permits as direct property
  access.
- **`jest-deep-containing` / `jest-expect-each` - declined.** They back matchers the testing rule
  assumes and neither is installed, but nothing here needs them.

### My brief was wrong, and the unit is what caught it

I briefed `Expense.findAllWithPagination(...)`, **taking the catalog agent's report at its word**.
It does not exist - the mixin's statics land on a handler, so the call is
`Expense.$.findAllWithPagination(...)`. The unit probed it rather than following the brief, and
checkpoint 6 would otherwise have met `undefined is not a function`.

**The same report also said the date converter defaults to UTC when the timezone is not bound. It
throws.** I measured that one myself before briefing on it, which is the only reason it did not
propagate. Two claims from one report, one checked and one not, and the unchecked one was wrong -
that is the lesson rather than anything about the agent.

### Two traps turned into tests instead of notes

- **`findAllWithPagination` is `count()` + a scoped `findAll()`, not `findAndCountAll()`**, and the
  limit and offset arrive through the scope. **Putting `limit` into `options` silently produces a
  wrong total** - no error, a plausible number. Pinned by a test across a first page, a last partial
  page and a second owner's shorter set.
- **`count(options)` takes the same `include` the `findAll` gets.** Both associations are
  `belongsTo`, so nothing multiplies rows and the count is exact without `distinct: true` - asserted
  by a test that runs *with* the include. The comment states the limit of that claim, because the
  first `hasMany` include added here breaks it quietly, and `#monthly-summary` is the likely place.
- And the value object's getter is **`totalNumber`** while the contract's field is `totalRecords`.
  A mismatch that compiles.

### Q38 turned out to be already broken here, by a test older than the question

`expenses` held rows whose minimum id is `10000222` - block `100`, `#data-model`'s - because
`tests/_orders/Expense/Expense.js` calls `Expense.create({ id, ... })` directly. **So a test already
writes explicit ids into a product-written table**, which is exactly what Q38 says is unsafe.

Measured rather than reasoned: the real DDL carries the `AUTOINCREMENT` keyword, so SQLite keeps a
**high-water mark** and an explicit insert *below* it does not move it. Block `100` therefore sits
permanently beneath the mark this feature's block-`102` seeder sets at every refresh. Three things
hold that up and **none is written down anywhere** - that prefixes are issued ascending, that the
seeder runs at all (it did not exist until this checkpoint, and before it the mark was set by
whichever explicit insert ran first, which *is* the Q38 sequence), and that the dialect keeps a
high-water mark. The third is SQLite's; **production is MariaDB and it was not measured**, so "an id
is never reused after a hard delete" is not a claim made here about production. Q38 is amended with
all of it.

**The seeder's own safety is the stronger claim and is stated as such**: it runs once, in one
`bulkInsert`, before any product code in that process, so the collision mechanism is *absent* rather
than unexercised. The prohibition it implies - no test may give an `expenses` row an explicit id -
is only a rule, and it is written into the seeder's docblock rather than only here, so the person
about to break it meets it at the moment of breaking it.

### Q49, and what checkpoint 5 could honestly do about it

Section 11 refuses an expense dated after today and **nothing names a timezone** - not the spec, not
either env file, not `sequelize/config.cjs`. Measured at `2026-09-14T22:00:00.000Z`:

```
Asia/Tokyo -> today is 2026-09-15    an expense dated 2026-09-15 is accepted
UTC        -> today is 2026-09-14    the same expense is refused as future-dated
```

That is 07:00 JST - **the rule inverts for the first nine hours of every working day**, which is what
makes it a business rule rather than a detail. It went behind one constant set to `Asia/Tokyo` and
labelled in its own comment as Q49's *recommended reading, not a confirmed decision*.

**Answered after this checkpoint closed: `Asia/Tokyo`, and the constant now says so** (`1849f7d`).
It was put as its own question and answered as itself rather than waved through on a standing
approval. **The answer came from the peer session's user, not from this session's**, and both the
constant and Q49 say so rather than flattening it to "the user decided" - the same distinction
recorded for the `specs/` merge at `a9cb3dc`. Nothing observable changed, because the value was
already what the recommendation said; only its standing did. **The spec still names no zone**, so
that sentence is pull request #22 and a backend constant remains the only written record until it
merges.

**One correction to how I framed it when assigning the work:** I said an answer would change one
line. It changes **two** literals - the constant and the test asserting the default. The unit kept
the literal in the test deliberately, since a test reading the value under test would assert nothing,
and reported the discrepancy rather than quietly satisfying my sentence.

### My own process error, and it is the second of its kind

**I ran all three units in parallel in one repository.** They ran `npm test` concurrently against a
single SQLite file and corrupted each other: one `db:teardown` failed with the file busy, `test.sh`
swallows that with a skip-on-teardown fallback, the run continued against a database that was never
rebuilt, and it died several steps later on
`SQLITE_CONSTRAINT: UNIQUE constraint failed: expense_categories.id` - **surfacing as five unrelated
`_orders` suites going red.** Both units diagnosed it correctly, neither weakened anything to reach
green, and the re-runs were clean.

**This is Q33's shared-database order-dependence for a third time, and this time I caused it.** The
units were independent in the files they touched, which is what I checked; they were not independent
in the database they tested against, which I did not. The verification above was re-run serially by
the main session for exactly that reason.

**Also recorded because it cost a unit a confusing failure:** `npm test` and `npm run db:refresh`
both begin an `export NODE_ENV` line and npm defaults to `cmd.exe` here, which answers that `export`
is not recognized. `npm_config_script_shell=bash` is needed, and the acceptance gate will need it too.


## Checkpoint 6 - five real operations, and the security property that is an absence

**68 suites, 1654 tests, lint clean.** Baseline entering the checkpoint was 49 suites / 776 tests.
Every figure re-run by the main session serially, never accepted from a unit's report.

All five operations are served from the `actual/` pool. The five stubs stay where they are - the
frontend builds against them until checkpoint 16.

### The not-found rule is the absence of a code, so the absence is what is pinned

Section 7 and section 11 both say another member of staff's expense is answered as **not found,
never as forbidden**, and that the answer says nothing about whether it exists. At `removeExpense`
that collapses three situations into one, because section 7 deletes outright rather than archiving:

| situation | answer |
|---|---|
| this caller removed it a moment ago | `204.M006.002` |
| it exists and belongs to somebody else | `204.M006.002` |
| no row has ever held that id | `204.M006.002` |

**Proved across all three at once rather than pairwise** - two describes, six cases each, every case
carrying the same expected literal, and the already-removed ids are rows the describes above
genuinely deleted rather than simulated.

**The mechanism is one `where` and one missing code.** `findExpense()` narrows by `id` *and*
`StaffMemberId` in a single read, so another owner's row is never selected and the resolver has
nothing to tell the cases apart with. There is no second not-found code for a later reader to reach
for, and `errorCodeHash` is asserted **whole** with one `toEqual` - so an `ExpenseNotOwned` or
`ExpenseAlreadyRemoved` added later turns that test red instead of quietly leaking existence.

**Mutation-tested, not asserted and hoped:** dropping `StaffMemberId` from `correctExpense`'s
`where` produces six failures, including both identical-code describes and the untouched-page one.

### "Nothing is recorded" and "changes it in place" are each proved as both halves

Section 11's refusal criteria say *"and nothing is recorded"*. **The throw is not the proof.** The
refused call is the Arrange; the Act is a read of that member of staff's own page through the
product's own `ExpensesQueryResolver`, asserted whole including `totalRecords: 0`. No
`Model.findAll` or `count` appears in any of these files. **Mutation-checked** - flipping one case's
`0` to `1` fails, so the read is live rather than vacuously true.

**"Correcting changes it in place" needs the pair.** The read-back asserts `totalRecords` is the
owner's seeded 10 **and** that the corrected row comes back **under the id it already had**. A
delete-and-reinsert answers ten as well, and fails the first half. Either assertion alone would pass
a resolver that met the criterion's words and not its meaning.

### The timezone rule is load-bearing in a test rather than merely implemented

Every date case states its own `context.now`. The one that matters is `2026-09-15T15:30:00.000Z` -
00:30 on the 16th in Tokyo while UTC is still the 15th - where an entry dated `2026-09-16` is
**accepted** as today. **Under UTC that same case would be refused**, so its passing is the evidence
that `CALENDAR.TIMEZONE` is read, rather than that the arithmetic happens to be right. That is the
rule section 6 now carries, pinned by a case that fails if anybody reaches for the host's zone.

### Q46's mechanism ran backwards, five times, exactly as checkpoint 4 predicted

Each time an `actual/` resolver landed, a **passing** assertion in the stub-execution suite went
red: the operation stopped being reachable from the stub pool and started being refused without a
session. **A check met correctness and called it failure** - the mirror of Q46, where a check met a
defect and took its side. Both are one fact: a suite measures agreement between code and tests, and
agreement is not correctness.

Checkpoint 4's record predicted this before any `actual/` resolver existed. **Five for five**, and
each time the prediction made the red diagnosable in seconds rather than investigated. Every block
was rewritten into the stronger claim; none was deleted. The full amendment is in Q46.

The file's own header now answers the question its last rewrite raised - *what is this file for when
nothing in it is reached from a stub?* It is for what no unit test can ask: whether each operation
is reachable at all, and what a caller with no session gets back. Both are properties of the wiring
rather than of any resolver.

### A declared-but-unexercised branch, found by noticing an inconsistency

All three mutation resolvers declared `StaffMemberNotFound`, pinned it in a whole-hash `toEqual`,
and **nothing exercised the branch that raises it.** The query resolver had covered its equivalent
three ways since the checkpoint's first operation - **the inconsistency is what made it visible**,
not a coverage tool.

That is this project's "legal but empty" mechanism sitting in the code that enforces a session
requirement. Sixteen cases now cover it, each carrying a request that is *also* wrong in a way that
would answer a different code if the guard moved - so the **order** is pinned, which is what section
11's "before it reads anything" actually asks. Mutation-checked one guard at a time, the other two
files staying green each time, which is the attribution.

**And the exercise disproved a docblock.** It claimed that unguarded, a misconfiguration would write
a row owned by nobody. It would not: the insert fails at the column. **`Expense.verifyStaffMember()`
is not what stops it** - that hook returns early on a null owner and is a no-op for this exact case.
What a caller gets without the guard is `notNull Violation: Expense.StaffMemberId cannot be null`,
or where the key is absent, `WHERE parameter "staff_member_id" has invalid "undefined" value`. **The
guard closes a raw-driver-message leak, not an unowned row.** Corrected to what was measured - the
same shape as the CORS docblock corrected at `#sign-in`, a plausible sentence nobody had run.

### Two questions closed, neither by inventing an answer

- **Q50** - nothing fixed a maximum page size, so a caller could ask for a hundred thousand rows.
  `PAGINATION.MAXIMUM_LIMIT` is 100, behind its own constant, **chosen and not yet confirmed**, with
  the chooser named. Asked *after* `InvalidLimit` so `-3` is called invalid rather than excessive,
  and that order is asserted. The boundary tests write `100` and `101` as literals rather than
  reading the constant, because a test that read the value under test would assert nothing -
  mutation-checked by moving the maximum to 1000, which fails exactly the two over-maximum
  describes.
- **Q51** - the `_orders` suites assume a freshly seeded database and are not re-runnable alone.
  Correct, and held only by everyone happening to know it. `tests/_orders/README.md` now carries the
  reasoning and the failure signatures, and **each of the eight barrels carries a pointer, because
  the barrel is the file you must edit to add a suite.** A rule nobody meets at the moment of
  breaking it is not a control.

### Two of my briefs were wrong, and units caught both

- I briefed `Expense.findAllWithPagination(...)`, **taking a catalog report at its word.** It does
  not exist; the mixin's statics land on a handler. Checkpoint 6 would have met
  `undefined is not a function`.
- I briefed that an arg-less member is a plain `test()`. The rule's exception is **only** an arg-less
  static method and a static getter; an arg-less *instance* method still uses `test.each`.

Both were caught by units that checked rather than complied. **The pattern in both is the same one
that produced my two stalls**: a description standing in for the thing - announcing an action
without taking it, and relaying a claim without running it.

### Two stalls, seven days, and only one of them was a mechanism

- **A unit died on `ENOTFOUND` mid-run**, emitting no completion, so nothing woke this session. Its
  four files turned out to be sound - lint clean, its own suites passing - and the two failing tests
  were correct behaviour arriving. A continuation unit finished the two missing pieces rather than
  redoing good work. **Cost: three days.**
- **I ended a turn with "Dispatching `correctExpense` next" and did not call the tool.** Then did the
  same again with Q50 and Q51. **Cost: four days, then nine minutes** - the difference being a
  watchdog the peer session proposed and I accepted, which wakes this session after about a day of a
  clean tree with no movement.

**The two are indistinguishable from outside and have different fixes**, which is the argument for
the watchdog catching both and for not filing the second as the first.


## Checkpoint 7 - not applicable, established rather than assumed

**Nothing was written, and that is the finding.** The checkpoint's own clause says to decide this
**with the placement skill, not by eye**, so the decision was made by matching
`hor-execution-placement-pattern` and walking **all five operations through its flow individually**.
A checkpoint marked not-applicable with no stated reason is indistinguishable from one that was
skipped.

| Operation | Beyond answering the caller | Placement |
|---|---|---|
| `expenses` | nothing - no read receipt, no last-viewed stamp, no access log row | request path |
| `expenseCategories` | nothing - `context` is never even read | request path |
| `recordExpense` | nothing - one existence read and one insert, in one transaction | request path |
| `correctExpense` | nothing - and **no history row**: §9.3 declares no history table, and the only `_bk` tables in the spec belong to `#sign-in`'s secrets and digests | request path |
| `removeExpense` | nothing - and the spec **forecloses** the one candidate | request path |

Every row short-circuits at the flow's first or second step. **The trigger question - enqueue,
post-worker, or schedule - is never reached**, because nothing gets past "light work goes in the
API".

### The spec says it outright, which is better than the walk concluding it

§8's middleware footnote:

> **Redis is not declared, because this version runs no background job.** Every write finishes
> inside its own request, and nothing here leaves the process.

And §7's retention line kills the only candidate a hard delete would suggest - *"An entry its owner
removes is deleted outright rather than archived - what is retained is what remains."* So there is
no archive write, no tombstone, no cleanup sweep to schedule.

### The `#sign-in` precedent points the same way, for a reason worth keeping

`SignInMutationResolver` does two things that *look* like post-response candidates and runs both
synchronously. The interesting one is `recordSignInFailure()`: the attempt row is written **before**
the `InvalidCredentials` throw, not after the response. **It has to be** - §7's limit counts rows
that must already exist when the *next* request reads them, so deferring the write would let a burst
slip the limit. That is a write which is part of the main processing rather than a side effect.

**So the precedent is that this project puts everything in the request path deliberately**, and this
feature has strictly less to place than `#sign-in` did.

### §4's out-of-scope items would each need a placement, and none of it is this checkpoint's to invent

Approval by a manager (1.1.0) would put the decision in the request path and the **notification** in
a post-worker; the month export (1.2.0) is file generation, which is named on the skill's heavy side,
so a request-based Worker. **All of them would need Redis declared, which §8 deliberately does
not.** Adding a post-worker for a notification nobody has asked for would be code with no
requirement behind it, maintained and tested forever.

### Two findings from a checkpoint that changed no code

- **The skill had no digest, and my brief did not ask for one.** `.hora/digests/` held 40 files and
  `hor-execution-placement-pattern` was not among them, so the unit read all 216 lines of the skill
  instead. `/hora-build` takes a digest when a matched skill has none at the installed version; I
  omitted that step. Taken afterwards and pinned to `hora-skills-ort-renchan 0.1.0`, which is what
  `#monthly-summary` will read when it reaches the same checkpoint. **A process gap of mine, not the
  unit's.**
- **`ioredis` is a declared dependency with no importer.** Verified directly: `ioredis@^5.8.0` in
  `package.json`, a `redis:7.4` service in `docker-compose.development.yml` behind
  `profiles: [redis]`, and **zero imports** anywhere in `app/`, `server/`, `sequelize/` or `tests/` -
  only three identical commented-out `NOTE: Uncomment the following line to enable Redis PubSub`
  lines in the engines. `@openreachtech/renchan-job-bullmq` is absent, `app/jobs/` does not exist,
  `server/graphql/post-workers/` does not exist, and all three engines carry `postWorkersPath: null`.

  So §8's "Redis is not declared" is true of the **design** while the tree carries boilerplate
  residue. **Somebody reading `package.json` alone could conclude a job facility already exists**,
  which is exactly the wrong conclusion to reach at the moment a later version needs one. Recorded
  in the digest rather than fixed here: `package.json` is conflict-proof, and this is residue of the
  boilerplate rather than of this feature.

**68 suites, 1654 tests, lint clean - unchanged, and run anyway**, because "I changed nothing" and
"I broke nothing" are different claims.


## Checkpoint 8 - the audit, and a defect this feature opened rather than wrote

**`hor-security-audit`, run read-only over this feature's change set** - 58 files read from the
**working tree** rather than a commit range, because the backend commits do not land on a trunk
until the gate boundary after checkpoint 9, so a range would have been empty. Plus the five
operations the pinned contract declares, since a new caller wired to unchanged code still needs
auditing for authentication and exposure.

**0 HIGH, 1 MEDIUM, 1 LOW, 4 INFO.** Every one fixed or accepted-and-recorded; nothing left as a
silent pass.

### The non-disclosure property holds by shape rather than by check, which is the strongest form

The audit enumerated every read and write of an expense in production code. **There is no read that
narrows by `id` alone**, `staffMemberId` comes only from the session, and **none of the four input
types declares a field to take an owner from.** So "nobody sees anybody else's" is a property of the
contract's shape, not of a comparison somebody could forget.

It also closed a channel I had not asked about: **the input validators perform zero database
access**, so a `203.*` code can never be a function of whether a row exists. And in `correctExpense`
the owner read runs *before* the category read, so probing somebody else's entry answers not-found
whatever category is named - `ExpenseCategoryNotFound` is unreachable on a non-owned id.

### MEDIUM - this feature opened an amplification surface it did not write

Before `#expense-entry`, the staff audience exposed only sign-in operations: none paginated, none
DB-heavy. **`expenses` is the first**, and nothing bounded a document's shape. A GraphQL document may
repeat one field under many aliases, and the 16 kB body cap allows roughly **250 aliased `expenses`
calls** - each a `count()` *and* a `findAll()` of up to 100 joined rows, so about **500 database
round trips in one request**, on a path with no rate limiter.

**The limits are measured, not rounded.** All 24 GraphQL documents written in this repository's tests
and in the frontend were parsed: every real document asks for **exactly one** root selection, and the
deepest is **4**. Chosen 10 and 6. `getIntrospectionQuery()` measures **depth 15**, so depth beneath a
meta-field is not counted at all - GraphiQL is mounted, and a cap that broke introspection would be
a worse defect than the one it fixes.

**Two details that would have defeated a naive counter.** Fragments are expanded before counting,
because `query { ...Many }` bypasses a counter reading only the operation body. And a fragment is
never followed twice, because this runs *before* `NoFragmentCyclesRule` and a cyclic spread would
recurse forever - it under-counts a cycle deliberately, which is safe in exactly one direction:
under-counted means passed on to GraphQL, which refuses it for the cycle.

Amplification drops from ~250x to <=10x. Staff only: admin and customer serve one `healthCheck` each
and read no row, so there is nothing to amplify.

**And my brief was wrong, for the third time this feature.** I said renchan supports
`validationRules` and the engine "simply never sets one". The forwarding is real - and **an engine
cannot reach it**: `validationRules` comes from `GraphqlHttpHandlerBuilder.extraCreateHandlerParams`,
a static getter returning `{}` that never consults the engine, and `BaseGraphqlServerEngine` has no
such member at all. **Verified by the main session.** Using that seam would need two framework
subclasses plus a line in `server/index.js` - which cannot be imported here (Q24), so reverting it
would silently remove the limit **with nothing failing.** The limit is express middleware instead,
where the whole chain is testable today.

### LOW - a hazard about the future, not the present

renchan builds its resolver map from the **union** of the actual and stub pools but its **filter**
map from the actual pool alone. A stub-only name therefore gets `filter === undefined`, and the
wrapper's `filter?.()` is a no-op - no `Unauthenticated`, no `Unauthorized`.

**Nothing is exposed today**: every operation has both, so the actual always wins. **The hazard is
that deleting or renaming one `actual/` resolver would silently convert its operation into an
unauthenticated, state-changing stub answering fabricated success** - and this feature added five
such pairs, three of them mutations, the largest set in the repository.

The guard derives both sets from the real loader off each engine's own config, with a non-vacuity
case, for all three audiences - **a hand-written list would rot into exactly the false pass it
guards against.** Proved to fire by moving `ExpensesQueryResolver` out of the tree.

### The INFO that was mine

`ExpensesInputValidator`'s class comment still said the limit was uncapped - **in two places**. I
added the cap at checkpoint 6 and left the comment. The audit's reasoning is why it matters: a stale
comment invites a later maintainer to delete the cap as an invention, **which would reopen the
MEDIUM.**

### Accepted and recorded rather than fixed

- **Q53** - the `sort` echo (injection path closed by construction; a live instruction for the
  frontend not to render it as markup) and the model hook's row-id messages (unreachable from any
  resolver path, masked in production).
- **Q54** - the two boilerplate audiences still carry `origin: '*'` and 10 mb bodies. Harmless while
  they serve one `healthCheck`; **the exposure arrives silently the moment either gets a real
  operation.**
- **Q55** - nothing rate-limits the read path. The shape limit caps one request's fan-out, not
  requests per second, and §7 asks for a limiter only on the two sign-in operations. **Raised as the
  honest limit of the fix rather than as a new discovery.**
- **`ioredis`** - correctly scoped **out** by the audit: it predates this feature. My brief said it
  was this feature's, and it is not. Already recorded at checkpoint 7.

**70 suites, 1767 tests, lint clean**, re-run by the main session.


## Checkpoint 9 - the use cases walked as real calls, not read as code

**All three of §11's use cases complete against the API as built.** Checkpoint 2 verified them
against the *spec*; this verified them against **the thing that got built** - by signing in as a
seeded member of staff and issuing real GraphQL documents through the built schema, in order, with
the real shapes printed at every step.

**Nothing fell short, so nothing was sent back to checkpoint 3.** No field was added on the way past.

### How it was driven, and the one obstacle worth recording

`server/index.js` still cannot be imported on this machine, so a plain `node` script died at
`ERR_UNSUPPORTED_ESM_URL_SCHEME` (Q24, and its WSL amendment). **The walkthrough therefore ran
through jest**, which supplies its own module registry and resolves the specifier Node's loader
refuses - the same reason the whole suite runs here at all. Same schema, same resolver pools, same
engine, same context class, **real access token on a real header**. What it still does not evidence
is the socket: express, the middleware chain, the body cap and the CORS allow-list. Q24's amendment
now records that the socket is **unreached rather than unreachable**.

The walkthrough was a throwaway - written, run, read, and deleted. The tree is clean.

### Use case 1 - records a fare that evening and sees it in their entries

```
expenseCategories -> transport 10000001, meals, supplies, other       (fills the category field)
recordExpense     -> { expenseId: 10200015 }                          (the identifier, and nothing else)
expenses          -> id 10200015, 2026-09-20, 1200, 'Taxi back...', recorded, transport
                     pagination { limit 5, offset 0, totalRecords 1 }
```

### Use case 2 - typed 12,000 instead of 1,200, opens that entry, corrects it

```
recordExpense  -> { expenseId: 10200016 }  amount 12000
expenses       -> 10200016 comes back with its CURRENT values, so "opens that entry" needs no
                  read-one operation. This is checkpoint 2's finding, now observed rather than argued
correctExpense -> { expenseId: 10200016 }
expenses       -> id 10200016 STILL, amount now 1200, totalRecords still 2
```

**Corrected in place, and both halves are visible in one answer:** the same id, and a count that did
not move. A delete-and-reinsert would have shown a new id above the block.

### Use case 3 - recorded the same lunch twice, removes the duplicate

```
recordExpense -> { expenseId: 10200017 }
removeExpense -> { expenseId: 10200017 }
expenses      -> 10200017 absent, totalRecords back to 2
```

### The non-disclosure rule, as a caller actually experiences it

| what was asked | answer |
|---|---|
| remove an entry already removed | `204.M006.002` |
| remove an id no row has ever held | `204.M006.002` |
| correct an entry belonging to `10110002` | `204.M005.002` |

**Identical within each operation, and carrying nothing else** - no message, no row, no count. A
caller cannot tell a removed entry from one that never existed from one that is somebody else's.
Observed end to end rather than inferred from the error hash.

### The refusals §11 names, and the optional memo

```
amount of zero        -> 203.M004.005
dated after today     -> 203.M004.007
no memo presented     -> { expenseId: 10200018 }   accepted
no session presented  -> 102.X000.001              the engine, before anything is read
```

**And a gap in my own first walkthrough, worth recording because it is the shape of a bad
verification.** I asserted the optional memo by recording one without a memo and then reading page
one - where it did not appear, because ordering is `spentOn` descending and that row was the oldest.
**I had shown that the write succeeded, not that the memo reads back empty**, which is what §11
actually asks. A second call at `offset: 2` returned it:

```
{ id: 10200018, spentOn: '2026-09-19', amount: 800, memo: null, status: 'recorded',
  expenseCategory: { id: 10000003, name: 'supplies' } }
```

`memo: null` crosses as a JSON null rather than a failure. The near-miss is the point: **a
verification that stops at the first plausible-looking answer proves the thing next to the claim.**

### One more property confirmed in passing

`limit: 2, offset: 2` answered `totalRecords: 3` - **the total is the whole set, not the page.** That
is the `totalNumber` to `totalRecords` mapping and the "limit never reaches `options`" rule from
checkpoint 5, observed through the API rather than through a unit test.

**70 suites, 1767 tests, lint clean**, re-run after the walkthrough was deleted.

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
