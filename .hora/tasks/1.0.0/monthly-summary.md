# #monthly-summary  A chosen month's entries, with their total
<!-- spec: monthly-summary @ sha256:4d2c078e0d83906e -->
<!-- repositories: backend, frontend-staff -->

Conflict: appends to the staff audience's SDL and to its GraphQL type declarations under
          `types/`. Two other features carry the same mark, and both come first — re-read the
          real files before writing

Constraint: exporting a month for a claim form is out of scope **for now** (1.2.0, spec §4).
            **This is the seam:** one operation returns both the month's entries and its total,
            so that the two can never disagree and so that the export calls this rather than
            summing again. Do not split them into two operations, and do not let the screen
            sum the rows itself

Constraint: no total is stored (spec §7). The month is a single indexed read, summed per
            request, and it stays that way at the foreseen size

Constraint: more than one currency is **permanently** out of scope (spec §4). `totalAmount` is
            an integer number of yen, typed `Int!` (Q2)

Note: the month-boundary rule is an acceptance criterion — the first and the last day of the
      chosen month are **in**, the day before and the day after are **out**. This is why
      `spentOn` crosses the contract as a plain ISO date rather than a datetime (Q3): an
      invented midnight plus a timezone shift can move a day across the boundary

Note: an empty month **says the month is empty** and shows a total of zero. It is not an empty
      table (spec §12 acceptance) — the frontend convention has an empty-state component for
      exactly this

Note: this operation takes no pagination (spec §12.1), unlike `expenses`. Spec §7 caps the
      heaviest read at a few hundred rows

Note: another member of staff's expense in the same month must change neither the entries nor
      the total. Same scoping rule as #expense-entry, and checkpoint 8 reads it here too

## Spec gate
- [x] 1. Draft or confirm the specification  <!-- interactive, main session. Section 12 read against sections 4, 6, 7, 9.3 and the pinned contract. TWO GAPS, both already on release and both would have been built on. Q60: section 7's page-size row claimed monthlyExpenses inherits a 100-row ceiling it cannot have -- no pagination input exists, the contract says so, and section 7's own heaviest-operation row says a few hundred rows. I WROTE THAT CLAUSE and it passed my review, a peer's and a user decision. Q61: a month's entries had no stated order; decided by the user as section 6's entry order row, one sentence for both screens -- and MEASURED to be a correction rather than a tidying, since MariaDB returned tied rows the opposite way from SQLite while the CI job was green over it for want of an assertion. Three things checked and sound: the timezone (section 6 already had it), section 4's approval seam (honoured by construction, since the total IS the sum of the entries shown), and the absence of pagination. Nothing invented: no bound on year or month was proposed, because that a month is 1-12 is what the word means. AND Q60's correction had reached the spec only: constants/paginationConstants.cjs still carried the reversed claim twice, once citing section 7 as its authority -- corrected here by rewriting. Found late because the root search for the clause returned clean -- and THE CAUSE I FIRST RECORDED FOR THAT WAS ALSO FALSE. It was not the nested repository (a root grep descends into it fine, verified); the phrase WRAPS across a line break, so it exists on no single line and no line-oriented search could match it. A wrapped sentence is already one of this project's six recorded verification failures, hit again and then misdiagnosed as a different one. Q58 twice in one finding, and the wrong cause had reached five artefacts before it was tested -->
- [x] 2. Verify the use cases can be met  <!-- interactive, main session. Both of section 12's use cases walked against the one operation section 12.1 declares; both met by the same operation with different input, which is the design section 4's export seam asks for. All six acceptance criteria reachable, and two are STRUCTURAL rather than policed: the total equals the sum of the entries shown because one operation returns both and there is no second computation to disagree, and the month boundary cannot drift because spentOn is DATEONLY crossing as YYYY-MM-DD so no invented midnight can move a day. No operation section 12.1 does not declare is needed -->


## Checkpoint 1 - the specification confirmed, and two gaps that had already reached release

**§12 is coherent with §4, §6, §7, §9.3 and the pinned contract** - after two corrections. **Both
gaps were already merged into `release/1.0.0` before this reading**, so both would have been built
on rather than found. A third correction follows from the first and is still in flight: backend
PR #16, which fixes the file that cites the clause §7 had already lost.

### Gap 1 - §7 claimed this feature inherits a ceiling it cannot have, and I wrote the claim

§7's `Page size` row, merged at `f547202` as Q50's answer, ended: *"`expenses` is the only paginated
operation in 1.0.0; **`monthlyExpenses` inherits the same ceiling**."*

**False on three counts**, and it put §7 in contradiction with itself:

| | |
|---|---|
| §12.1 | `MonthlyExpensesInput(year, month)` - **no pagination input at all** |
| the pinned contract | a comment reading *"No pagination: spec 7 caps the heaviest read at a few hundred rows."* |
| §7's own *heaviest single operation* row | *"a few hundred rows read and summed"* - **more than 100**, in the operation the clause named |

**I added the clause to make the rule sound general**, and generality was the wrong instinct: it
described an operation I had not re-read when §12.1 was a grep away. **It then passed my review, a
peer's review and a user decision** - because the row is self-consistent and reads as thorough.
Seeing it required holding §12.1 in mind, which is what checkpoint 1 of the feature that owns §12 is
for. Recorded as **Q60**, corrected at `738dd6e`.

### Gap 1 continued - the correction reached the spec and stopped there

Q60 corrected §7. **It did not correct the file that cites §7**, and nothing would have caught that:
`constants/paginationConstants.cjs` carried the reversed claim twice over, in the docblock a reader
opens precisely when they want to know what the number means.

| line | what it said | |
|---|---|---|
| 10 | *"`#monthly-summary` inherits this pagination shape"* | there is no pagination shape to inherit |
| 14-15 | *"`monthlyExpenses` inherits the same ceiling **by that row's own words**"* | the row's own words now say the opposite |

**The second is the worse one, because it cites its authority.** A reader who checks the citation
finds §7 saying `monthlyExpenses` is deliberately unpaginated, and is left deciding which of two
confident sentences to believe.

**Corrected here, by rewriting rather than appending**, on the same reasoning as §6's resolver
docblock a week ago: a docblock stating a reversed rule is worse than none, because it is a claim a
later reader relies on, written by somebody who was right at the time. The stale sentences are gone
and what replaced them says the bound is §7's *heaviest single operation* row instead - a few
hundred rows, **more than this 100 and always was**.

### And the first explanation I recorded for how it was missed was itself false

**Worth reading in the order it happened, because the second error is the more instructive one.**

The first search for surviving copies was `grep -rn "inherits the same ceiling"` from the repository
root. It returned exactly one hit - **the checkpoint record's own quotation of the false clause** -
and read as clean.

**The cause I recorded was that the root search cannot see a nested repository.** Plausible,
confidently written, and **false**. A root `grep -rn` descends into `expense-note-backend` perfectly
well; a control search for another phrase in that same file matched it on line 4.

**What actually happened is that the phrase wraps:**

```
 * `limit` is refused as invalid input rather than silently reduced. `monthlyExpenses` inherits the
 * same ceiling by that row's own words. **Read the spec, not this comment, for what the rule is**
```

`inherits the` ends line 14 and `same ceiling` begins line 15, so **the string being searched for
does not exist on any single line of the file**, and a line-oriented tool cannot match it however
many repositories it descends into. What found it was a later search that happened to include the
bare token `monthlyExpenses`, which sits whole on line 14.

**A wrapped sentence is already one of this project's six recorded verification failures.** So this
is not a new mechanism - it is that one, hit again, and then **misdiagnosed as a different one**.

**Both steps returned a plausible value rather than an error, which is Q58 twice in one finding.**
The grep returned a clean result rather than saying it could not match across lines; and my
diagnosis returned a cause that explained the symptom rather than saying it had not been tested.
**The second is the worse half: a wrong search costs a search, and a wrong cause gets written into
five artefacts as the lesson** - which is where it was caught, by testing the claim before letting
it stand rather than after.

**The rule that follows is about the search, not the repositories**: a phrase long enough to wrap is
searched for by a short unwrappable token inside it, or with a tool that reads across lines. A
clean result from a multi-word grep over prose is not evidence of absence.

### Gap 2 - a month's entries had no stated order, and the tie was wrong in production

§12 never said what order entries come back in. §11.2's clause stopped at `spent_on` with no
tie-break, deliberately: `#expense-entry`'s checkpoint 1 declined to invent one.

**Within a single month that decision stops holding.** A train fare and a lunch on the same day is
one ordinary working day, and §12's own second use case has somebody reading down the same column
twice against a card statement.

**Decided by the user** (Q61) over two named alternatives: §6 gains an `entry order` row - newest
`spent_on` first, and where two share a date, the more recently recorded first - **one sentence for
both screens**, because one screen breaking the tie while the other did not is the same disagreement
by another route.

**And the measurement made it a correction rather than a tidying.** Without the tie-break:

| engine | tied rows | |
|---|---|---|
| **SQLite** - every local suite | **descending** by id | *accidentally* §6's order, via §9.3's index walked backwards |
| **MariaDB** - §8's store | **ascending** by id | **the opposite of §6** |

**`live` was answering a day's entries oldest-recorded first while every local suite reported the
required order.** And the MariaDB CI job had been green over it since `#sign-in` - **not for want of
an engine, but for want of an assertion.** Running on the right engine buys nothing if nothing
asserts the property.

### Three things checked and found sound, recorded so nobody re-checks them

- **The timezone.** §6's `month` row already carries *"read in `Asia/Tokyo`"*, added by #22 at
  `#expense-entry`. A month boundary is the same question as "dated after today" with more at stake
  - an expense at 23:00 on the 31st belongs to a different month depending on the answer - and §6
  answers it once for both.
- **§4's approval seam.** §4 requires that *"every read goes through `status` rather than assuming a
  recorded expense is a final one"*. §12 never mentions `status`, and does not need to: the shared
  `Expense` type carries it, and **the total is defined as the sum of the entries shown**. So
  whatever filter 1.1.0 applies to the entries flows to the total automatically. **The one-operation
  design is what makes that true** - the two cannot disagree because they are one answer.
- **No pagination, and that is consistent rather than an omission.** §7's *heaviest single operation*
  row bounds it at a few hundred rows; §12.1 declares no pagination input; the contract says why. All
  three agree, now that Q60's clause is gone.

### Nothing invented

**No clause was proposed for a bound on `year` or `month`.** That a month is 1-12 is what the word
means rather than a decision somebody owes, and §12's silence on it is not a gap. A future month
reads as empty, which is truthful - and §11 already refuses a future-dated expense, so one cannot be
made to appear there.

## Checkpoint 2 - the use cases can be met, walked against the one operation §12.1 declares

| §12 use case | The path | Verdict |
|---|---|---|
| filing a claim at the end of the month: picks that month, reads the total, writes it on the form | `monthlyExpenses(year, month)` returns `totalAmount` in the same answer as the entries | **met** |
| checking last month against a card statement: switches to the previous month and reads its entries **without leaving the screen** | the same operation with a different input; §12.2's call table says it is called *"on opening, and on moving to another month"* | **met** |

**One operation serves both use cases**, which is the design §4's export seam asks for rather than a
coincidence.

### Every acceptance criterion is reachable, and two are structural rather than policed

- **"the total equals the sum of the entries shown"** - **structural.** One operation returns both,
  so there is no second computation to disagree. A test can confirm it; nothing can break it without
  changing the operation's shape.
- **"the first and last day are in, the day before and after are out"** - a date comparison on
  `spent_on`, which is `DATEONLY` and crosses as `YYYY-MM-DD` (Q3), so no invented midnight can move
  a day across the boundary.
- **"an empty month shows a total of zero and says the month is empty"** - `totalAmount: Int!` is
  non-null, so zero is expressible and absent is not; the empty-state component exists.
- **"recorded, corrected or removed is reflected on the next read"** - no total is stored (§7), so
  the month is summed per request.
- **"another member of staff's expense changes neither"** - the same scoping rule `#expense-entry`
  holds, and checkpoint 8 reads it here too.
- **"refused without a session, before it reads anything"** - the engine's filter, as long as
  `monthlyExpenses` is absent from `schemasToSkipFiltering`.

**No operation §12.1 does not declare is needed**, and nothing went back to `/hora-spec` from this
checkpoint that was not already corrected before it ran.

## Backend gate
- [x] 3. DB and API schemas  <!-- skills: hor-graphql-schema, hor-type-interface, hoc-naming, hoc-jsdoc; digests ALL REUSED at hora-skills-ort-renchan 0.1.0 and hora-skills-ort-core 0.2.0, none taken, the installed versions being unchanged. NO MIGRATION AND NO MODEL CHANGE, established by reading #data-model's create-table migration rather than assumed: it ships every section 9.3 column plus the composite (staff_member_id, spent_on) index, cut for this feature's read before this feature existed. Four files: the SDL, its test, the two type interfaces, and resolver id Q004. Expense is REFERENCED, never redeclared -- the contract's own instruction and section 4's export seam expressed in the schema. Contract agreement measured TWICE by two different instruments (field-by-field, and printSchema of the whole schema as one string): IDENTICAL. MY BRIEF WAS WRONG A FOURTH TIME -- I briefed three files and it is four; the SDL test precedent begins at exactly the checkpoint I told the unit to mirror, and complying would have shipped a surface with no test while its sibling had one. Two digests found to disagree with EACH OTHER on where type declarations live; the tree outranks both -->
- [x] 4. Stub API  <!-- skills: hor-stub-api; digest REUSED at hora-skills-ort-renchan 0.1.0, none taken. One schema-accurate stub, same class name and signature the actual will carry, seven September 2026 entries already in section 6's entry order with no sorting applied. totalAmount is SUMMED from the stub's own rows, never hand-written, because section 12's first criterion holds by construction and a hand-written total could disagree with the rows beside it -- a deliberate, recorded deviation from hor-stub-api's only-one-computation line, the spec outranking the digest as the contract outranked a skill at Q48. Four of section 12's criteria are NOT testable at a stub and the docblock disclaims them rather than implying them. THE STUB-FILTER GUARD FIRED FOR THE FIRST TIME IN ITS LIFE and it was right: a stub-only operation is served with no authentication filter, monthlyExpenses's actual is checkpoint 6's, so tests/__tests__/ is RED by exactly one case. Left red rather than exempted -- an exemption is the very edit that test exists to make look like a security change -- bounded by closing at checkpoint 6, by never reaching release (the branch merges only after checkpoint 9), and by every checkpoint verifying the failing set is EXACTLY that one case. The guard's own docblock had gone false and was corrected, comment only. Q62 raised -->
- [x] 5. The modules the implementation needs  <!-- catalog check FIRST and once, then ONE implementer unit. 33 tracked packages searched: ADOPTED mentsu-value-inspector (already a declared dependency, already the pattern); DECLINED mentsu-search-condition, renchan-funnel, mentsu-validation-rules, mentsu-schema's DateonlyScalar and mentsu-value-normalizer, each with its reason, because 'not used' and 'not considered' look identical later. NOTHING in the catalog computes a month boundary and THIS REPOSITORY HAS NO DATE LIBRARY AT ALL, so CalendarMonthRangeBuilder was written fresh -- a new class rather than a method on CalendarDateInspector, whose create() requires a calendarDate a month range does not have. The class exists because section 7 requires a single indexed read against section 9.3's composite index, and YEAR()/MONTH() would be non-sargable AND does not exist in SQLite. MY BRIEF WAS WRONG A FIFTH TIME and it changed the design: the range takes NO timezone, because spentOn is DATEONLY and a month's two ends are the same strings in every zone -- section 6's Asia/Tokyo governs deriving a date from an instant, which is the recording side. The unit added the century leap-year cases (1900 and 2000) I had not asked for, which are the ones that prove the technique rather than the result. Exit condition checked as the clause demands: all six imports checkpoint 6 will make CONFIRMED TO RESOLVE by the main session in one process. No seeder needed -- June 2026 already has both boundary days plus a second member of staff's row; prefix 103 allocated for checkpoint 6's own fixtures. Failing set still EXACTLY the one checkpoint-4 reconciliation -->
- [x] 6. Actual API  <!-- skills: hor-query-resolver, hor-resolver-validator, hor-constant-definition, hor-backend-testing, hoc-jest; digests ALL REUSED at hora-skills-ort-renchan 0.1.0 / hora-skills-ort-core 0.2.0. Both suites GREEN: __tests__ 1693/1693, _orders 326/326, lint clean, every figure re-run by the main session. Q62'S WINDOW IS SHUT -- reconcile-stub-resolvers-with-actual went green ON ITS OWN when the actual resolver landed and was never touched, which is this checkpoint's own exit evidence. The total is summed over the rows returned, never a second SUM(), because section 12.1's reason for one operation is that the two cannot disagree and a second query is a second observation. WHAT THE VALIDATOR DELIBERATELY DOES NOT DO is the interesting half: no minimum year (retention says how long a row is kept, not which months may be asked for), no maximum and no not-in-the-future rule (section 11 refuses a DATED expense, a rule about writing, and refusing a future month would break section 12.2's navigation). The 1-9999 year bound is the DATE FORMAT's constraint, measured: an unbounded -5 builds '00-5-06-01' which returns a plausible empty read, so the bound turns a plausible empty month into a stated refusal -- Q58 answered at the input. A CHECKPOINT 3 MISS CLOSED HERE: section 6's entry order was a private constant of ExpensesQueryResolver and this feature owed the identical clause, so two copies could drift and reintroduce Q61's defect; declared once now, and deleting the second key fails 4 tests across BOTH resolvers. Five mutation checks; the sharpest is that 12 tests catch a widened range and NONE catches it at the upper end except the 2 in _orders. My brief was wrong a sixth time about the test path -->
- [x] 7. Worker  <!-- NOT APPLICABLE, established with hor-execution-placement-pattern (digest REUSED at hora-skills-ort-renchan 0.1.0) rather than by eye. This feature declares ONE operation, so the per-operation walk is complete rather than sampled. monthlyExpenses is a query: it short-circuits at the flow's FIRST question (is it a write?) and the trigger question at step 4 is never reached. The aggregation was the one thing that could have argued otherwise, and the skill's own heavy/light rule answers it -- heavy means external I/O, AI calls, large record counts or file generation, while section 7 bounds this at a few hundred rows under one second over rows already in hand, which is the skill's 'short aggregations within a single request'. Section 8 says it outright in the spec (Redis is not declared, because this version runs no background job), so this walk CONFIRMS rather than decides. Section 4 forecloses the one candidate: CSV export is file generation and so the heavy side by name, but it is 1.2.0 and its seam makes it a CALLER of this operation rather than a rewrite. Nothing written, nothing invented to make the checkpoint non-empty. One finding restated because it is still true: ioredis remains declared with ZERO importers, so package.json still suggests a job facility section 8 says does not exist -->
- [x] 8. Security audit  <!-- hor-security-audit, read-only, in a VERIFIER as the clause requires. THE DIGEST DID NOT EXIST -- Q59's fourth instance -- so it was taken first and pinned at hora-skills-ort-renchan 0.1.0, verified against the installed package. 0 HIGH, 5 MEDIUM, 1 INFO, and NO FINDING ORIGINATES IN THIS FEATURE'S CHANGE SET. Scope arbitration recorded rather than silently resolved: the skill's frontmatter scopes it repo-wide and redirects diffs to /security-review, checkpoint 8 overrides that in writing with its reason, and /security-review is not equipped here anyway. Change set read as release/1.0.0..feature/monthly-summary, which needed justifying because the clause assumes a range would be EMPTY before the gate -- here every checkpoint is committed on the feature branch and the backend tree is clean, so both views agree. Q63 RAISED: introspection is enabled in every environment and is RECORDED NOWHERE -- a grep over all questions and all acceptance records returns nothing, so it was either never checked or judged out of scope without being written down, and THE RECORDS CANNOT DISTINGUISH THOSE. A check that was run and passed and a check that was never run look identical in a record that only lists findings; Q58's family as an absence. Q55 AMENDED because this feature holed one of its own mitigations: monthlyExpenses is the product's first read with no enforced row ceiling, and the shape limit's arithmetic was measured against a 100-row-per-alias read. The other four MEDIUMs are Q16 and Q54, verified still exactly as recorded. Six checks answered only partially, each NAMED, per the skill's own rule that a gap must never look like not applicable. Q62's window confirmed shut by a second instrument. My brief was wrong a seventh time about the change set's contents -->
- [x] 9. Verify the use cases again, against the built API  <!-- main session, in conversation. Both of section 12's use cases driven as REAL CALLS against the built schema with a real access token on a real header, through the same GraphqlSchemaBuilder the server uses, nothing stubbed; in process because server/index.js cannot boot here (Q24). June 2026 answered totalAmount 3451 over 2026-06-30 (1) and 2026-06-01 (3450) -- the sum is right, both month ends are present, the order is newest spentOn first, and 10110002's 2026-06-18 (1860) is ABSENT from both the list and the total, which would read 5311 if scoping leaked, so the number itself is the evidence. Use case 2 moved to July (2100) and May (0, empty) ON THE SAME TOKEN with no second sign-in, which is the half of 'without leaving the screen' a code reading cannot establish. No session answered 102.X000.001 -- the ENGINE's Unauthenticated code, not this resolver's, so the refusal came before resolve() was entered; month 13 answered 203.Q004.002. Nothing fell short, nothing went back to checkpoint 3, and NO FIELD WAS ADDED ON THE WAY PAST. Walkthrough deleted, tree clean. One mechanical finding: a plain-named file under tests/_orders/ does not run at all -- jest.config declares no testMatch so only *.test.js matches -- and the first attempt reported 'No tests found, exiting with code 0', a pass-shaped answer to a question nobody asked. Q58 again -->

## Checkpoint 3 - one query's surface, no table, and a fourth wrong brief

**The DB half is not applicable, and it was established rather than assumed** - which is what this
checkpoint's own clause asks for. `#data-model` shipped §9.3's `expenses` table complete: every
column, and **three indexes including the composite `(staff_member_id, spent_on)`**, whose migration
carries the comment *"One member of staff's month is the heaviest read this version has."* That
index was cut for this feature before this feature existed. **No migration, no model change.**

### What was written

| file | |
|---|---|
| `server/graphql/schemas/staff/004-monthly-summary.graphql` | new. `type Query`, `MonthlyExpensesInput`, `MonthlyExpensesResult`, verbatim from the pinned contract |
| `tests/__tests__/server/graphql/schemas/staff/004-monthly-summary.js` | new. 4 tests |
| `types/StaffGraphQL.d.ts` | the two interfaces, `expenses` typed `Array<Expense>` |
| `server/graphql/resolver-id-hash-staff.js` | `monthlyExpenses: 'Q004'` |

**`Expense` is referenced and never redeclared**, which is the contract's own instruction and §4's
export seam in the schema: a month's entries and the total taken over them cannot describe different
shapes if there is only one shape. The loader concatenates the directory into one schema, so a
second declaration would be a duplicate type rather than an override.

**There is deliberately no MUTATION section.** §12.1 declares one query and nothing else.

### The agreement with the contract was measured twice, by two different instruments

Not by eye, and not once. **Two instruments agreeing is the point** - Q58 is this project's standing
finding that a single verdict is evidence about the instrument as much as about the subject.

| | what it compared | result |
|---|---|---|
| the unit | type-by-type and field-by-field, including argument names and full nullability | 25 types, 62 fields, no DIFF/EXTRA/MISSING |
| the main session | `printSchema(lexicographicSortSchema(...))` of the **whole** schema against the whole contract, as one string | **IDENTICAL** |

```
node <script>   # built through the framework's own SchemaFilesLoader, as the server loads it
types   SDL 28 / contract 28      <- 28 counts built-in scalars, 25 does not; same convention both sides
fields  SDL 62 / contract 62
Query   expenseCategories, expenses, monthlyExpenses, signedInStaffMember
RESULT: printSchema IDENTICAL
```

**The type counts differ between the two instruments and neither is wrong** - they filter built-in
scalars differently, and each compared like with like. Recorded because a reader meeting `25` and
`28` for the same schema deserves the reason rather than a doubt.

### My brief was wrong for the fourth time, and the unit caught it again

**I briefed three files. It is four.** `#expense-entry`'s checkpoint 3 also left an SDL test, at
`tests/__tests__/server/graphql/schemas/staff/003-expense-entry.js`, and **that directory exists
solely for it** - 001 and 002 have no such test, so the precedent begins at exactly the checkpoint I
told the unit to mirror.

**Had the unit complied, this checkpoint would have shipped a schema surface with no test behind it
while its sibling had one.** The unit read the tree instead and wrote the parallel file.

**Four for four now, and the mechanism is the same one every time**: I brief from what I remember of
a sibling rather than from the sibling. The defence is the unit's licence to trace rather than
comply, and it has now paid four times.

The test it wrote asserts the **merged runtime schema** rather than this file's text, and asserts
`print(astNode)` of each whole definition - so **an added field fails it as well as a missing one**.
It also pins the printed `Expense`, which fails if `#monthly-summary` ever gives itself a second row
shape.

### A disagreement between two digests, recorded rather than silently resolved

`hor-type-interface` prescribes one file per resolver at `types/resolvers/<category>/<name>.d.ts`
under `namespace graphql.<category>`. `hor-graphql-schema` prescribes one
`types/<Audience>GraphQL.d.ts` under `namespace server.graphql.<audience>`.

**The two digests disagree with each other, not merely with the tree.** The tree does the latter and
the tree outranks a digest, so that is what was followed - but a future unit reading
`hor-type-interface` alone would be led somewhere the tree does not go.

### Verification

Every figure re-run by the main session, not taken from the unit's report.

```
npm_config_script_shell=bash npm run lint                                    clean
npm_config_script_shell=bash npm test -- --seeded --maxWorkers=3 tests/__tests__/   65 suites / 1516 tests
npm_config_script_shell=bash npm test -- --seeded --maxWorkers=3 tests/_orders/      8 suites /  316 tests
```

`__tests__` was 64/1512 before this checkpoint, so the delta is exactly the four new tests. **The
unit ran only `__tests__`; `_orders` is the main session's addition**, because CI runs both and a
green claim that covers one of them is the narrower claim said as the wider one.

Committed as two commits matching `#expense-entry`'s own granularity, read out of its history rather
than chosen: the SDL with its test, then the types with the resolver id.

## Checkpoint 4 - one stub, and a guard that fired for the first time

**The stub exists and the checkpoint's own exit condition is met**, but `tests/__tests__/` **is
red**, deliberately and by exactly one test. That is the whole of this checkpoint's interest.

### The stub

`stub/queries/MonthlyExpensesQueryResolver.js`, same class name and signature the actual will carry
at checkpoint 6, so that swap is a change of endpoint rather than a rewrite. Seven September 2026
entries, written already in §6's `entry order` with **no sorting applied** - a stub holds literals,
and ordering them by hand is how it stays literal. First and last day of the month present; two
entries sharing a date, written later-recorded-first, so a screen meets §6's tie-break on its first
read; one null memo; one corrected entry; every `status` `recorded`.

**`totalAmount` is summed from the stub's own rows and never hand-written.** §12's first criterion
is that the total equals the sum of the entries shown, and checkpoint 1 established that this holds
*by construction*. A hand-written total could silently disagree with the rows printed beside it, and
a screen built against that would look right and be wrong.

**That is a deliberate deviation from `hor-stub-api`, recorded rather than slipped in.** The
digest's general rule forbids `map`/`filter`/`reduce` **over input**, which this is not - the reduce
runs over a module-level hardcoded array. But its stub-specific line says the pagination slice plus
`.length` is *the only computation any stub may perform*, and this is a second one. **The spec
criterion outranks the digest**, the same arbitration Q48 settled between the contract and a skill.
The deviation is kept minimal: the reduce sits at module scope, so `resolve()` stays a pure return
of literals.

**Four of §12's criteria are not testable at a stub, and the docblock says so rather than leaving it
implied** - the exclusion half of the month boundary (a stub filters nothing), the empty month (one
hardcoded month cannot be empty), the session refusal and the own-entries-only rule (both need an
owner and a filter a stub has neither of).

### The guard fired, and it was right

`tests/__tests__/server/graphql/reconcile-stub-resolvers-with-actual.js` asserts that **every
operation in the stub pool also has one in the actual pool**, because renchan builds its
authentication filter map from the *actual* pool alone - so a stub-only operation is served with
**no filter at all**.

`monthlyExpenses`'s actual resolver is checkpoint 6's work. So the stub added here puts the tree in
exactly the state that file exists to report, and it reported it:

```
npm_config_script_shell=bash npm test -- --seeded --maxWorkers=3 tests/__tests__/
Test Suites: 1 failed, 65 passed, 66 total
Tests:       1 failed, 1522 passed, 1523 total
  reconcile-stub-resolvers-with-actual > ... > audience: staff
  Expected ArrayContaining [... monthlyExpenses ...]   Received [... without it ...]
```

**The two lists differ by `monthlyExpenses` and nothing else.**

**This is not a false positive and it was not treated as one.** `#expense-entry` never met it
because the guard was written at *its* checkpoint 8, once both pools were already complete - so
**this guard has never before run against a mid-feature tree**, and `#monthly-summary` is the first
feature to reach checkpoint 4 with it installed.

### Left red, and why that is the honest option rather than the lazy one

Four alternatives were considered and each is worse:

| option | why not |
|---|---|
| exempt `monthlyExpenses` in the test | **precisely the edit the test exists to make look like a security change** |
| land the actual resolver here | empties checkpoint 4 of its purpose - a stub exists to give the frontend an endpoint *before* real logic |
| a placeholder actual returning literals | **strictly worse than a stub**: it is filtered, so it *looks* authentic, and checkpoint 6 must remember to gut it |
| run the test only at the acceptance gate | relocates the cost rather than paying it, and leaves the window unchecked anyway |

**The red stands because it is the only option where the tree's actual state and the suite's verdict
agree.**

**Three things bound it, and together they are why this is tolerable rather than merely tolerated:**

1. **It closes at checkpoint 6**, which is the next checkpoint that writes a resolver.
2. **It never reaches `release/1.0.0`.** `feature/monthly-summary` merges only once checkpoint 9
   passes, and checkpoint 6 is before that. **The unfiltered operation exists on an unmerged branch
   and nowhere else.**
3. **While it is open, every checkpoint verifies the failing set is *exactly* this one case.** *"1
   failed, and it is this one"* is a checkable claim; *"some tests fail"* is not - and the danger of
   an expected red is that a genuine regression hides inside it.

### The guard's own docblock had gone false, and was corrected

It said *"Every operation of every audience currently has both an actual and a stub, so the actual
always wins and everything is filtered"*, under the heading *"It is not live today, and that is the
point"*. **Both became false one commit earlier.**

Corrected in place, **comment only - every changed line begins with ` *`, and the assertion, the
cases and the sentinels are untouched.** Same reasoning as `paginationConstants.cjs` earlier today:
a docblock asserting a state the tree has left is a claim a later reader relies on, written by
somebody who was right at the time. It now records that it fired, why that was correct, and why the
red was left standing.

Raised as **Q62**, because every future feature hits this at its own checkpoint 4 and no checkpoint
currently owns the window.

### Verification

```
npm_config_script_shell=bash npm run lint                                          clean
npm_config_script_shell=bash npm test -- --seeded --maxWorkers=3 tests/__tests__/  66 suites: 65 passed, 1 FAILED
                                                                                  1523 tests: 1522 passed, 1 FAILED
npm_config_script_shell=bash npm test -- --seeded --maxWorkers=3 tests/_orders/     8 suites / 316 tests, all green
```

**The one failure, named in full, and it is the only one:**

```
tests/__tests__/server/graphql/reconcile-stub-resolvers-with-actual.js
  reconcile-stub-resolvers-with-actual
    > to leave no stub operation without an actual counterpart
      > audience: staff
```

**EXPECTED. This is Q62's window, and it closes at checkpoint 6.** Recorded this way on purpose: *a
red suite whose cause is a known structural window is a different claim from a red suite*, and the
difference should be legible here without anybody first going to read Q62.

**Everything that makes it a bounded claim rather than an excuse is checkable from this block
alone** - the failing set is one named test, the cause is one named operation whose actual resolver
is one named checkpoint away, and `_orders` is untouched and fully green.

Every figure re-run by the main session rather than taken from the unit's report.

## Checkpoint 5 - one module, and the first brief this feature that owed no correction

**One module, not three.** The catalog check ran first and once, as this checkpoint's own clause
requires, and what it settled was mostly what *not* to write.

### The catalog: 33 tracked packages, and three verdicts

| job | verdict |
|---|---|
| (year, month) → a month's date range | **NOTHING MATCHES**, and **there is no date library in this repository at all** |
| summing an integer field | **NOTHING MATCHES, and that is the right answer** |
| validating `year` / `month` | **ADOPT `mentsu-value-inspector`** - already a declared dependency and already the pattern |

**The declines are recorded because "not used" and "not considered" look identical six months later.**
`mentsu-search-condition` (a saved list-view filter contract whose own README says it generates no
query); `renchan-funnel` (its `TodayValueSuite` is the closest thing in the catalog to date
arithmetic, but it offsets by days only and drags a Kafka-shaped dependency in for one string);
`mentsu-validation-rules` (owns a `BetweenConditionSuite`, so it had to be named rather than skipped
- but it is a DB-stored, user-configurable rule-tree engine, the opposite shape to a fixed GraphQL
input); `mentsu-schema`'s `DateonlyScalar` (a predicate, not a generator).

**The money question was asked and answered in the right direction.** `bignumber.js` reaches this
repository only transitively, and the catalog's only BigNumber surfaces are *converters*, not
arithmetic. §9.3 types `amount` as `int`, §4 puts a second currency permanently out of scope, and
`Number.MAX_SAFE_INTEGER` is ~9x10^15 yen against §7's few hundred rows. **Reaching for precision
machinery because a field is called a total would be buying it for a problem that does not exist.**

### The module: `CalendarMonthRangeBuilder`

A new class beside `CalendarDateInspector`, **not a method on it** - that class's concept is one
calendar date and its verdicts, and its `create()` requires a `calendarDate`, so reaching a method
through it would mean inventing a date the method ignores. Add a class, do not widen one.

**Why the class exists at all** is §7 rather than taste. One member of staff's month must stay a
single indexed read against §9.3's composite `(staff_member_id, spent_on)`. The obvious
`WHERE YEAR(spent_on) = ? AND MONTH(spent_on) = ?` wraps the indexed column in a function, which
makes the predicate non-sargable - **and SQLite does not have those functions at all** (it needs
`strftime`) while MariaDB does. A range over the bare column is the one predicate both dialects
answer from the index. **The spec forces the shape.**

### My brief was wrong a fifth time, and this one changed the design

**I said the range should be read in `CALENDAR.TIMEZONE`. It must take no timezone at all.**

`spentOn` is a `DATEONLY` - a calendar date with no instant and no zone - and **the first and last
day of June 2026 are the same two strings in every timezone on earth**. The zone was already spent
upstream at `#expense-entry`, when a date was chosen from an instant. Threading one through here
would add a property that changes no output and imply a zone-dependence the data model does not
have.

**Where the zone still decides something is *which* month is the current one** (§12.2), and
`CalendarDateInspector#buildTodayCalendarDate()` already answers that, in the right place.

**This does not contradict checkpoint 1's note that §6 already covers the timezone** - and the
distinction is worth keeping, because the two are easy to collapse. §6's `Asia/Tokyo` governs
**deriving a date from an instant**, which is the recording side and is where an expense at 23:00 on
the 31st lands in one month or another. **Reading a month somebody named explicitly needs no zone.**
The docblock says so in the file, because a reader who knows §6 will otherwise wonder why the class
ignores it.

### The unit added two cases I did not ask for, and they are the ones that matter

**The century rules: `2000-02` is 29 days and `1900-02` is 28.** A hand-written `% 4` gets 1900
wrong; `Date.UTC` does not. **Those cases prove the technique rather than the result** - every other
February case would pass against a naive implementation too.

Verified independently by the main session, eight probes through the real module:

```
NODE_ENV=development node <probe>
2026-06 -> 2026-06-01 .. 2026-06-30      2000-02 -> 2000-02-01 .. 2000-02-29
2024-02 -> 2024-02-01 .. 2024-02-29      1900-02 -> 1900-02-01 .. 1900-02-28
2026-02 -> 2026-02-01 .. 2026-02-28      2026-12 -> 2026-12-01 .. 2026-12-31
```

### The exit condition is "they are there", and that was checked rather than assumed

This checkpoint's clause says to **list what checkpoint 6 will import and confirm each one
resolves**, and that it is the main session's job because a unit sees only its own module. Six
imports, all resolved in one process: `CalendarMonthRangeBuilder`, `CalendarDateInspector`,
`Expense`, `IntegerValueInspector`, `ValueInspector`, `BaseInputValidator`.

**The probe failed on its first run with `no NODE_ENV`** - the model pulls in `app/globals/env.js` -
which is worth recording as the shape of the check rather than a fault: an import probe that does
not construct the environment tests less than it appears to.

### No seeder, and that was established rather than assumed

The `expenses` development seeder already covers **June 2026 with both boundary days for one member
of staff** (`2026-06-01`, `2026-06-30`) **and a June row belonging to a second** (`10110002`,
`2026-06-18`). So §12's boundary criterion and its another-member-of-staff criterion are both
reachable from fixtures that already exist.

**What is absent is a row on the day before and the day after a month**, which §12's criterion names
explicitly. That is checkpoint 6's own test fixture rather than a seeder change: editing a shared
seeder would move counts that `#expense-entry`'s suites assert absolutely, and `#expense-entry`
already set the pattern of a test creating its own member of staff. **Row-id prefix `103` was
allocated for exactly that**, ascending from `102` - the only rule the bank has, and one written
down in no skill, only in `#expense-entry`'s record where it was noted as undocumented.

### Two notes handed forward rather than acted on

- **The folder now reads inconsistently on one verb.** `hoc-naming` gives `generate~` to a primitive
  and `build~` to a temporary object. The new class follows it: `generateFirstCalendarDate()` returns
  a string, `buildCalendarDateRange()` returns the pair. **The sibling's `buildTodayCalendarDate()`
  returns a string under `build~`**, against the table. Costed rather than guessed: **11 call sites
  across 2 files, both the class and its own test - no resolver calls it**, so a rename is a cheap
  `retake/`. **Not done here**, because renaming another feature's method to satisfy a style table is
  how a checkpoint becomes an unbounded tidy-up.
- **Two terms are glossary-worthy** and were reported rather than added by the unit: *calendar month
  range*, and *first / last calendar date*.

### Verification

```
npm_config_script_shell=bash npm run lint                                          clean
npm_config_script_shell=bash npm test -- --seeded --maxWorkers=3 tests/__tests__/  66 passed, 1 failed / 1575 passed, 1 failed
npm_config_script_shell=bash npm test -- --seeded --maxWorkers=3 tests/_orders/     8 suites / 316 tests, green
```

`__tests__` went 65 → 67 suites and 1523 → 1576 tests, the delta being the 53 new cases.

**The failing set is still exactly `reconcile-stub-resolvers-with-actual > ... > audience: staff`
and nothing joined it** - which is the check Q62 commits every checkpoint in this window to making,
and the reason it is a checkable claim rather than a tolerated red.

## Checkpoint 6 - the real operation, and the window closed itself

**Both suites fully green.** `tests/__tests__/` **1693 / 1693**, `tests/_orders/` **326 / 326**, lint
clean - every figure re-run by the main session.

### Q62's window is shut, and that is this checkpoint's own exit evidence

`reconcile-stub-resolvers-with-actual` had been red since checkpoint 4, by exactly one case, because
`monthlyExpenses` was in the stub pool and not the actual one. **It went green on its own the moment
this resolver landed, and the test file was never touched** - `git status` reports it unmodified.

**That is the shape a guard should have.** Nobody edited it to accept the mid-feature state, and
nobody had to remember to re-enable it. It reported a true thing, stayed red while the thing was
true, and stopped when it stopped being true.

### The total is summed over the rows the operation returns

Never by a second `SUM()`. §12.1's stated reason for one operation is that the two can never
disagree, and §12's first criterion is a statement about *those* rows - **a second query is a second
observation, and it can see a different set.** An empty month sums to `0` by the reduction's seed
rather than by a branch, which is why criterion 3 needs no special case.

The month reaches the database as a range over the bare `spentOn` column, inclusive at both ends, so
§9.3's composite index serves it. **Another member of staff's rows are outside the query rather than
filtered out of its result**, so they can reach neither the entries nor the total.

### What the validator does *not* do is the deliberate part

Checkpoint 1 proposed no spec clause bounding `year` or `month`, so the line between *malformed
input* and *invented policy* had to be walked rather than assumed.

| | |
|---|---|
| **refused** | `month` outside 1-12; `year` outside 1-9999 |
| **deliberately not refused** | a minimum year - §7's seven-year retention says how long a row is **kept**, not which months may be **asked for**, and somebody reading 1970 and being told it is empty has been told the truth |
| **deliberately not refused** | a maximum year, or "not in the future" - §11 refuses an expense **dated** after today, which is a rule about **writing** a row. Refusing a future month would also make §12.2's month navigation refuse where it should show an empty month |

**The year bound is the date format's constraint rather than a business rule, and it was measured
rather than argued:**

```
node -e   an unbounded year, through the same padding the builder uses
  -5     ->  00-5-06-01
  12345  ->  12345-06-01
```

Those strings are compared against a `DATEONLY` column and **come back as a successful empty read**.
**The bound turns a plausible empty month into a stated refusal** - Q58's family answered at the
input rather than discovered in the output. It moves only if `spentOn` stops being `YYYY-MM-DD`,
never because somebody decides which years a member of staff may read.

### A checkpoint 3 miss, closed here rather than left

§6 states the entry order in **one sentence for two screens**. That clause was a **private module
constant** of `ExpensesQueryResolver`, and this checkpoint was about to write a second copy.

**Two private copies can drift, and a drift reintroduces exactly the defect Q61 was raised to fix** -
one screen breaking the tie while the other does not is that same disagreement by another route. So
the clause is declared once and imported by both.

**It belonged to checkpoint 3**, whose own clause says a constant two operations both add to is that
checkpoint's shared file. **Missed there, closed here, recorded rather than quietly folded in.**

`ExpensesQueryResolver`'s docblock was **rewritten, not deleted**. It said the constant was declared
once *"so that the clause this operation owes is a fact of the file rather than a literal buried in a
method"* - **a sentence describing a file-local constant, which had stopped being true.** It now says
the clause is no longer this file's own, names where it went, and says what remains this file's own:
answering for it.

**The sharing is load-bearing rather than tidy, and the mutation check proves it**: deleting the
second order key fails **four tests across both resolvers**, not only the one that declares it.

### Mutation checks - five, each applied, measured and reverted

**This is the project's recurring defect and the reason the checks are run**: a test that passes
under broken and fixed code alike. An existing test once asserted a request offset and never what
the screen showed, and passed over a real removal bug.

| mutation | caught by |
|---|---|
| drop the owner from the `where` | **14 tests** |
| drop the second order key | **4, across both resolvers** |
| widen the range a day at each end | **12 in `__tests__`, plus the 2 in `_orders` that are the only ones catching the upper end** |
| slice an entry off the total | **9 tests** |
| move the session guard after input validation | **2** - and these are the only ones pinning *"before it reads anything"*, which a thrown code alone does not state |

**The third row is the one worth keeping.** Twelve tests catch a widened range and **none of them
catches it at the upper end** - only the two `_orders` cases do. A checkpoint that had settled for
`__tests__` alone would have had twelve green assertions and a live off-by-one at the end of every
month.

### Test placement, reasoned rather than guessed

`monthlyExpenses` **writes nothing**, so by the per-method rule its tests are `__tests__` - 117 of
125. The other eight are in `_orders` because **the test's own path writes**: the day-after boundary,
the same-date tie and the recorded/corrected/removed criterion all need rows made in a stated order.

**Splitting them would have produced a test that asserts nothing** - the writes and the assertion
would land in different phases, against different databases.

Fixtures obey all three of `_orders/README.md`'s rules: no explicit `expenses` id (Q38), no
`expense_categories` row created, and `staff_members` ids from **this feature's own prefix `103`**.

### One brief error, and it was mine again

**The test path.** My brief and `hor-query-resolver` both say the test path mirrors the resolver
*without* the `actual/` segment. **The tree keeps it** - every `actual/queries/` and
`actual/mutations/` test does, with one lone exception. The unit followed the tree.

Six briefs, six corrections. **Every one caught by a unit that read the tree rather than complying.**

### Verification

```
npm_config_script_shell=bash npm run lint                                          clean
npm_config_script_shell=bash npm test -- --seeded --maxWorkers=3 tests/__tests__/  69 suites / 1693 tests, ALL GREEN
npm_config_script_shell=bash npm test -- --seeded --maxWorkers=3 tests/_orders/     8 suites /  326 tests, ALL GREEN
```

`--runInBand` was used **only inside the mutation checks**, on a single `_orders` barrel, and never
for a reported figure - said rather than omitted, because a figure measured on a different
invocation from CI's is what cost this project two days at the backend gate.

## Checkpoint 7 - not applicable, and it short-circuits at the first question

**Nothing was written, and nothing was invented to make the checkpoint non-empty.**

### The decision procedure, run rather than recalled

`hor-execution-placement-pattern`'s flow opens with a question this feature answers immediately:

> **1. Is it a write?** If read-only, return it via an API query / GET and you're done.

**`monthlyExpenses` is a query.** It writes nothing, opens no transaction, and calls nothing that
does. So it stops at step 1, and **step 4 - what triggers the Worker - is never reached.** The
skill states the same thing a second way: *"Read-only processing is out of scope: as a rule it just
returns synchronously via the API and is not turned into a Worker."*

**This feature declares exactly one operation** (§12.1), so that is the whole of the per-operation
walk rather than a sample of it.

**The one thing that could have argued the other way is the aggregation**, and the skill's own
heavy/light rule answers it: heavy means *"external I/O, AI calls, large record counts, or file
generation"*. §7 bounds this at **a few hundred rows** and requires it **under one second**, and the
sum is over rows already in hand. It is the skill's *"short aggregations that complete within a
single request"*, which is the API side by name.

### The spec says it outright, which is better than this walk concluding it

§8, verbatim:

> **Redis is not declared, because this version runs no background job.** Every write finishes
> inside its own request, and nothing here leaves the process.

**A walk that concludes what the spec already states is a walk that confirms rather than decides**,
and that is the stronger position to be in - the same one `#expense-entry`'s checkpoint 7 reached.

**And §4 forecloses the one candidate somebody might reach for.** Exporting a month as CSV is *"file
generation"*, which is the heavy side by name - but it is **1.2.0, out of scope**, and §4's seam
says the export will *call* this operation rather than re-derive it. **So the heavy thing this
feature might one day acquire is a caller of it, not a rewrite of it.**

### One finding, and it is unchanged rather than new

**`ioredis` is still a declared dependency with zero importers.** Measured again rather than carried
from `#expense-entry`'s record:

```
grep -n ioredis expense-note-backend/package.json       ->  "ioredis": "^5.8.0"
grep -rn ioredis app/ server/ sequelize/                ->  no match
```

**So `package.json` alone still suggests a job facility that does not exist**, against a §8 that says
in writing there is none. It is restated rather than omitted because it is still true and this is
the checkpoint that owns the question - a caveat that still applies is said, not dropped because it
was said before.

## Checkpoint 8 - the audit, and a check nobody can prove was ever run

**0 HIGH, 5 MEDIUM, 1 INFO. No finding originates in this feature's change set.** Every file the
feature adds or edits passed checks 1, 2, 5, 6, 7, 20, 21 and 22 on its own terms.

Run read-only in a **verifier** agent, as the checkpoint's own clause requires - the audit finds, it
does not fix.

### The digest did not exist, so it was taken first

`hor-security-audit` is equipped and **had no digest** - the fourth instance of Q59's condition.
Taken before the checkpoint ran, pinned `hora-skills-ort-renchan 0.1.0`, verified against the
installed package rather than assumed. 506 lines at a size ratio of 0.79, deliberately high: the
source is **34 finding clauses, 18 pass clauses and a set of fixed thresholds**, and compressing
further would have cost criteria rather than words.

### A scope arbitration, settled by reading rather than by preference

**The skill's own frontmatter scopes it repo-wide** and redirects diff-only reviews to
`/security-review`. **Checkpoint 8 overrides that deliberately**, in writing:

> Run it against this feature's changes, not the whole repository ... Scoping it here keeps the
> finding list attributable to the work that just happened.

And `/security-review` **is not equipped in this project**, so it was never an alternative. Recorded
because the disagreement is real and a later reader meeting the frontmatter deserves the resolution
rather than the puzzle.

**The change set was read as `release/1.0.0..feature/monthly-summary`, and that needed a
justification** the clause does not supply. The clause says to read the working tree *because a
commit range would be empty before the gate*. **Here it is not empty**: every checkpoint's work is
already committed on `feature/monthly-summary`, which has not merged. The backend working tree is
clean, so the two views agree and the range is the complete change set. **19 files, +6388 / -49.**

### The finding that matters: introspection, and why it took four features to surface

**GraphQL introspection is enabled in every environment, including production.** `graphql-http` does
not disable it by default; only `NoSchemaIntrospectionCustomRule` in `validationRules` does, and a
grep for either name returns **two hits, both inside one comment** explaining that an engine cannot
reach `validationRules` at all.

**Two details corroborate that it is expected to work today**, which is what makes it a decision
nobody took rather than a setting somebody missed: the body-size cap is sized to leave room for *"the
introspection query a schema-aware client sends"*, and the depth limit exempts meta-fields.

**Reconnaissance, not access** - the schema's shape is enumerable before authenticating while the
filter still refuses the operations. **This feature enlarges what is disclosed by two types; it did
not create the condition.**

**And it is recorded nowhere.** `grep -rni "introspect"` over every question and every acceptance
record returns nothing, so it was either never checked or judged out of scope without being written
down - **and the records cannot distinguish those two.** That is the finding under the finding:

> **A check that was run and passed, and a check that was never run, look identical in a record that
> only lists findings.**

**Q58's family in the shape of an absence rather than a plausible value.** It is the argument for an
audit that records a **verdict per check** rather than only its findings - which this run did, which
is why a three-feature-old gap surfaced at all. Raised as **Q63**.

### The finding this feature genuinely widened

**Q55 anticipated `#monthly-summary` by name and got the direction right. It also contained a
mitigation this feature has since holed.**

Q55 argued the read path was not urgent partly because *"each page is capped at
`PAGINATION.MAXIMUM_LIMIT` rows"*. That held while `expenses` was the only read. **`monthlyExpenses`
consults no cap at all** - §12.1 declares no pagination input - so it is **the product's first read
with no enforced row ceiling**.

What bounds it instead is §7's prose, *"at most a few hundred rows"*: **an assertion about data, not
an enforced limit.** And the shape limit's arithmetic no longer closes either - its 10 root
selections were measured against `expenses` at 100 rows per alias, while ten aliased
`monthlyExpenses` calls read ten whole months bounded by nothing the code enforces.

**Class unchanged, magnitude changed**, so Q55 was amended rather than a new entry opened.

### The other four, verified still exactly as recorded rather than assumed

Q16 three times (no datastore TLS, hardcoded credentials in `sequelize/config.cjs`, `.env.live`
tracked under a bare `.env` ignore) and Q54 once (the boilerplate audiences' `origin: '*'` and 10mb
body). **`credentials: true` is not set on either, so check 10's HIGH combination does not apply** -
a caveat stated rather than omitted.

**A pre-existing accepted finding restated as new inflates the list; a new finding dismissed as
pre-existing hides it.** Each was settled by evidence - the recorded question's own line numbers
against the current file, and `git diff --name-only` showing the file is not in this change set.

### What the audit could not answer at this scope, named rather than implied

The skill's own rule 4 is *"a gap must never look like 'not applicable'"*, and it applies to scoping
too. **Six checks were partial**: install hooks (the dependency tree's own scripts were not audited),
ports (there is **no production deployment manifest in this repository**), datastore TLS and rate
limiting (an edge proxy in front is outside the repository), seeder secrets (**this feature touches
no seeder, so N/A for the change set - but no repo-wide sweep was done, which is a gap rather than an
absence of subject**), advisories (`npm audit --omit=dev` only: **5 moderate, 0 high/critical**, one
with no fix available), and error leakage (renchan's production masking was not re-derived; Q53
covers it).

### Q62's window confirmed shut, independently

The verifier ran the guard read-only: **6 passed, 6 total**. A second instrument agreeing with
checkpoint 6's own measurement, which is the standing practice after Q58.

### My brief was wrong a seventh time

I listed `constants/paginationConstants.cjs` in the change set. **It is not there** - the file is
`constants/expenseEntryOrderConstants.cjs`, and `paginationConstants.cjs` is the pre-existing file
holding the cap `monthlyExpenses` deliberately does not consult. Confirmed: `git diff --name-only`
over the range matches it **zero** times, across 19 files.

Also qualified rather than corrected: *"the working tree is clean"* is true, but only because
`expense-note-backend` is **its own repository**. The outer repository is not clean - it carries this
record and the new digest. The conclusion held; the reasoning needed the extra step.

## Checkpoint 9 - both use cases walked as real calls, not read as code

**Driven through the built schema with a real access token on a real header**, using the same
`GraphqlSchemaBuilder` the running server uses and a real `StaffGraphqlContext` per request. In
process rather than over a socket, because `server/index.js` cannot boot on this machine (**Q24** -
the socket is unreached rather than unreachable).

**Nothing was stubbed.** The schema, the resolvers, the validator, the session clerk, the encipher
and the database all ran for real. The member of staff is the seeded `10110001`, signed in with the
plaintext password recorded against the digest in the seeder - no row was created by hand and **no
id from this feature's prefix was spent.**

### Use case 1 - filing a claim at the end of the month

> *picks that month, reads the total, and writes it on the claim form*

```
signIn                                -> staffMemberId 10110001, access token minted
monthlyExpenses(year: 2026, month: 6) -> totalAmount 3451
                                         2026-06-30   1      supplies   "one envelope, bought singly"
                                         2026-06-01   3450   supplies   "notebooks and pens ..."
```

**Five things are true in that one answer, and each is an acceptance criterion rather than a
detail:**

| | |
|---|---|
| **3450 + 1 = 3451** | criterion 1 - the total **is** the sum of the entries shown, in the same answer |
| **the 1st and the 30th are both present** | criterion 2's inclusion half, at both ends of the month |
| **`06-30` comes before `06-01`** | §6's entry order - newest `spentOn` first |
| **`10110002`'s `2026-06-18` (1860) is absent, and 1860 is not in the total** | criterion 5. The total would be **5311** if the scoping leaked, so the number itself is the evidence |
| **`status: "recorded"` is exposed on every row** | §4's approval seam, read through the field rather than assumed |

**The category came back eager-loaded**, so a month is a bounded number of queries rather than one
per row.

### Use case 2 - checking last month against a card statement

> *switches to the previous month and reads its entries **without leaving the screen***

```
monthlyExpenses(year: 2026, month: 7) -> totalAmount 2100    (1200 + 640 + 260)
monthlyExpenses(year: 2026, month: 5) -> totalAmount 0, expenses []
```

**Both on the same access token as use case 1, with no second sign-in and no re-authentication.**
That is the half of the use case a code reading cannot establish: *"without leaving the screen"* is a
claim about the session surviving the navigation, and the only way to show it is to navigate.

**And May is criterion 3 answered on the way past** - a month holding nothing returns `totalAmount:
0` with an empty list, not an error and not a null.

### The two refusals, as a caller actually meets them

```
no access token at all  ->  data: null   errors: ["102.X000.001"]
year 2026, month 13     ->                errors: ["203.Q004.002"]
```

**`102.X000.001` is the engine's own `Unauthenticated` code, not the resolver's.** That is criterion
6 shown rather than argued: the refusal comes from the filter, **before `resolve()` is entered**, so
nothing was read. A code from this resolver's own hash would have proved the opposite.

**`203.Q004.002` is `InvalidMonth`** - the validator refusing a month that is not one, rather than
the operation cheerfully reporting that the thirteenth month of 2026 is empty.

### Nothing fell short, and nothing was added on the way past

**No use case needed an operation §12.1 does not declare.** Nothing was sent back to checkpoint 3,
and **no field was added while walking** - which the checkpoint warns against by name, because a
frontend in another repository is already building against this contract.

**The walkthrough was deleted afterwards and the tree verified clean.** It was a temporary file; what
it established is written here, and what must keep holding is held by the 125 tests checkpoint 6 left
behind.

**One mechanical note worth keeping**, since it cost two attempts: a file placed under
`tests/_orders/` with a plain name **does not run**. `jest.config.js` declares no `testMatch`, so
jest's defaults apply and only `*.test.js` or a path under `__tests__/` matches - which is exactly
why `_orders` suites are pulled in by a `_.test.js` barrel. The first run reported **"No tests found,
exiting with code 0"**, which is a pass-shaped answer to a question nobody asked. **Q58 again**, and
caught only because a walk that prints nothing is obviously wrong.

### Verification at the gate

```
npm_config_script_shell=bash npm run lint                                          clean
npm_config_script_shell=bash npm test -- --seeded --maxWorkers=3 tests/__tests__/  69 suites / 1693 tests
npm_config_script_shell=bash npm test -- --seeded --maxWorkers=3 tests/_orders/     8 suites /  326 tests
```

**Both green, and `--maxWorkers=3` is exactly what CI runs.** Backend working tree clean.

## Frontend gate
- [x] 10. Open the frontend  <!-- skills: hof-nuxt, hof-furo-env; digests REUSED at hora-skills-ort-furo 0.1.0, none taken. Route /monthly-expenses, named for what is on screen rather than the feature id, with no alias because / is already claimed. Guarded BY EXISTING via the global gateway middleware rather than a second mechanism. Route confirmed in the BUILT bundle, not inferred from the folder. No env variable added -- all three furo-env files already point at the staff endpoint, verified against the backend's own port and graphqlEndpoint. EXIT CONDITION MET IN HALF AND THE UNIT SAID SO: 'reachable' is not met, and find-unreachable-screens was RUN rather than predicted -- /monthly-expenses is a TRUE positive where /expenses is the known false one caused by a page-level alias. No link added on the unit's initiative, correctly. AND IT FOUND A TRAP IN .hora/tree/: that document claimed Nuxt auto-registration is configured, when nuxt.config turns auto-import off TWICE (components.dirs empty, imports.autoImport false) -- laid directly in the path of checkpoints 12 and 15, where a page's components would silently resolve to nothing. Verified and corrected -->
- [x] 11. Reconfirm UI/UX and the use cases  <!-- main session, in conversation. The third pass, and the one that found the gap: checkpoint 2 asked whether the spec supports the use cases and 9 whether the API does; this asks whether a PERSON CAN DO THEM ON A SCREEN, and the answer was no for want of one link. The context file still said this screen 'does not exist. Do not link to it, do not stub it' -- and it is read automatically by the UI generator at 12 and 15 and the auditor at 18, so checkpoint 12 would have read that about the very screen it was told to build. DECIDED: the two signed-in screens link to each other, one link each, in the page's own header -- NOT chrome in the shared layout, which would put links to guarded screens in front of somebody with no session on the one screen reachable without one. Four rules added, two of them to stop a reasonable instinct: the month is screen state never a route segment; NEVER disable a future month (section 11 refuses a DATED expense, a rule about writing, and a future month is truthfully empty -- Q56's date-field lesson does NOT transfer); never sort (measured: no client-side sort exists anywhere today); an empty month says so and still shows a zero total. Nothing went back to checkpoint 2 -->
- [x] 12. Component design  <!-- skills: all twenty hof-cp-* matched EXHAUSTIVELY, five adopted and fifteen declined with a reason each; digests reused at hora-skills-ort-furo 0.1.0. NOTHING NEW BUILT. The month control is FOUR controls because section 12's two use cases are two gestures on one piece of state, not a choice between two designs. FuroDatePicker rejected ON MEASUREMENT: there is no month mode -- granularity 'day' is hard-coded after the parcel spread and is not a parcel key. Rule 28 drove a real decision: the year window is recomputed around the SELECTED year, because one anchored on today would eventually select a year absent from its own options. MY OWN CHECKPOINT 11 ERROR FOUND HERE: the four rules I added collided with existing 21-26, so four numbers were used twice and checkpoint 18 cites them by number; renumbered 27-30. Rule 26 also still called the page size 'not yet user-confirmed', the THIRD document this feature has caught asserting a superseded state. Q64 raised: @iconify-json is absent, the bundle builds a remote collection against api.iconify.design, and furo draws ph: icons unasked -- so every dropdown caret is a third-party dependency, hidden by a context line saying no icon set is installed -->
- [x] 13. The frontend modules the implementation needs  <!-- skills: hof-error-handling, hof-modules; digests reused at hora-skills-ort-furo 0.1.0. HALF TWO APPLICABLE: three Q004 codes mapped, read from the backend's own errorCodeHash. 204.Q004.001 takes the session-guard family's sentence and the non-disclosure pairing DELIBERATELY does not apply -- that pairing exists because one code covers three outcomes about an entry somebody owns, while this covers one outcome about the caller's own session and has nothing to disclose. Both 203.Q004 codes say the same thing and neither blames the reader, because the screen's controls cannot produce an invalid year or month. HALF ONE NOT APPLICABLE, decided rather than assumed: all three candidates have exactly one consumer today. Yen formatting is the honest candidate and was HANDED FORWARD as an explicit choice rather than manufactured. MY BRIEF WAS WRONG AN EIGHTH TIME AND WOULD HAVE BROKEN THE BUILD -- I specified ERROR_LOCALE_HASH and i18n locale paths and this repository has neither -->
- [x] 14. API client  <!-- skills: hof-graphql; digest reused at hora-skills-ort-furo 0.1.0. The Payload/Capsule/Launcher trio for the one operation section 12.1 declares. THE CONTRACT IS THE ORACLE and nothing about the expected shape is transcribed: validate() catches a field the contract does not declare, a coverage walk catches one the document omits, and the root field and variable name are pinned so neither can pass vacuously against expenses, which shares the Expense row type. Mutation-checked BOTH directions. The vendored contract was verified sound both ways first -- fingerprint matches AND body byte-identical, 202 lines, zero diff -- because the fingerprint alone answers only whether the AUTHORITY moved, which Q57 says itself. THE STUB WAS ALREADY UNREACHABLE: renchan resolves actual ?? stub and checkpoint 6 landed the actual eight checkpoints earlier, so 'works against the stub' could only be established in process and CHECKPOINT 16 HAS NO STUB TO LEAVE BEHIND. Q65 raised. totalAmount falls back to null not 0, both asserted side by side, because section 12 makes 0 a real answer. I ran 13 and 14 concurrently against one tree and the whole-repo count moved under both units -- Q33's shape on the frontend: fencing directories does not fence a figure -->
- [x] 15. UI  <!-- skills: hof-uiux-forge, every hof-css* , hof-furo-context-patterns, hof-prohibits, hof-layout-margin, hof-animation and the five adopted component skills; digests reused at hora-skills-ort-furo 0.1.0. ALL FOUR STATES built and reachable; find-unreachable-screens went 2 to 0. A zero and an absence are kept apart throughout -- landed is totalAmount !== null, which is why checkpoint 14's capsule falls back to null. The empty state renders INSTEAD OF the table, because the slot would keep four column headers above a colspan, which is the empty table rule 30 forbids. The yen-formatter choice checkpoint 13 handed forward was TAKEN: extracted and /expenses converged in the same change, with no constructor or create() signature altered so the large sibling suite was undisturbed. Two library gaps compensated for in this project's code (a FuroSelect id reaches no DOM element; a loading FuroButton has no accessible name) and one judged rather than patched (--color-link is 3.68:1, under the 4.5:1 floor, so links are drawn in the title foreground with an underline). TWO DOCUMENTS CORRECTED that would have misled checkpoint 18: two component digests asserted this project imports no furo stylesheet when nuxt.config loads furo.css FIRST, and the design document still cited the pre-renumber rule numbers in eight places -->
- [x] 16. Wire the data-fetching logic in  <!-- skills: hof-furo-context-patterns, hof-graphql, hof-error-handling; digests reused at hora-skills-ort-furo 0.1.0. NO MARKUP MOVED and it was PROVED rather than asserted -- the template slice is byte-identical to the committed one, 352 lines, diff empty. One watcher with immediate: true covers the opening read and every month change. A STALE-RESPONSE GUARD, NOT A DISABLE: neither step button gained disabled or loading, because the design rejected that in writing and rule 28 would make a greyed-out next-month button a finding. TWO OF SECTION 12'S SIX CRITERIA HAVE NO FRONTEND SURFACE and are not claimed -- the month boundary is not drawable from a screen that names a month and never a date range, and another member of staff's rows are backend scoping this screen cannot even ask for. Criterion 1 is backed as the frontend's half only: the total shown is the figure the operation returned, never recomputed. Checkpoint 16's own 'this is where the stub is left behind' is VACUOUS here (Q65) so it was not claimed; the checkable half was done by blob hash. Q66 raised: the launcher's per-request hooks mean the first answer back lowers the wait for both, so the screen briefly says a month is empty while it is still loading -- reported rather than fixed, because both cheap fixes are wrong and a correct one needs a request token nothing specifies. THE FIRST OF MY BRIEFS ON THIS FEATURE THAT WAS RIGHT -->
- [x] 17. Local test environment  <!-- NOT APPLICABLE by the clause's own terms: one already exists and this feature added no service, no role and no seed data it needs -- checkpoint 5 established that the expenses seeder already covers June 2026 with both boundary days and a second member of staff. BUT THE ENVIRONMENT'S DESCRIBED STATE WAS WRONG AND WAS NOT ACCEPTED: it was reported as still up with live seeded, and docker info said the daemon was NOT answering while the mariadb container read Exited (255) 20 minutes ago. Brought up, then the DATABASE was asked rather than the start command's exit code -- because a start command returning 0 having started nothing is one of Q58's own instances -- and it answered, with live holding 14 expenses and 13 staff members. Two further corrections: port 3306 is FREE locally (netstat showed OUTBOUND connections to a remote host, not a listener), and the container had been down about twenty minutes. Three times in this stretch my own exit code read 0 over a failing command because it came from a piped head; each was re-run properly. The application itself still does not start on this machine -- Q24 plus Windows binaries under WSL -- which is a platform limitation recorded at #expense-entry's acceptance, not a property of this feature -->

## Checkpoint 10 - the route opened, and a document that would have wrecked checkpoint 12

**`/monthly-expenses`**, named for what is on screen rather than for the feature id. The sibling is
`/expenses` - the kebab-case plural of the collection it reads, which is also the query it opens on -
and §12.1's query is `monthlyExpenses`. **`/monthly-summary` would have been the only route in this
application named after a feature id.**

A route of its own rather than `/expenses/summary`: the two read different operations, neither is
subordinate, and a nested path would sit in the way of a future `/expenses/[id]`. **No alias**, since
`/` is already claimed and two records claiming one path resolve arbitrarily.

**Guarded by existing.** `middleware/000.gateway.global.js` guards every path but `/sign-in` and
steps aside only for `$furo: { skipFilter: true }`; `FuroMeta#get:skipFilter` reads
`this.furo.skipFilter ?? false`, so a page setting no such meta is guarded without declaring
anything. One mechanism, not a second.

**The route was verified in the built output rather than inferred from the folder** - the production
bundle carries `"monthly-expenses"` and `"/monthly-expenses"`, and no `.js` sibling leaked into the
route table, so `nuxt.config.js`'s `pages:extend` hook handled the new folder too.

**No environment variable was added, and that was checked rather than assumed.** All three
`.furo-env` files already point `ENDPOINT_URL` at the staff endpoint, verified against the backend's
own listening port and `graphqlEndpoint` rather than trusted. `monthlyExpenses` is one more operation
on the same endpoint.

### The exit condition was met in half, and the unit said so rather than claiming it whole

**"Reachable" is not met**, and the unit reported that plainly instead of reading the word narrowly.
It ran the script rather than predicting it:

```
node .claude/skills/hof-acceptance-review/scripts/find-unreachable-screens.mjs expense-note-frontend-staff
  /expenses          nothing outside this screen mentions the path   <- known FALSE positive (page-level alias)
  /monthly-expenses  nothing outside this screen mentions the path   <- TRUE positive
```

**The contrast is the whole point.** `/expenses`'s report is the false positive `#expense-entry`'s
acceptance already recorded, caused by an alias the script cannot see. **A second screen with no
alias and nothing linking to it is the real thing.** Closing it is checkpoint 11's decision and 15-16's
work, and the unit declined to add a link on its own initiative, which was right.

### A trap in `.hora/tree/`, found by a unit that read the config instead of the document

`.hora/tree/expense-note-frontend-staff.md` said **"Nuxt's component auto-registration is configured
in `nuxt.config.js`, so a component dropped into `components/` is registered without an import."**

**False, and `nuxt.config.js` turns auto-import off twice**: `components: { dirs: [] }` and
`imports: { autoImport: false }`. Verified by the main session before correcting it.

**It was laid directly in the path of checkpoints 12 and 15**, which build components and read this
document. A page whose components silently resolve to nothing renders empty, and **the config is the
last place somebody would look when the document told them the opposite.** Corrected, with the
correction's own reason written beside it.

**Two more stale documents were reported rather than edited**, correctly: `ai/contexts/uiux-context.md`'s
screen table (checkpoint 11's to own, below) and a comment in `SignInPageContext` still calling `/` an
empty stub, which the alias made false.

## Checkpoint 11 - the third pass, and the decision nobody had taken

**2 asked whether the spec supports the use cases, 9 whether the API does. This asks whether a person
can do them on a screen** - and the answer was **no**, for want of one link.

### What the context file said about this screen

> `— | #monthly-summary (§12.2) | **does not exist.** Do not link to it, do not stub it`

**Checkpoint 10 made that false**, and this file is read automatically by the UI generator at
checkpoints 12 and 15 and the UI auditor at 18. **Checkpoint 12 would have read "does not exist, do
not stub it" about the very screen it was told to build.**

### The decision: two screens, two links, in the pages' own headers

**Not chrome in `layouts/default.vue`.** That layout is shared with `/sign-in`, so chrome there would
put links to guarded screens in front of somebody with no session - **every one of them bouncing
straight back** - on the one screen §10.2 says is reachable without one.

**The link on `/expenses` is a change to a screen `#expense-entry` already shipped**, and it is
planned growth rather than a retake: that screen was complete for its own feature, and what changed
is that there is now somewhere to go.

### Both use cases now have a path, walked through the interface rather than the API

| §12 use case | the path on screen |
|---|---|
| filing a claim at the end of the month | sign in -> `/` -> `/expenses` -> **the new link** -> `/monthly-expenses`, which opens on the current month -> read the total |
| checking last month against a card statement | on `/monthly-expenses`, **move to the previous month without a route change** -> read its entries |

### Four rules added, and two of them exist to stop a reasonable instinct

- **21 - the month is screen state, never a route segment.** §12.2 wants the previous month *without
  leaving the screen*, and its own call table says the operation is called *"on opening, and on
  moving to another month"* - **a re-read, not a re-route.**
- **22 - never disable or refuse a future month.** **The instinct is to grey them out and it is
  wrong.** §11 refuses an expense **dated** after today, a rule about *writing a row*; §12 says
  nothing of the kind about *reading a month*. A future month is truthfully empty with a total of
  zero, which is §12's own third criterion, and refusing one would break this screen's navigation.
  **Q56's date-field lesson does not transfer** - that was about §11's refusal, and this screen has
  none.
- **23 - never sort the entries.** §6 states one clause for both screens and the backend returns it
  from a constant both resolvers share. **Measured: there is no client-side sort anywhere in this
  application** - no `.sort(`, no `sortBy`, no `orderBy` - which is what makes the two lists agree.
  A sort here would re-decide what Q61 settled.
- **24 - an empty month says it is empty and still shows the total**, zero rendered rather than
  hidden.

**Nothing went back to checkpoint 2.** Both use cases have a path; one of them needed a link that did
not exist, which is a gap in the interface rather than in the use case.

## Checkpoint 12 - six regions, nothing new built, and twenty skills named

**Every one of the twenty equipped component skills was named ADOPT or DECLINE with a reason.** Five
adopted - button, control-block, empty-state, select, table. **`FuroPagination` declined on the
record** so its absence is never read as an oversight: §12.1 declares no pagination and
`MonthlyExpensesResult` carries no `pagination` field.

The design lives at `ai/contexts/uiux-context-monthly-summary.md`, which the UI generator at 15 and
the auditor at 18 both pick up through their `uiux-context-<suffix>.md` pattern.

### The month control is four controls, not a choice between two

§12's use cases are not in conflict - **they are two gestures on one piece of state.** A select pair
makes *"switches to the previous month"* an open-scroll-pick, four gestures across a January
boundary; a prev/next pair leaves *"picks that month"* with no pick and puts a month eight back eight
clicks away. Both write the same `{ year, month }`, so there is no second source of truth.

**`FuroDatePicker` was rejected on measurement rather than taste: there is no month mode.**
`granularity: 'day'` is hard-coded on the line *after* the parcel spread, and `granularity` is not a
parcel key - so a parcel value is spread in and then overwritten. A date picker also asks for a
**date**, whose day would be invented and immediately discarded.

**Rule 28 (never disable a future month) drove a real design decision**: the year select's options
are recomputed around the **selected** year rather than around today's, because a window anchored on
today would eventually select a year absent from its own options and blank the trigger.

### Three divergences from the sibling screen, each reasoned

- **The empty state goes outside the table, not in its `#empty` slot.** The slot keeps four column
  headers above a `colspan`, which is *an empty table with a sentence in it* - and both §12's
  criterion and rule 30 say **"rather than an empty table"**.
- **Rows and total are cleared when the month changes, before the request goes out.** `/expenses`
  keeps rows under an overlay so layout never collapses; here the rows belong to the **previous**
  month, and showing them beside a heading that already says October is the same falsehood as a stale
  total.
- **The total has three states, not two** - figure, zero, dash. **A zero after a failed read would
  state a total for a month nobody read.**

### My own error, found by the unit and fixed by me

**The §8 rule numbers I wrote at checkpoint 11 collided.** I appended four rules anchored on rule 20
without checking that the section already ran to **26**, so the block read 19, my 21-24, 20, then the
existing 21-26 - **four numbers used twice**, and checkpoint 18's auditor cites these by number.
Renumbered to **27-30**; no rule's text changed.

**The unit found it and declined to fix it** on the ground that the file is checkpoint 11's. That was
right, and it is why the fix is mine.

**Rule 26 also carried a claim that had gone false** - the page size as *"chosen and not yet
user-confirmed"*, when Q50 was answered and §7 carries it. **Third document this feature has caught
asserting a superseded state**, after `paginationConstants.cjs` and the stub-filter guard. Same shape
every time: **the sentence was right when written, and nothing makes a right-when-written sentence
announce that it has expired.**

### Q64 raised, verified independently

`@nuxt/icon` is configured, **`@iconify-json` is absent**, the generated bundle builds a
`createRemoteCollection` against **`api.iconify.design`**, and furo draws `ph:check`,
`ph:circle-notch`, `ph:caret-down` and more **unasked** inside `FuroSelect` and `FuroTable`. With
`ssr: false` the browser fetches them. **Every dropdown caret in this product is a third-party
availability dependency**, and `/expenses` already does it.

What hid it: `uiux-context.md` §5 says *"Icon set: None installed"*. **The module is installed and
icons are drawn** - what is missing is the local collection.

### One finding nobody asked for, and it is design-load-bearing

**The opening month must be read in `Asia/Tokyo`, not the browser.** Unlike `/expenses`, **nothing
corrects a wrong month here** - `monthlyExpenses` truthfully answers whatever `{ year, month }` it is
given - so a browser outside `Asia/Tokyo` opening at a month boundary lands on the wrong month.
**That is exactly use case 1's moment**, filing at the end of the month.

## Checkpoint 13 - one half applicable, and a brief that would have broken the build

### Half two: three codes mapped

Read from the backend resolver's own `errorCodeHash` rather than from my list: `InvalidYear`
`203.Q004.001`, `InvalidMonth` `203.Q004.002`, `StaffMemberNotFound` `204.Q004.001`.

**`204.Q004.001` takes the session-guard family's sentence, and the non-disclosure pairing
deliberately does not apply.** `204.M005.002`/`204.M006.002` are byte-identical because **one code
covers three outcomes about an entry somebody owns** and §7 requires them indistinguishable. This one
covers **one** outcome, about the caller's **own** session, raised before the month is looked at and
before a row is read - **it has nothing to disclose.** Applying the uninformative sentence would
actively mislead: it would send somebody whose session had lapsed to reload a page about to refuse
them again. Verified: six `StaffMemberNotFound` codes now exist and all six share one sentence.

**Both `203.Q004` codes say the same thing and neither blames the reader.** The month control is two
selects - year clamped 1-9999, twelve months always offered - so **a member of staff cannot type an
invalid year or month**, and a message telling them to enter a valid year would be false in every
situation that can produce the code. **Reload, not retry**: a retry resends the same refused input
and fails identically.

### Half one: not applicable, and nothing invented to make it non-empty

Three candidates, each asked the clause's own question - *used in more than one component or page
**today***? All three answered one.

**The honest candidate is yen formatting**, whose second consumer arrives at checkpoint 15/16.
Making it shared today would mean rewriting a shipped `#expense-entry` file; writing the module
without that rewrite would leave **one eventual user beside an un-migrated duplicate** - worse than
either end state. **Handed forward as an explicit choice rather than left to be decided by default.**

### My brief was wrong an eighth time, and this one would have broken the build

I specified **`ERROR_LOCALE_HASH` and i18n locale paths**. This repository has **neither** - it uses
`ERROR_CODE_HASH` + **`ERROR_MESSAGE_HASH`** with literal English strings, keyed by the code value
directly rather than the digest's two-hop form. **The digest names the i18n fork as an open decision
and says not to guess.** The unit read the file and followed the repository; **had it complied, the
first `t()` call would have failed.**

## Checkpoint 14 - a client whose oracle is the contract, and a stub that was never reachable

### Nothing about the expected shape is transcribed

`validate(buildSchema(contract), parse(document))` catches a field the contract does not declare; a
coverage walk catches a declared field the document **omits**, which `validate()` deliberately
allows; and the root field and variable name are pinned so neither check can pass **vacuously**
against the wrong operation - `expenses` shares the `Expense` row type, so that confusion is real.

**Mutation-checked in both directions rather than trusted.** Adding `pagination { limit }` failed 4
tests across 3 suites; deleting `totalAmount` made the coverage inspector return the missing path.

**The vendored contract was verified sound both ways before being trusted as an oracle** - the
recorded fingerprint `e98a3a6edf510ed8` matches the authority's current hash, **and** the copy's body
is byte-identical, 202 lines each, zero diff. **The fingerprint alone answers only whether the
authority moved, not whether the copy still matches it** - Q57 says so itself, which is why the
second check exists.

### The stub was already unreachable, and my brief had it wrong

Verified in renchan rather than assumed: `actualResolverSchemaHash[it] ?? stubResolverSchemaHash[it]`
- **the actual pool wins**, and checkpoint 6 landed it eight checkpoints before anything could build
against a stub.

**So "works against the stub" could only be established in process** - feeding the stub's own seven
entries and its summed total through the contract's schema, with no socket, no listener and no fetch.
**And checkpoint 16 has no stub to leave behind for this operation.** Raised as **Q65**, which
reframes what the stub is actually worth here: **a schema-accurate fixture written independently of
the resolver**, which is worth more than one the client's own author invented because it cannot drift
toward the client by accident.

### A judgement worth keeping

**`#totalAmount` falls back to `null`, not `0`, and both are asserted side by side** so the
distinction cannot be quietly removed. §12 makes `0` a real answer for an empty month, so a `0`
fallback would make *"not asked yet"* and *"asked, and the month was empty"* **indistinguishable on
screen**.

### A process error of mine, and it is Q33's shape on the frontend

**I ran checkpoints 13 and 14 concurrently against one working tree.** Directory fencing kept the
files clean - neither unit touched the other's - but **the whole-repo test count moved under both of
them**, 44 to 46 to 47 to 49 as files landed. Both units noticed, and both isolated their own
attributable figure rather than quoting a number that was not theirs.

**That is the frontend echo of Q33**, where three parallel units on one SQLite file caused a real
failure. Here the harm was smaller and entirely about attribution: **a shared count is not a shared
file, and fencing directories does not fence a figure.**

## Checkpoint 15 - the screen, in all four states, and two documents corrected

**All four states rendered and reachable** - filled, loading, empty, failed. **The three other than
filled are the ones acceptance fails on**, which is why the clause gives all four to one unit rather
than four authors none of whom holds the whole condition.

**`find-unreachable-screens` went 2 to 0.**

### A zero and an absence are kept apart throughout

*Landed* is `totalAmount !== null` - **which is exactly why checkpoint 14's capsule falls back to
`null` rather than `0`.** §12 makes zero a real answer for an empty month, so it cannot also mean *"no
answer yet"*. The total reads a dash while loading and after a failure, and **zero only for a month
that was read and was empty**.

**The empty state renders instead of the table, not in its `#empty` slot** - the slot keeps four
column headers above a `colspan`, which is *an empty table with a sentence in it*, and both §12's
criterion and rule 30 say **"rather than an empty table"**.

### The yen-formatter choice, taken rather than defaulted

Checkpoint 13 handed it forward. **Decision: extract, and converge `/expenses` in the same change.**
The design requires both screens' amount columns to match *field for field*, and two independent
`Intl.NumberFormat` configurations satisfy that **only until somebody edits one**.

**The cost was contained**: no constructor or `create()` signature changed, so the large
`ExpensesPageContext` suite was undisturbed - all 747 baseline tests still pass, with one added
import line.

### Two library gaps compensated for in this project's code, not in `node_modules`

- **An `id` on `FuroSelect` reaches no DOM element** - it spills onto a fragment, so a control
  block's `<label for>` would resolve to nothing **silently**. The id goes on the trigger parcel,
  which is also the only way either select gets an accessible name.
- **A loading `FuroButton` has no accessible name**, so the retry button carries a durable
  `aria-label`.

**And one finding judged rather than patched:** furo's `--color-link` is **3.68:1**, under the 4.5:1
floor. Rather than redefine a library token in a shared file, both links are drawn in the title
foreground with an underline, **so the affordance is not carried by colour alone**. Recorded because
the next screen reaching for `--color-link` meets the same thing.

### Two documents corrected, both of which would have misled checkpoint 18

- **`hof-cp-button` and `hof-cp-control-block` both asserted that this project imports no furo
  stylesheet**, so *"every token resolves to nothing"* and the error text *"inherits the ambient
  colour instead of rendering red"*. **`nuxt.config.js:57` loads `furo.css` first**, with a comment
  saying it must. The premise is false and the conclusions do not follow. Corrected in both, with
  what is still true kept: `variables.css` really does declare nothing, and the app really does
  declare no `@layer` order.
- **The design document still cited the old rule numbers 21-24** in eight places. Renumbered to
  **27-30**. The unit implemented against the section and noticed the two disagreed.

**The unit reported these as "every `hof-cp-*` digest".** Measured: **two**, not twenty - and my own
first sweep found *one*, having matched a different "resolves to nothing" in `hof-cp-select` that is
about `<label for>` and **is still true**. **Two wrong counts and one right one, for the same
sentence.**

### Two deviations from the binding design, both to keep the screens honest with each other

**The table's field names are the derived ones.** `FuroTable` renders `row[field]`, so the design's
nominal `expenseCategory` would print `[object Object]` and `amount` a bare integer - **failing the
very sentence that lists them**, which requires the two screens to match field for field.

**The date column is headed `Date paid`, as `/expenses` heads it.** A column headed differently on
two screens is precisely *two lists of the same rows looking like different things*.

## Checkpoint 16 - the read wired in, and the first brief that was right

**No markup moved, and that was proved rather than asserted**: the template slice of `index.vue` is
**byte-identical** to the committed one, 352 lines each, diff empty. **The whole change is inside
`<script>`** - which is what checkpoint 15's seam existed for.

One watcher on the chosen month with `immediate: true` covers **both** the opening read and every
change. **A stale-response guard, not a disable**: the read snapshots the month and discards an
answer for any other, so three quick Previous clicks land three months back. **Neither step button
gained `disabled` or `loading`** - the design rejected that in writing, and rule 28 would make a
greyed-out next-month button a finding in its own right.

### Two of §12's six criteria have no frontend surface, and are not claimed

| criterion | frontend surface |
|---|---|
| the month boundary | **none.** The screen names a month and never a date range. What is asserted instead is that the variables carry exactly `year` and `month` **and no date** |
| another member of staff's rows | **none.** This screen sends no staff identifier at all and has no way to ask for anybody else's month |

**Criterion 1 is backed as the frontend's half only**: the total shown is the figure the one
operation returned, **never recomputed from the rows** - which is what makes the two unable to
disagree. The arithmetic stays the backend's.

**Criterion 6's refusal reaches a member of staff as a sentence** - *"Your session is no longer
valid. Sign in again."* - and the dotted code never appears.

### Checkpoint 16's own clause is half vacuous here, and the half that is checkable was done

*"This is where the stub is left behind"* is not true for this operation - **the screen was never on
the stub** (Q65). So it was not claimed. *"Confirm the stub is still intact"* **is** checkable and was
done: the working-tree blob hashes identical to the blob at `HEAD`, the last commit to touch it is
still the one that created it, and the backend tree is clean.

### The defect it reported rather than invented a fix for

**The launcher's hooks are per-request, so with two reads in flight the first answer back lowers the
wait for both.** In that window the table says *"No expenses in October 2026"* **while October is
still loading** - a screen briefly asserting a month is empty when it does not yet know.

**Both cheap fixes are wrong**: re-raising the flag on a stale return sticks the spinner on forever
when the current answer arrives first, and disabling the buttons is what the design and rule 28 both
forbid. **A correct fix needs a request token, and nothing specifies one.** Raised as **Q66** - the
shape is shared with `/expenses`, but the *exposure* is this screen's, because stepping months
quickly is the gesture §12.2 names.

### The eighth brief, and the first that was right

**Nothing in it was wrong** - the seam, the launcher's interface, the renchan reading and Q65's
framing all held, each re-verified rather than taken. **And a test caught what I missed anyway**: the
constructor-spy test failed until `create()` forwarded the new client hash.

## Checkpoint 17 - not applicable, with the environment established rather than inherited

**Not applicable by the clause's own terms**: *"one already exists and this feature added no service,
no role and no seed data it needs."* Both halves hold - no service, no role, and **checkpoint 5
established that the `expenses` seeder already covers June 2026 with both boundary days and a second
member of staff**, which is why no fixture was written.

**But the environment's described state was wrong, and I did not accept it.** It was reported as
still up with `live` seeded. Measured:

```
docker info                      -> daemon NOT answering
docker ps -a                     -> expense-note-backend-mariadb-1   Exited (255) 20 minutes ago
```

Brought up, and then - **because a start command returning 0 having started nothing is one of Q58's
own recorded instances** - the database was asked rather than the exit code:

```
docker exec ... mariadb -uroot -ppassword -e "SELECT 1 AS answering"   exit 0  ->  answering 1
live: expenses 14, staff_members 13
```

**Two further corrections to what was believed.** Port 3306 is **free** locally - what `netstat`
showed were *outbound* connections to a remote host, not a listener, which is the same port that
blocked this before. And the container had been down about twenty minutes, so *"inherited rather than
established"* was the right caveat to carry and the wrong state to assume.

**Three times in this stretch my own `$?` read 0 over a failing command**, because the exit came from
a piped `head` rather than from the command. Each was re-run capturing the status properly. **Q58's
family, in the checks meant to establish the environment.**

**The application itself still does not start on this machine** - Q24, plus this tree's Windows
binaries under WSL - which is a platform limitation recorded at `#expense-entry`'s acceptance and not
a property of this feature.

## Acceptance gate
- [x] 18. Acceptance (E2E and unit both)  <!-- /hora-accept, feature-gate form, scoped. VERDICT: passed at static reach, with GATE 4 NOT RUN. Unit suites across every repository in full ON RELEASE, not on a feature branch: backend 69/1693 and 8/326, frontend 50/959, three lints clean, build green -- 2978 tests, no step reusing a recorded result. BOTH ACCEPTANCE SKILLS HAD NO DIGEST, Q59's fourth and final instance; both taken first and each verified against the installed package by DIFFING the equipped copy rather than reading a name. ONE FINDING, Minor, and it is about an INSTRUMENT not the product: find-orphan-template-members reports NOT APPLICABLE against a project that binds a context on every screen, because its pattern requires the VARIABLE to be named context -- blind to this project's idiom across three features, and #expense-entry's acceptance recorded its exit 2 as a limitation done by reading rather than asking whether the not-applicable was TRUE. Redone by reading: 3 screens, 109 bound members, 0 orphans. TWO DOCUMENTS CREATED THAT SHOULD HAVE EXISTED -- the acceptance declaration (phase 1 says create it rather than infer a flow and audit against the guess) and ai/specs/e2e/, which did not exist at all, so #expense-entry's 25 scenarios were derived in its run and never written down and THAT DERIVATION IS GONE. Wrote this flow's 14 and reserved EXP and SIG as namespaces. The skill's own check-scenario-coverage.mjs EXISTS NOWHERE, so the coverage claim is hand-derived and the document says so. AND A CLAIM I WROTE INTO THE ACCEPTANCE RECORD WAS FALSE -- 'all 49 pins consistent', from a grep that could not tell the scoped form from the unscoped; it is 28 and 21. Q58 inside the acceptance record, caught by an outside observation rather than by my own check -->
