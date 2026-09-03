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
- [ ] 1. Draft or confirm the specification
- [ ] 2. Verify the use cases can be met

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
