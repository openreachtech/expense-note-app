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
