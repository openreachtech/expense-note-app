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
- [ ] 3. DB and API schemas
- [ ] 4. Stub API
- [ ] 5. The modules the implementation needs
- [ ] 6. Actual API
- [ ] 7. Worker
- [ ] 8. Security audit
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
