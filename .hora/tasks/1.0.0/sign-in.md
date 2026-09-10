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

Note: **a stub is a public endpoint.** The authentication filter is built from the `actual/`
      resolvers alone, so a stub-only field gets `filter === undefined` and no check runs at all
      — the same mechanism `schemasToSkipFiltering` uses, without anybody listing it (Q28). So
      `signedInStaffMember` is reachable without a session for as long as it is a stub, though it
      is deliberately absent from the skip list. Three gates need to know: checkpoint 8's audit
      sees four public operations where §7 permits three; the frontend gate builds a client
      against an endpoint that never refuses, then meets one that does at 16; and a deployment
      where checkpoint 6 missed an operation serves it publicly with hardcoded data and nothing
      fails

## Spec gate
- [x] 1. Draft or confirm the specification  <!-- skills: hoc-requirement-definition; digests: none taken — an interactive checkpoint, run by the main session, hands no agent a digest. Matched against hora-skills-ort-core 0.2.0. Four edits routed to /hora-spec and approved as exact text before writing: §10.3 added, three counts corrected -->
- [x] 2. Verify the use cases can be met  <!-- skills: hoc-requirement-definition, hof-uiux-context; digests: none taken — interactive, no agent. Matched against hora-skills-ort-core 0.2.0 and hora-skills-ort-furo 0.1.0. Unlike #data-model this feature targets a frontend, so hof-uiux-context is in surface and was read; the file it owns does not exist yet and lands at the frontend gate (below) -->

## Backend gate
- [x] 3. DB and API schemas  <!-- skills: hor-database-design, hor-sequelize-migration, hor-sequelize-model, hor-graphql-schema, hor-graphql-server-engine, hor-type-interface, hor-cookie-authentication, hor-constant-definition, hoc-naming, hoc-jsdoc; digests: hora-skills-ort-renchan 0.1.0 and hora-skills-ort-core 0.2.0. hor-graphql-schema and hor-graphql-server-engine had no digest at the installed version and were taken before any agent ran. GAP: the tests this checkpoint owed came from the always-on testing rule rather than the exit condition, and hoc-jest and hor-backend-testing were read in full because neither had a digest yet — both now taken, for checkpoints 6 and 16 -->
- [x] 4. Stub API  <!-- skills: hor-stub-api, hor-backend-testing, hoc-jest, hoc-naming, hoc-jsdoc; digests: hora-skills-ort-renchan 0.1.0 and hora-skills-ort-core 0.2.0. hor-stub-api, hoc-jest and hor-backend-testing were all taken for this checkpoint. LIMIT: "callable from outside" is evidenced through the framework's own schema-and-resolver path with real GraphQL documents executed in process, not over a socket — Q24 means no server on this machine listens at all -->
- [x] 5. The modules the implementation needs  <!-- catalog checked FIRST, once, for the whole checkpoint, against @openreachtech/hora-ecosystem 0.1.0 (33 tracked repositories); skills: hor-cookie-authentication, hor-sequelize-model, hor-sequelize-seeder, hor-database-design, hor-constant-definition, hoc-classes-principles, hoc-classes-constructor, hoc-classes-inflators, hoc-methods, hoc-properties, hoc-scope, hoc-async, hoc-naming, hoc-jsdoc, hoc-jest, hor-backend-testing; digests: hora-skills-ort-renchan 0.1.0 and hora-skills-ort-core 0.2.0, plus six hoc- skills read in full for want of a digest -->
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

## Row-id prefix — `101`, and a correction

**`101` is this feature's row-id prefix**, allocated through the equipped allocator skill with
`sign-in` as the requester, under the lock that skill requires. Ids are `10100000`–`10199999`,
and they are exclusive to this feature in every table, in every seeder and in every test that
creates its own rows. `#data-model` holds `100`.

**Corrected from what this file said an hour earlier.** The first version of this section called
`101` a *resolver* id block and laid out `M101` / `Q101` / a hundred-block per feature. That was
wrong: the allocator hands out **row** ids for database rows, and it had already handed `100` to
`#data-model` for the four category rows `10000001`–`10000004`. The number was right by
coincidence — `101` is what the allocator returns for the second requester either way — which is
exactly why the mistake would have survived unnoticed. **The reusable part: `101` was written
down from memory of a previous session rather than read back from the allocator**, and the two
different things it could have meant were never separated until the skill was read.

**The equipped digest carried the same hazard and was corrected with it.** The seeder skill's
digest stated "allocated row-id prefix is `100`" as a project fact, which is `#data-model`'s
prefix; an implementer agent for `#sign-in` handed that digest would have written `100xxxxx` ids
straight into `#data-model`'s space. It now states where the prefix comes from instead of
naming one.

## Resolver ids — a separate thing, and this feature creates the file

The always-on resolver rule requires every query and mutation to hold an entry in
`server/graphql/resolver-id-hash-<audience>.js`, with that id embedded in each error code
(`203.M018.001`). **These are unrelated to the row-id prefix above** — different space,
different allocator, different purpose.

**The backend row ships no such file for any audience.** Its two audiences hold one
`healthCheck` each and declare no error codes, so nothing forced the file into existence.
`#sign-in` opens the staff audience, so it creates the staff one and with it the numbering the
later two features append to.

| Feature | Ids | Operations |
|---|---|---|
| `#sign-in` | `M001`–`M003`, `Q001` | signIn, signOut, renewAccessToken; signedInStaffMember |
| `#expense-entry` | `M004`–`M006`, `Q002`–`Q003` | recordExpense, correctExpense, removeExpense; expenses, expenseCategories |
| `#monthly-summary` | `Q004` | monthlyExpenses |

**Plain sequential, not a block per feature.** The rule's scheme numbers within an audience and
the letter already carries the kind, so `M001` and `Q001` coexist. All three features write one
audience's file, and `_plan.md` fixes their order — so the next feature's numbers are known
from the plan without reading what merged. A hundred-block would buy nothing here and leave
`resolver-id-hash-staff.js` reading as though 297 operations were missing.

## Checkpoint 3 — what was built, what verification sent back, and what proved it

Two units, both exclusive: the `sign_in_attempts` table, and the staff audience's whole API
surface. The four operations could not be four units — they share one SDL directory, one
`.d.ts`, one engine and one context pair — so the checkpoint ran the surface whole, which is
what `/hora-build` prescribes when a file two units would both write cannot be given to one.

**Verification failed this checkpoint once, and the send-back was right.** The substance was
never in doubt — the verifier built the schema through the framework's own `GraphqlSchemaBuilder`
and got exactly the contract's type map, and read the applied table back out of the SQLite file
— but four units had shipped with no tests, while the tree held an exact tested sibling for
every shape they introduced.

### Three decisions, and both layout answers came from the always-on rules

| Question | The equipped skill | What was built |
|---|---|---|
| SDL: one flat file per audience, or a directory of numbered files? | `hor-graphql-schema` leaves it **unsettled**, and said so | **a directory** — `directory-structure.md` splits `schemas/` by audience as directories |
| GraphQL types: one file per resolver, or one per audience? | `hor-type-interface` wants `types/resolvers/<category>/`, namespace `graphql.<category>` | **one `types/StaffGraphQL.d.ts`**, namespace `server.graphql.staff` — `graphql-resolvers.md` prescribes it verbatim |
| Lift shared members into the app base class? | `hor-graphql-server-engine` advises lifting | **duplicated in the concrete engine**, as the tree does |

**The two layout answers point opposite ways and that is not an inconsistency.** Both follow the
always-on rules under Q10; the rules simply prescribe a directory for SDL and one file for types.
The directory also earns its keep: three features contribute to this audience, so each adds a
file instead of editing a shared one.

**Not lifting into the base was the cheaper future.** `/hora-build` classes a base class as a
conflict-proof file, so lifting would have rewritten three existing engines so one new one could
be added.

### The skip list is the feature's most dangerous line, and it is exactly three

`schemasToSkipFiltering` holds `signIn`, `signOut`, `renewAccessToken`. Verification traced what
that mechanically does: the framework maps ignored schemas to `null` and the resolver wrapper
calls `filter?.(…)`, so **no** filter runs — no `Unauthenticated`, no `Unauthorized`, no
`DeniedSchemaPermission`. An entry there is a public endpoint, and the framework offers no
"authenticate by the other credential" gate, so **the authentication debt moves wholly into the
three resolvers.** The engine says which operation owes which check, so checkpoint 6 cannot lose
it. `signedInStaffMember` is deliberately absent — §10 requires it refused without a session.

### `findUser` is deliberately unimplemented, and the failing direction is closed

It returns `null` through `super.findUser()`, with a docblock that opens by saying so and lists
three obligations it cannot discharge here: an access-token read with the fifteen-minute expiry
check, that read routed through `SessionClerk` — which carries no access-token method, making it
checkpoint 5's — and a decision about which entity it returns, because the framework publishes
`userEntity.id` as `#userId` and therefore as `#staffMemberId`. **Return the token row and
`staffMemberId` silently becomes the token's id.**

Every non-skipped operation therefore refuses, which is refusal rather than admission, and is
the correct answer for an audience with no resolvers.

### What could be proved, and what could not

| Claim | How |
|---|---|
| the migration works | ran it — all ten up, the composite index created as named, `down` drops both, re-migration clean |
| the SDL loads **from a directory** | the framework's own `SchemaFilesLoader` over the real path, then `makeExecutableSchema` |
| `healthCheck` is gone | absent from the **built** schema, not merely from the file |
| `rawBody` is read nowhere | grepped this repository and all of `renchan` — written in three engines, read by nothing |
| nothing regressed | the full suite, at every stage |

**Two limits, neither papered over.** Docker Desktop was down, so the migration is verified on
**SQLite only** and MariaDB verification is owed — `datetime(3)` is the dialect-sensitive field,
and it is not load-bearing for a fifteen-minute window. And a full server boot is impossible on
this machine at all (Q24), so `listen(4900)` and an HTTP probe of the endpoint were never
exercised; the schema was verified through the same code path minus the socket.

### The tests, and the two guards that were green while broken

The finding was closed with 68 tests, then reopened by a second verification for a structural
breach and closed again. Both halves matter:

**The breach.** The engine test declared four mocks and two context instances at describe scope
and fed **one** `cases` array to **four** sibling `test.each` calls. It did not false-pass —
the global `afterEach(restoreAllMocks)` covered it — but eight tests were mutating two shared
objects while installing spies on them, on the authentication filter. Now fifteen `cases` arrays
to fifteen `test.each` calls, nothing at describe scope, and `mockReturnValue` rather than
`Once`, so the hazard is gone rather than relocated.

**The two holes, and both were falsified against the source rather than the test:**

| Break this | Tests that now fail |
|---|---|
| add the `rawBody` verify callback back to the engine | **2** — the fifteen-line comment's deliberate omission is under test |
| drop the broker from `StaffGraphqlShare.createAsync` | **6** — it left the suite green before |
| `AUTH_COOKIE_SECURE=false` | **2** — staff was the only audience depending on that cookie and the only one outside the existing guard |
| disable `beforeSave` / `beforeBulkCreate` / `beforeBulkUpdate` | **5 / 2 / 2** — each of the three write paths is independently covered |

**That 5/2/2 split is worth more than a larger number would be.** It was first measured as "9 of
9", from a probe run against a database still holding the previous run's rows — every test failed
on a primary-key collision rather than on the missing hook. A falsification proves nothing unless
the baseline passes under the same conditions; two things had been changed at once. Re-run with a
refresh before each probe, the split shows each hook has its own test, where "all 9" would have
meant one path was doing all the work.

**One residual weakness, recorded rather than fixed.** Position 3 of the middleware stack is
pinned as `name: ''` — the upload middleware is an unnamed arrow — which catches a reorder or a
drop but not substitution with a *different* anonymous middleware. Pinning it by identity needs a
spy on renchan's ESM namespace, which is not writable, and the alternative was wrapping the import
in the engine solely so a test could reach it. **Production code shaped by its test is the worse
defect**, so the narrower assertion stands.

### The documentation defect this checkpoint caused

The README said "the three servers" and, worse, "Each GraphQL endpoint answers a health check out
of the box" — false the moment this audience opened, and precisely the sentence that would lead
the next reader to restore the `healthCheck` Q23 had removed hours earlier. **The comment left in
the SDL guarded the schema; the README argued against it.**

Fixed in both languages, along with five more lines the audience falsified — and the tagline,
which the first fix pass read straight past while hunting counts and a grep for number words then
found. Every replacement is written **without a count**, so the next audience falsifies nothing:
a scope where there was a number, or a pointer at the command or file that enumerates the real
thing.

**Fourth instance in this project of a count true when written and falsified by growth**, after
the spec's false authentication blankets, the sentences the backup tables broke, and the three
counts this feature's own spec gate corrected. The first three were prose in `specs/`; this one
was code documentation, and this change set caused it.

## Checkpoint 4 — the stubs, and the one thing they hand forward

Four stubs, one unit, no engine or SDL or contract change: the engine already declares the stub
pool and `DeepBulkClassLoader` scans it, so a stub registers by existing.

**The exit condition says "callable from outside", and this machine cannot do that at all.** Q24:
`server/index.js` dies on `await activate()` before anything listens, and it breaks the customer
and admin audiences with it. So the claim was evidenced the same way checkpoint 3's schema was —
through the framework's own path minus the socket: `StaffGraphqlServerEngine.createAsync()` →
`GraphqlSchemaBuilder.buildSchema()` → real GraphQL documents through `graphql()`. All four answer
with data satisfying the SDL and no `errors` key.

**That execution is a shipped test, not a line in a report.** It is the only artifact that
actually evidences this checkpoint's exit condition, so it had to survive the run that produced
it.

### What lint caught that the suite could not

The execution test first asserted the **whole** GraphQL envelope with one `toEqual`, which is the
stronger assertion — a stray `errors` key fails it. **It is not writable here: `data` is on the
`id-denylist`,** so it cannot be an object-literal key, and four cases tripped it.

Rewritten to assert `actual.data` against the payload plus `expect(actual).not.toHaveProperty('errors')`
— two allowed matchers, no banned identifier, no logic in the body. **Then falsified:** a stub
made to throw fails exactly its own case. Without that second assertion it would have passed,
because a GraphQL error arrives *beside* a still-present `data`.

### What the stubs deliberately do not do

**No stub authenticates, and each says so in its own JSDoc.** A stub-served field is handed no
filter (Q28), so a stub that read `context` and refused would be theatre — and worse, it would
read as evidence that the endpoint is protected. `SignedInStaffMemberQueryResolver` carries the
sharp version: the engine leaves it out of `schemasToSkipFiltering` **so that the filter would
refuse it**, and as a stub it is reachable anyway.

**The tests back this checkpoint's exit condition, not this feature's acceptance criteria**, and
that distinction is the honest one. Every §10 criterion — identical refusals, a session surviving
a reload, a spent refresh token revoking its series, the eleventh attempt inside fifteen minutes
— needs the real resolver and is checkpoint 6's to prove. A stub cannot be made to demonstrate
any of them, and none was bent to look as though it does. The one criterion these do reach is
that no operation returns a password or a hash: the shapes are asserted whole.

### Handed to checkpoint 6

**`execute-stub-operations.js` will start lying the moment the real resolvers land.** It passes
only while the four return these exact literals, which the real ones will not. It sits under
`…/staff/stub/`, so checkpoint 6 either deletes it with the stubs or rewrites it against seeded
data — and the second is the better trade, since it is the only test that exercises the audience
end to end through the framework rather than a class in isolation.

**`validate-unique-error-code.js` was deliberately not given a `staff/stub/` case.** A stub
declares no error code at all, so the assertion would be `[] toEqual []` — vacuous today and
vacuous forever, since a stub owning a code is what the convention forbids. The stronger check is
where it went instead: each stub's own `.get:errorCodeHash` test asserts **empty**, not merely
unique. `customer/stub` and `admin/stub` are uncovered for the same reason.

## Checkpoint 5 — the catalog check, four modules, and a hole the guard was hiding

**The catalog check ran first, once, and it changed what this checkpoint was.** The plan assumed
the session layer had to be built. It was mostly already there: `SessionCredentialGenerator`
covers §9.6 and §9.7's token pair, series and digest storage **entirely**; `SessionClerk` had 21
members including every revocation primitive; and the models already carried `isAvailable`,
`isExpired`, `extractUserId`, `verifiesPassword` and `generateNormalizedEmail`. So the checkpoint
shrank to two gaps, two counters, one encipher and the seeders.

**It also stopped two mistakes before they were made.** `express-rate-limit` is already a
dependency of this repository and is exactly what §7 rejects by name — HTTP middleware keyed per
request or per IP, where §7 keys on the address "because the staff sit behind one office address
and an IP-keyed limit would let one person's wrong password lock out everybody". And
`mentsu-random-text-generator` is catalogued but builds every character from
`Math.floor(Math.random() * …)`, so it must never mint a credential; `SessionCredentialGenerator`'s
own docblock had already argued that out.

**One package's absence is the finding worth passing on.** `mentsu-encipher` is the only
catalogued name matching a password hash, and it is `turned-off` in the catalog's own rulesets —
so the most security-sensitive module in this feature had no in-house answer and needed a
dependency chosen. That belongs with whoever maintains the catalog.

### The security hole, and that an instruction of mine was hiding it

`#spendRefreshToken()`'s guard was `{ tokenHash, usedAt: null }`. A refresh token **revoked but
never spent — exactly what `signOut` leaves behind** — matched it, was marked spent, and issued a
fresh pair for a dead series. So did one past `expiredAt`.

**§10's "a session stops working the moment its holder signs out" was false**, held up only by a
resolver pre-check checkpoint 6 had not written yet.

**The assignment told the agent not to change that guard.** It was meant as "do not remove the
`usedAt: null` condition, which is what makes reuse detectable"; the agent read it as hands-off,
which is what it actually said, and reported the hole rather than fixing it. That reading was
correct and the instruction was wrong. Corrected, and the guard now writes the model's own
`isAvailable()` as a `where` clause.

**Which makes §10's identical-refusal criterion structural rather than agreed.** Expired, revoked
and already-spent now all reach `updatedCount === 0` — one branch, one message, one revocation —
instead of three code paths each choosing to return the same error. The message stopped naming a
state (it said "already spent", which for an expired token was simply false), and
`revokeReusedSeries` became `revokeUnavailableSeries`, because the branch cannot tell the three
apart and §10 requires that it not.

### The rollback trap, and why the obvious fixes fail

A revocation on the reuse branch is undone by the error that reports it: `#invokeRotateSession`
throws when the result reports an error, and that rolls its transaction back. The series would
read as revoked in the returned result and stay live in the database — and the criterion would
fail **silently**, because the operation refuses either way.

| the obvious fix | why it fails |
|---|---|
| a second transaction | `beginTransaction` defaults to `SERIALIZABLE`, and the guarded `UPDATE` already holds that row on the unique index — the second writer waits behind the transaction it is trying to outlive |
| revoke outside the caller's transaction | same lock, same wait |
| split the spend into its own committed transaction | breaks spend/issue atomicity: a failure while issuing strands a live session with a spent cookie |

**What was built instead reframes it: a rotation has three outcomes, not two.**
`RotatingSessionResult#shouldRollBack()` is `hasError() && !hasRevokedSeries()`, and
`#invokeRotateSession` asks that instead of `hasError()`. A refusal that revoked commits; a
revocation that itself failed carries no revocation, so it rolls back and nothing falsely claims a
series was revoked.

**One asymmetry is irreducible and is written into the docblocks:** a revocation cannot survive a
transaction somebody else chooses to roll back. On the route the clerk owns it guarantees the
commit; on a caller-supplied transaction the contract is `shouldRollBack()`, not `hasError()`. No
caller passes one today, so checkpoint 6 must not write `if (result.hasError()) rollback` around a
rotation.

### Everything was mutation-checked rather than argued

| break this | tests that fail |
|---|---|
| the rollback fix (revert to `hasError()`) | **6** — `revokedAt: null`, the silent failure exactly |
| the guard (revert to `usedAt: null` alone) | **8**, with the 30 originals still green |
| the email normalization (replace with identity) | **8** |
| `Op.gte` → `Op.gt` on the window bound | **3** |
| the `rawBody` omission, the broker, `AUTH_COOKIE_SECURE` | 2, 6, 2 — from checkpoints 3 and 4 |

**Under the old guard the expired case did not merely fail to refuse — it issued a working pair
with a fresh fortnight's expiry, minted from a token dead two weeks.** That is the one output in
this checkpoint worth reading twice.

### Three design calls, each with a reason that is not "it looked nicer"

**Base plus two thin concretes for the counters, not one parameterized class.** The two do not
differ only in values: the sign-in limit normalizes its key and the renewal limit must not, which
is an overridden method rather than a parameter. Parameterizing would also have put §7's numbers
at every call site, so each resolver would restate the spec.

**The renewal counter needs no table**, which is checkpoint 2's finding paying off: each rotation
inserts a §9.7 row carrying `sessionKey` and `generatedAt`, so a series' rows inside the last hour
*are* its renewal count.

**The constants split two ways, and the distinguishing test is worth recording.** §7's windows and
thresholds are a shared category read by two limits, their resolvers and their tests, so they take
the `.cjs` master plus ESM bridge pair that `authConstants` and `expenseStatusConstants` use. The
encipher's cost factor is **one module's own knob** — nothing else reads it, and bcrypt writes the
factor into each digest so no seeder ever needs it — so it sits at the top of its own file, per
`javascript-style.md`. Two units read one digest oppositely and both were right.

### Q22 closed in substance, and the trap it existed to avoid

Thirteen members of staff, eleven with a working credential, **verified against the running
database** rather than reported: every address already lower-cased, every digest matching the
shape the testing rule asserts, `compare` true for all eleven recorded plaintexts and false for a
near-miss on all eleven, and `down` reverting cleanly.

**`bulkInsert` bypasses model hooks entirely**, so `StaffMemberSecret`'s normalizing hook never
fires for a seeded row. An address seeded with a capital in it fails no insert and no test — it
just makes the account unfindable by `signIn` and uncountable by §7's limit. A test holds it by
normalizing the address the way `signIn` will and requiring a row back.

**`development/` rather than `dev-master/` decides something real.** `test.sh` runs `tests/empty/**`
on master seeds only, *then* `db:seed:dev` — so no account exists in phase one, which is exactly
the split §10's "an address with no account" criterion wants (Q29). Under `dev-master/` phase one
would already hold accounts and the distinction would be gone.

**Two members of staff deliberately hold no complete credential**, and Q22 is why: accounts are
issued by hand across three tables, so a half-issued one is the most likely way one goes wrong
here, not a hypothetical — and §10's identical-refusal criterion needs a row to test against.

### The exit condition, gathered by the main session

Every module checkpoint 6 imports was confirmed to resolve **with the members it will call** —
`PasswordEncipher`, `SessionClerk` (all six), `SessionCredentialGenerator`, `RotatingSessionResult`,
the three rate-limit classes, the constants bridge, the resolver id hash and the cookie clerk. Ten
of ten. The six models are established by the passing suite.

### Handed to checkpoint 6, and two are load-bearing

1. **The equipped skill contradicts §10.** `references/resolvers.md` has `renewAccessToken` return
   a distinct `RefreshTokenReused` error; §10 requires expired, revoked and already-spent refused
   identically. The clerk now hands back one indistinguishable error, so the resolver only has to
   not undo that.
2. **§7's limits are written and not wired.** The revocation-on-refusal write now sits on the
   unauthenticated path — `renewAccessToken` is reachable without a session by design — and the
   reasoning that this is bounded rests on the 60-per-hour-per-series limit **being attached**. A
   fabricated cookie never reaches the guard (`findRefreshToken` returns null), so the reachable
   case is one real dead cookie replayed, which the per-series limit bounds. If the limit ships
   unwired, that branch is a free write for a stolen cookie.
3. **`findUser` cannot get the member of staff from the clerk**, by design — the clerk holds only
   the two token models. `extractUserId()` on the returned access-token entity is the id to read
   `StaffMember` by, and doing so is not a second door onto the token tables.
4. **The resolver's `isAvailable` pre-check is now defence in depth**, not the only defence. Keep
   it; do not rely on it.
