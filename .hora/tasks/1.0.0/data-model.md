# #data-model  The three tables, and the four seeded categories
<!-- spec: data-model @ sha256:aed4f9e90c4b7b39 -->
<!-- repositories: backend -->

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

Note: this feature has no `<!-- usecases -->` block, and that is correct — a data model has no
      user-facing use case of its own, and the features built on it carry them. Its
      `<!-- acceptance -->` block is what checkpoint 18 reads

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
