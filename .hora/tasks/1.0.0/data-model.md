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
- [x] 1. Draft or confirm the specification  <!-- skills: hoc-requirement-definition; digests: none taken — an interactive checkpoint, run by the main session, hands no agent a digest. Matched against hora-skills-ort-core 0.2.0 -->
- [x] 2. Verify the use cases can be met  <!-- skills: hoc-requirement-definition; digests: none taken — interactive, no agent. Matched against hora-skills-ort-core 0.2.0. GAP: the checkpoint delegates to "the shared UI/UX project context", and the only equipped skill covering that is frontend-surface (hof-), out of surface for a backend-only feature. Ran without it -->

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
