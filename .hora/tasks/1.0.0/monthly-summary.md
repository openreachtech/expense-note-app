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
