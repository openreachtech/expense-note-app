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
- [x] 1. Draft or confirm the specification  <!-- interactive, main session; no agent, so no digest. Section 11 read against sections 4, 7, 9.3 and the pinned contract. Two gaps found, each changing behaviour a member of staff can see, both answered by the user: "most recent first" now means `spent_on` (section 9.3's index already pointed there), and a second removal is answered as NOT FOUND, collapsing into the ownership rule section 11 already states. Raised as PR #20 against `specs/` -- a PROPOSAL; the spec on release is unchanged until it merges. A `sort` clause and a same-date tie-break were deliberately NOT proposed -->
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
**pull request #20** against `release/1.0.0`, touching `specs/1.0.0/spec.md` and nothing else. **The
spec on release is unchanged until the user merges it.** The decisions themselves are the user's
answers and are already settled, which is why the feature is not blocked on the paperwork.

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
