# #sign-in  Signing in, renewing, signing out, and asking who is signed in
<!-- spec: sign-in @ sha256:e8d736e09febcf1f -->
<!-- repositories: backend, frontend-staff -->

Scope: **four operations, not three.** §10.1 adds `renewAccessToken` alongside `signIn`,
       `signOut` and `signedInStaffMember`, and `SignInResult` now carries `accessToken`. Q1
       is closed: the session is the two-token cookie flow, and its nine tables are
       #data-model's

Constraint: **the staff audience adds NO `healthCheck`.** The backend row ships one for its own
            audiences, and copying it across out of habit is the obvious thing to do — do not.
            §10.1 is the complete operation list, and §7's Authentication row is written
            against it staying that way: "every operation authenticates by one of two
            credentials". A fully-public operation nobody declared makes that false

Constraint: **two operations are reachable without a session, and only two.** `signIn`, which
            begins one, and `renewAccessToken`, which authenticates by the refresh-token cookie
            alone because it has to work once the access token has expired. `signOut` also
            proves itself by the cookie but still requires a session. Whatever the engine's
            skip-filter list ends up holding, those three are the operations that skip the
            access-token header — and nothing else does

Constraint: **rate limiting is this feature's work, and it is new scope** (§7). `signIn`, 10
            FAILED attempts per 15 minutes PER EMAIL ADDRESS — keyed on the address, not the
            caller's IP, because the staff share one office address and an IP-keyed limit
            would let one wrong password lock out everybody. Successes count for nothing.
            `renewAccessToken`, 60 per hour per refresh-token series

Conflict: this feature **opens the staff audience** — the SDL, the engine, the context pair and
          the one hand-written entry in `server/index.js` (which starts its servers with no
          scanning). #expense-entry and #monthly-summary then add resolver files only, which
          register by directory scanning. It also writes the scalars / `Pagination` / `Sort`
          block the later two append beside

Conflict: appends to the staff audience's SDL and to its GraphQL type declarations under
          `types/`. Two other features carry the same mark

Constraint: signing up and resetting a password by mail are out of scope **for now** (spec §4).
            Accounts are issued by an operator outside the product, so build **no sign-up
            operation and no sign-up screen** — and leave the password a one-way hash on the
            member of staff's own row, so a later reset changes that row alone

Constraint: there is no second actor (spec §3). Build no admin screen and no operator screen.
            The operator never signs in

Note: spec §7 forbids a password or a password hash reaching **any** response or **any** log
      line, and this feature's acceptance criteria repeat it. It is checkpoint 8's business as
      well as checkpoint 6's

Note: the two refusals — an address with no account, and a correct address with the wrong
      password — must be **identical**, and neither may say which of the two it was
      (spec §10 acceptance). That is a single error code, not two

Note: the same holds for `renewAccessToken`'s three failure states — an expired, a revoked and
      an already-spent cookie are refused identically, and the refusal reveals which in none of
      them. A spent one revokes the whole series, which is the reuse detection §9.7's
      `used_at` and `revoked_at` exist for

Note: `renewAccessToken` belongs to the frontend's client layer and to no screen's call table
      (§10.2). Checkpoint 14 builds it; checkpoints 15 and 16 give it no UI

Note: `signedInStaffMember` exists because a session is a cookie (spec §10.1). The screens ask
      it on opening; it is what sends an already-signed-in person on rather than asking twice

## Spec gate
- [x] 1. Draft or confirm the specification  <!-- skills: hoc-requirement-definition; digests: none taken — an interactive checkpoint, run by the main session, hands no agent a digest. Matched against hora-skills-ort-core 0.2.0. Four edits routed to /hora-spec and approved as exact text before writing: §10.3 added, three counts corrected -->
- [x] 2. Verify the use cases can be met  <!-- skills: hoc-requirement-definition, hof-uiux-context; digests: none taken — interactive, no agent. Matched against hora-skills-ort-core 0.2.0 and hora-skills-ort-furo 0.1.0. Unlike #data-model this feature targets a frontend, so hof-uiux-context is in surface and was read; the file it owns does not exist yet and lands at the frontend gate (below) -->

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

## Checkpoint 1 — what the close reading found

`/hora-plan` had already verified §10's use cases and acceptance criteria exist. Reading them
closely enough to build from turned up **four defects, all routed to `/hora-spec` and all
approved as exact text before anything was written** (PR #13).

| What was wrong | Where | What it was |
|---|---|---|
| `signIn` called "the one operation reachable without a session" | §10.1's caller cell | **false against its own table.** The `renewAccessToken` row two lines below says it is reachable without one *by design*, and §7's rate-limiting row says "these are the **two** operations reachable without a session". The cell predates `renewAccessToken` |
| migrations map to "the three tables" | §15 | §9 declares nine, and this version adds a tenth |
| `schemas/staff/` maps to "the SDL of the seven operations" | §15 | §10.1 + §11.1 + §12.1 hold four, five and one: **ten** |
| §7's sign-in limit had nowhere to keep its count | §9 and §8 together | checkpoint 2's finding — below |

**Three of the four are the same defect: a count that was true when written and a later section
made false.** Third instance of it in this document, after the false authentication blankets
(PR #6) and the sentences the backup tables broke (PR #7). Each was caught by a gate rather
than by a reader, which is the gate working — but a count in prose is what keeps producing them.

**One use case reaches past this feature's own gate, and checkpoint 1 passed on a stated
reading rather than a fix.** §10's first use case ends "every screen after that knows who they
are", and `#sign-in` has one screen. Recorded as Q21, with the reading and what it costs if
wrong. It is not the forward-reaching *criterion* that would have been a stop: §10's eight
acceptance criteria are all local, and none mentions another screen.

## Checkpoint 2 — the walk

Two use cases, walked end to end on paper against the spec, in the main session as this
checkpoint requires. **One walks clean. The other had two steps with nothing behind them.**

### Use case 2 — signs out on a shared machine — walks clean

| Step | What it needs | Held by |
|---|---|---|
| signs out | an operation reachable once the access token has expired | `signOut`, which §7 authenticates by the refresh-token cookie |
| the session stops working | a way to revoke a whole series rather than one token | §9.7's `revoked_at` + `session_key`; §9.6's rows deleted by the same key |
| the next person is asked to sign in | the sign-in screen reachable with no session | §10.2, and `signedInStaffMember` returning nobody |

**Server-side revocation is sufficient and the walk confirmed it.** Nothing in §10 says the
cookie is cleared in the browser, and nothing needs to: §10's second criterion is that the
session "stops working the moment its holder signs out", which a revoked series delivers even
if a stale cookie lingers.

### Use case 1 — signs in Monday morning — needed two things

**First: §7's sign-in limit had no store.** §7 limits `signIn` to 10 failed attempts per 15
minutes per email address, and §10's eighth acceptance criterion is written to be checked. But
a failed sign-in wrote no row anywhere — §9's nine tables record no attempt, and §8 declares
MariaDB alone while **stating outright that Redis is not declared**, with its reason. The
criterion had nothing behind it.

Resolved in the spec as **§10.3 `sign_in_attempts`** (PR #13), under §10 rather than §9 so its
`id` joins to `sign-in`, the feature §7 already assigns this rate limiting to and the only
feature that reads or writes it. Under §9 it would have belonged to `#data-model`, which is
accepted and whose own criteria never mention a sign-in attempt.

**`renewAccessToken`'s 60-per-hour needed nothing, and the walk is what established that.**
Each rotation inserts a §9.7 row carrying `session_key` and `generated_at`, so a series' rows
inside the last hour **are** its renewal count. No second table, and no column added to one.

**Second: nothing can issue the first account.** The use case begins with an address and
password "they were issued", and there is no account and no route to one — `seeders/` holds two
seeders, both for `expense_categories`, and §4 forbids a sign-up operation on purpose.
Recorded as Q22. **No spec change**: the build obligation is unambiguous and lands at
checkpoint 5 as development seeders for `staff_members`, `staff_member_secrets` and
`staff_member_password_hashes`, which this feature's tests need by the always-on testing rule
and which checkpoints 17 and 18 need to sign in as anybody at all. What stays open is the
deployed side, where a seeder has no business carrying a real password.

### What the walk deliberately did not do

**It did not walk §11's or §12's use cases.** Those are `#expense-entry`'s and
`#monthly-summary`'s, and `#data-model`'s checkpoint 2 already confirmed §9 can represent every
state all seven need. Re-walking them here would check the same thing against a spec that has
not changed for them.

## Resolver id block — 101, allocated here

The always-on resolver rule requires every query and mutation to hold an entry in
`server/graphql/resolver-id-hash-<audience>.js`, with the id embedded in each error code
(`203.M018.001`). **The backend row ships no such file for any audience** — the two audiences
it came with hold one `healthCheck` each and no error codes — so `#sign-in` creates the staff
one, and with it the numbering every later feature appends to.

**One hundred block per feature, in `_plan.md` order:**

| Feature | Block | Operations |
|---|---|---|
| `#sign-in` | **101–199** | `M101` signIn, `M102` signOut, `M103` renewAccessToken, `Q101` signedInStaffMember |
| `#expense-entry` | 201–299 | three mutations and two queries, allocated at its own checkpoint 3 |
| `#monthly-summary` | 301–399 | one query, likewise |

**Why a block rather than one sequence:** three features append to the same file, two of them
after this one merges. A single running counter makes the next number depend on what merged
first, so two features developed in parallel collide on it. A block per feature is decided
once, here, and neither later feature has to read the file to know its own numbers.

**Mutations and queries number independently** (`M101` and `Q101` coexist), which is the rule's
own scheme — the letter carries the kind, so the digits need not.
