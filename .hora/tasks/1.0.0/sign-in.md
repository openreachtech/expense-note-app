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
- [x] 6. Actual API  <!-- skills: hor-mutation-resolver, hor-query-resolver, hor-resolver-validator, hor-graphql-server-engine, hor-cookie-authentication, hor-backend-testing, hoc-jest, hoc-classes-notations, hoc-methods, hoc-errors, hoc-naming, hoc-jsdoc; digests: hora-skills-ort-renchan 0.1.0 and hora-skills-ort-core 0.2.0. hor-mutation-resolver, hor-query-resolver and hor-resolver-validator had no digest at the installed version and were taken before any agent ran; each carries a "Settled by the main session" section for the arbitrations below. 15 of the 18 skill-versus-rule conflicts fell in this checkpoint -->
- [x] 7. Worker  <!-- n/a: decided with hor-execution-placement-pattern, read in full (no digest — an interactive decision, no agent ran), not by eye. Every piece of this feature's processing belongs in the request path; the two candidates that do not are barred by §8 and §7 respectively. Reasoning below -->
- [x] 8. Security audit  <!-- skills: hor-security-audit, read in full (no digest at the installed version). Ran over the whole change set of checkpoints 3 to 6, on code already committed, lint-clean and passing 876 tests. Six findings, two of them MEDIUM and reachable from the internet; all six fixed in f427e22, b0a7cc5 and 62d6a61, each with a test that fails without the fix. Three INFO items recorded, one of them (the WebSocket channel bypassing both HTTP mitigations) left open as an engine-level decision this feature does not own -->
- [x] 9. Verify the use cases again, against the built API  <!-- interactive, run by the main session against the merged tree; no agent, so no digest taken. All eight of §10's acceptance criteria walked one at a time against something that would fail if the criterion stopped holding. Seven held; the eighth -- "none writes one into a log line" -- was FALSE on live, staging and production, where no `logging` key left Sequelize at its `console.log` default and every sign-in wrote an email address to stdout. Fixed in 3ac78a0 with a test that reads the config file rather than a connection, because the suite runs only under `development`, which already had logging off. Both use cases verified as far as a backend can carry them; the screen half is checkpoint 18's -->

## Frontend gate
- [x] 10. Open the frontend  <!-- skills: hof-nuxt, hof-furo-env; digests: hora-skills-ort-furo 0.1.0. Neither had a digest at the installed version; both taken before the implementer ran, and both carry a conflicts section naming where `D:\ORT\rules\` wins. Route `/sign-in` was FORCED by the boilerplate's existing `middleware/000.gateway.global.js`, not chosen. Reachability established from the built route table rather than a curl, because `ssr: false` makes nitro answer every path with the same SPA fallback. Env repointed from the boilerplate's customer:3900 to the staff endpoint on 4900, verified against the backend engine. One unit claim checked and rejected: the tree doc was not stale, the word "middleware" is overloaded -->
- [x] 11. Reconfirm UI/UX and the use cases  <!-- interactive, main session, in conversation; skills: hof-uiux-context, read in full (no digest -- no agent ran). Wrote `expense-note-frontend-staff/ai/contexts/uiux-context.md`, which did not exist; every answer tagged [spec]/[user]/[tree]/[chosen] so checkpoint 18 does not audit an arbitrary answer as a rule. Three questions put to the user: devices, accessibility target, tone. BOTH of section 10's use cases were found to end outside this feature -- the sign-out control lives on section 11.2's screen and "every screen after that" means screens #expense-entry owns -- so neither is closable at this feature's gate. Not sent back to checkpoint 2: the spec is coherent, the paths exist at version level. Q41 raised, and checkpoint 9's claim that the screen half is "checkpoint 18's" is corrected there -->
- [x] 12. Component design  <!-- skills: hof-uiux-forge, hof-cp-text-field, hof-cp-button, hof-cp-control-block, hof-prohibits; digests: hora-skills-ort-furo 0.1.0, all five taken at this checkpoint since none existed. Four components exist (FuroEmailField, FuroPasswordField, FuroButton, FuroControlBlock), one is new -- the form-level refusal message, because section 10 requires an identical refusal and `errorMessages` attaches to a single control. Eight source-verified facts contradict the skills or the library's own manifest, two of them WCAG 2.2 AA gaps to compensate for at 15. Three digests independently found that NO component had a working colour: nothing imported furo.css, so `--color-ring` -- FuroButton's only focus indicator -- resolved to nothing. Fixed in 2899424, verified out of the built bundle -->
- [x] 13. The frontend modules the implementation needs  <!-- skills: hof-error-handling, hof-modules; digests: hora-skills-ort-furo 0.1.0, both taken at this checkpoint. The kit requires stating WHICH half applies: the error-code mapping applied (13 codes), the shared-module half is n/a -- one screen, `components/` empty, no second call site -- so `app/modules/` was deliberately not created. Static-string variant taken over the i18n locale-path variant, following the no-localization-layer decision already recorded at checkpoint 11 rather than making a new one; no i18n dependency added. `hoc-properties`' Map prohibition and the skill's own "no reverse map" both killed the skill's two-hop design, so the hash is keyed by code directly. `BaseAppGraphqlCapsule` is conflict-proof and was written by the main session, with a test reading removed for using a banned case-generating loop that ESLint does not catch -->
- [x] 14. API client  <!-- REACHED IN PART, and recorded as such. skills: hof-graphql; digest: hora-skills-ort-furo 0.1.0, taken at this checkpoint. Four Launcher/Payload/Capsule trios. The "matching the contract exactly" clause is MET -- each document extracted from the file on disk and validated against `.hora/contracts/1.0.0/` with graphql 17.0.2, with a negative control that was itself corrected after tripping on the wrong error. The "works against the stub" clause is NOT met and is unmeetable here: Q24 means no server boots on Windows, so nothing listens. Not faked and no mock server called a stub. `types/graphql-schema.d.ts` created -- a type projection, not a second authority, because nothing can validate against it -->
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
   unauthenticated path — `renewAccessToken` is reachable without a session by design — so §7's
   60-per-hour-per-series limit needs attaching to it. A fabricated cookie never reaches the guard
   (`findRefreshToken` returns null), so the reachable case is one real dead cookie replayed.

   > **CORRECTION, made at checkpoint 6 — the sentence this paragraph originally ended with was
   > wrong, and it was mine.** It read: "which the per-series limit bounds. If the limit ships
   > unwired, that branch is a free write for a stolen cookie." **The limit does not bound it.**
   >
   > `AccessTokenRenewalRateLimit` counts rows in `staff_member_refresh_tokens` by `sessionKey`
   > and `generatedAt` — and **a refused renewal generates no row**, because the guard matched
   > nothing so no rotation happened. So replaying a dead cookie never advances its own count, and
   > the limit is never reached however many times it is replayed.
   >
   > **What the limit does bound is row growth**, which is what §7 actually asks of it: a live
   > series can rotate at most 60 times an hour. That is intact and wired at checkpoint 6.
   >
   > **The residual, stated properly:** each replay of a known-dead cookie costs one transaction
   > and two statements, and after the first refusal both are no-ops — `revokedAt` no longer
   > matches, and the access tokens are already deleted. So it is bounded in *effect* rather than
   > by the limit. Recorded as Q38. Found by the checkpoint 6 unit that wired the limit, which
   > checked the claim rather than inheriting it.
3. **`findUser` cannot get the member of staff from the clerk**, by design — the clerk holds only
   the two token models. `extractUserId()` on the returned access-token entity is the id to read
   `StaffMember` by, and doing so is not a second door onto the token tables.
4. **The resolver's `isAvailable` pre-check is now defence in depth**, not the only defence. Keep
   it; do not rely on it.

## Checkpoint 6 — the actual API, and the checkpoint where the arbitrations happened

**Four resolvers replaced four stubs**: `signIn`, `signOut` and `renewAccessToken` as mutations,
`signedInStaffMember` as a query, each with its stub still standing beside it. Thirteen error codes
across them — `203.M001.001`–`004` for the validator, then `204.M001.001`–`003`, `204.M002.001`–
`002`, `204.M003.001`–`002` and `204.Q001.001`–`002` for the database refusals.

**Q28 closes here, and it closes by construction rather than by a fix.** A stub is a public
endpoint: the framework builds its authentication filter hash from the resolver pool it is given, so
while the only `signedInStaffMember` was a stub, the operation answered without a session. The pool
is now `actual/`, so the filter is built from resolvers that authenticate. Nothing was added to
close it — the hole was the stub's existence, and the stub is no longer what answers.

**`schemasToSkipFiltering` is the most dangerous line in the feature**, and it is written to be read
that way. An operation listed there is handed to `FilterSchemaHashBuilder` as an ignored schema,
mapped to `null`, and called as `filter?.(…)` — so it gets no `Unauthenticated`, no `Unauthorized`
and no `DeniedSchemaPermission`. **An operation wrongly listed there is a public endpoint.** The
list is exactly the three §7 declares reachable without an access token, and the docblock says why
each one is there rather than leaving the reader to infer it.

### The validator, and the two limits that are not the obvious ones

Five checks behind four error names: `MissingEmail`, `MissingPassword`, `MalformedEmail` and
`TooLongPassword`. Two of them are bounds rather than formats, and both were chosen against a
stated reason:

- **72 bytes, measured with `Buffer.byteLength`, not `String#length`.** bcrypt truncates at 72
  *bytes*, so a password of 72 multi-byte characters is silently cut. Counting characters would
  accept an input the hash does not fully cover (Q32). Replacing the byte count with `String#length`
  fails 3 tests.
- **191 characters, counted with `Array.from(email).length`, carried on `MalformedEmail`.** §9.4
  stores the address in `varchar(191)`; without the bound a 300-character address reached the
  INSERT. It rides the existing error name rather than adding a fifth, because §10 requires a
  malformed address and an over-long one to be refused identically.

`EMAIL_PATTERN` is `/^[^\s@]+@[^\s@.]+(?:\.[^\s@.]+)+$/u` — deliberately not an RFC 5322
attempt. It requires a dot-separated domain and rejects whitespace, and nothing more, because §10
asks only that a malformed address be refused and the account lookup decides the rest.

### Where the eighteen arbitrations landed

**Fifteen of the eighteen recorded below fell in this checkpoint**, which is where an equipped skill
and an always-on rule had the most to disagree about: the validator's entrypoint, its directory, the
error-code family, member order, method order, matcher choice, case-field names, the Act variable
and fixture hoisting. Three changed real structure. The full list, with what each skill said and
what was built, is its own section below — it cannot be reconstructed from the merged tree, because
the tree records only the outcome.

### The test file that became false, and was replaced rather than patched

Checkpoint 4 left `execute-stub-operations.js`, which drove the four operations end to end through
the framework's own schema-and-resolver path. **The moment actual resolvers landed, its name and its
subject were both wrong** — it was no longer exercising stubs. It was reworked into
`tests/_orders/SignIn/execute-staff-session-operations.js`, and given §10's cross-operation
criterion to own: sign in, read yourself, renew, read yourself again, sign out, and fail to read
yourself.

**Two units had also placed their `_orders` files by source path**, which the always-on rule forbids
— `_orders` groups by domain. Consolidated to `tests/_orders/SignIn/`.

### Everything mutation-checked, not argued

| break this | tests that fail |
|---|---|
| restore the `'*'` wildcard in `schemasToSkipFiltering` | **11** |
| remove the 191-character address bound | **12** |
| remove the resolver's `isAvailable` pre-checks | **4** |
| restore the upload middleware | **3** |
| give the two credential refusals different error codes | **11** |
| count the password with `String#length` instead of bytes | **3** |

The suite went from 242 to **910** across the checkpoint, lint clean throughout.

## Checkpoint 7 — the placement decision, and the one piece that would belong to a worker

**Not applicable, and the skill is what says so.** `/hora-build` is explicit that this is decided
with the placement skill rather than by eye, because "a write that looks synchronous, a side effect
that looks small, a notification that looks instant — each is a candidate".

### Running the decision flow over every piece of this feature's processing

| processing | flow step | placement |
|---|---|---|
| `signedInStaffMember` | step 1 — read-only, so "return it via an API query and you're done" | **API** |
| `signIn` | validate, one indexed `COUNT`, one indexed read, a 59ms compare, one insert or three | **API** — light, and the caller waits for the access token |
| `signOut` | read a cookie, one indexed read, two writes | **API** |
| `renewAccessToken` | read a cookie, one indexed read, one `COUNT`, one update and two inserts | **API** |

Nothing here is "heavy, time-consuming, or uncertain due to external dependencies" — the skill's own
example of heavy is processing that "takes tens of seconds". The slowest step in the feature is
bcrypt at 59ms, and it **cannot** be deferred: a password check the caller does not wait for is not
a password check. `postWorkersPath` stays `null`, and no `server/graphql/post-workers/` directory
exists in the repository.

### Three candidates considered and rejected, each for a stated reason

**1. Recording a failed sign-in attempt — rejected on correctness, not preference.** It looks exactly
like the skill's post-worker case: a small write, a side effect of a refusal, and the caller does
not need it. **But §7's rate limit counts those rows, so the row must be durable before the refusal
returns.** Deferred to a post-worker, eleven rapid attempts could each be answered before any row
landed, and every one of them would see fewer than ten rows. The limit would be unenforceable. This
is the clearest case in the feature of processing that *reads* as deferrable and is not.

**2. Pruning expired access tokens — genuinely worker work, and unavailable at 1.0.0.** The
placement skill puts it squarely in "heavy, automatic by interval/time → Worker (scheduled)". §9.6
says an expired access token is "deleted, not flagged", and **nothing deletes one**:
`SessionClerk#deleteAllAccessTokens` runs on sign-out and on a series revocation, never on expiry.

**§8 forecloses it in as many words** — "Redis is not declared, because this version runs no
background job. Every write finishes inside its own request, and nothing here leaves the process."
So the one piece of this feature's processing that the skill would place in a worker is the one
piece §8 declares there is no worker for.

**This is independent support for Q35 rather than a restatement of it.** Q35 was written from
reading §9.6 against §8; the placement skill, applied without reference to either, lands on the
same answer — which makes the §9.6-versus-§8 tension a real one rather than an artefact of how I
read it. Deleting on read was the alternative and is worse: it would turn the authentication hot
path, every operation of every screen, into a write.

**3. Auditing a detected reuse — a post-worker in shape, barred in substance.** It is the textbook
post-worker: a side effect unrelated to the response, wanted after it. **But every identifier that
would make the record useful is barred** — the token, its digest, the `sessionKey` and the member of
staff's address are each a credential or personal data under §7 — and the spec asks for no audit
trail. Recorded as Q34. A post-worker writing a line with nothing identifying in it would be
machinery for nothing.

### What this checkpoint would have got wrong by eye

**Candidate 1 is the trap.** Every instinct says a failed-attempt row is a side effect to be swept
out of the request path, and the skill's post-worker section reads as though it were written for it.
Only §7's use of those rows makes it load-bearing — and that is a fact about the *spec*, not about
the code's shape, which is exactly why the checkpoint insists the decision be made with the skill
and against the requirement rather than by inspection.

## Checkpoint 8 — the security audit, and why it had to run before the merge

**This checkpoint is the reason the backend gate does not merge at 6.** The audit ran over code that
was already committed, lint-clean and passing 876 tests, and it found six defects — two of them
security holes reachable from the internet. The full accounting of what a test could and could not
have caught is its own section below; this records what was changed.

### The six, and what each one actually was

| # | Severity | What | Fixed in |
|---|---|---|---|
| 1 | **MEDIUM** | the staff endpoint allowed **any origin**. `signIn` callable cross-origin with a readable response — credential stuffing relayed through the staff's own browsers | `f427e22` |
| 2 | **MEDIUM** | a session whose refresh half could not be delivered was **minted anyway**: over the framework's unconditional WebSocket channel there is no express response, so the cookie write is a silent no-op and `signIn` returned a working access token while discarding the refresh token | `b0a7cc5` |
| 3 | | the validator capped the password at 72 bytes and the address at **nothing**, so a 300-character address reached an INSERT into `varchar(191)` | `b0a7cc5` |
| 4 | | upload middleware on the audience — 10 files × 10 MB parsed before any resolver or filter — for a feature no section declares, beside a 10 MB JSON body limit for an email and a password. Removed; the JSON cap is now `16kb` | `b0a7cc5` |
| 5 | | a comment asserted §7's limit bounded the work an unknown address can buy. True per address, false in aggregate: §7 mandates address keying, and a caller rotating the address is bounded by nothing | `b0a7cc5` |
| 6 | | `RotatingSessionResult#hasRevokedSeries()` answers true when zero rows were revoked, because the predicate means "attempted without error". No consequence today; recorded so a future caller does not read it as "rows changed" | `b0a7cc5` |

**#2 is the one worth reading twice.** Every existing null-response test was on a *refusal* path;
nobody had written one on a *success* path. The fix refuses the sign-in when the cookie cannot be
written, rather than handing back half a session. Four tests fail without it.

### The CORS decision, and a claim I got wrong twice

The allow-list is read from `STAFF_CORS_ALLOWED_ORIGINS` and always passed to `cors` as an **array**.
Three properties, each verified against the installed library rather than assumed:

- **A missing or misspelled key yields an empty list, never a wildcard.** There is no way to ask for
  a wildcard through this key at all.
- **A deliberate `*` denies everything.** The value is compared element by element against the
  request's `Origin`, which is never the literal `*`.
- **The wildcard comes from omitting the `origin` option entirely** — `cors()` and `cors({})` merge
  the library's own default of `'*'`. Every falsy *value* is safe: `undefined`, `null`, `''` and `[]`
  each send no `Access-Control-Allow-Origin`.

**The third property is a correction.** The re-audit reported that an empty *string* is read as
"allow any", and I wrote that into `.env.development`, `.env.live`, the engine's docblock and a
commit message as fact before testing it. It is false. The code was already right — the engine
always passes an `origin` — but the stated reason for it was wrong in four places, and `62d6a61`
corrects them and says so plainly, because the next reader has no other way to know the claim was
measured. **This is the second time in this feature that a verifier's prose was propagated before
being checked**, and both times the code survived while the explanation did not.

### Three INFO items the re-audit raised, and what became of them

1. **The WebSocket channel bypasses both new HTTP mitigations.** The 16kb JSON cap and the CORS
   allow-list are express middleware; the framework mounts a WebSocket channel unconditionally, and
   `ws` defaults `maxPayload` to 100 MiB. Recorded, not fixed — it is the same channel that produced
   finding #2, and closing it is an engine-level decision this feature does not own.
2. **`.env.live` needed the key declared**, left empty so a deployment fails safe rather than
   silently allowing nothing it meant to allow. Done.
3. **A deliberate `*` denies everything** — surprising enough to debug for an hour. Now commented at
   the key and in the engine.

## Checkpoint 9 — the use cases walked against the built API, and a criterion that was false in production

**Run interactively by the main session, against the merged tree rather than against the plan.**
Every one of §10's eight acceptance criteria was taken in turn and matched to something that would
actually fail if the criterion stopped holding. Seven held. **The eighth was false on every deployed
environment**, and no test in the repository could have failed on it.

### The eight criteria, and what holds each one

| § 10 criterion | What would fail if it stopped holding |
|---|---|
| an address with no account and a wrong password refused identically, neither saying which | `should refuse an unknown address and a wrong password identically`, over the two separate refusals. Giving them different codes fails **11** |
| a session survives a reload, and stops working the moment its holder signs out | `should carry a session across a reload, and refuse it the moment its holder signs out` — the cross-operation test that owns this criterion end to end |
| no operation returns a password or a hash, and none writes one into a log line | **the returning half:** each resolver's `#formatResponse()` compares the **whole** returned object, not `objectContaining`, precisely so an added field breaks it. **The logging half: see below — it did not hold** |
| `signedInStaffMember` without a session is refused and returns nobody | `when the context carries no member of staff` |
| a refresh token expired, revoked or already spent refused identically in all three | `should refuse a dead refresh token identically, whichever way it died`, plus one test per state. Structural rather than agreed: all three reach `updatedCount === 0`, one branch, one message |
| a reused refresh token revokes every token in its series | `should revoke every token in the series a reuse presented` |
| `renewAccessToken` with no cookie at all is refused and returns nobody | `should refuse a call carrying no refresh-token cookie at all` |
| an eleventh failed sign-in on one address inside fifteen minutes refused; a first attempt on another address not; successful sign-ins refused nothing | three tests, one per clause. The third is asserted at **eleven** successful sign-ins where §10 says ten — deliberately one past the threshold the failures use |

**Both use cases were walked too, and both are only half-verifiable here.** "Every screen after that
knows who they are" and "the next person is asked to sign in" are statements about screens, and the
frontend opens at checkpoint 10. What the backend half can show is that a session is carried across
a reload and dies at sign-out, which the cross-operation test does. The rest is checkpoint 18's.

### The criterion that was false: every sign-in wrote an address to a log line

§7's Personal data row is unambiguous — "None of the three is ever written to a log line" — and §10
repeats it for passwords. **`live`, `staging` and `production` declared no `logging` key, and
Sequelize's default when the key is absent is `console.log`:**

    logging: hasOwnProperty.call(this.options, 'logging')
      ? this.options.logging
      : console.log                    // sequelize/lib/sequelize.js:249

Sequelize logs the SQL text, and the address travels in a `WHERE` clause rather than in a bound
parameter. Two statements on the sign-in path carry it — the account lookup, and the `COUNT(*)` that
§7's per-address limit runs on **every attempt**:

    WHERE `email` = 'someone@example.com'

So every sign-in attempt on every deployed environment wrote an address to stdout. Fixed in
`3ac78a0`: `logging: false` on all four environments, with the reasoning at the top of the file so
the next reader does not restore a default that looks harmless.

**Why no test could have caught it, which is the part worth keeping.** `development` already set
`logging: false`, and the whole suite runs under `development` — so the defect lived *only* in the
configuration of the environments the suite never opens. A passing suite was not weak evidence here;
it was no evidence at all. The new test therefore reads the **file** rather than a connection, so it
holds for environments this machine cannot open, and a second test reconciles its enumerated cases
against the file's own keys so a fifth environment cannot be added unchecked.

**And the password half of the same criterion does hold, for a reason worth stating rather than
assuming:** the digest is compared through the encipher in JS, never in a SQL equality, so it never
reaches a `WHERE`; and a single-row insert binds its values rather than inlining them. The criterion
was half true and half false, and only walking each clause separately showed which was which.

### The exit condition

Eight criteria walked, seven held on the first pass, one fixed and re-walked. Both use cases
verified as far as a backend can carry them, with the screen half recorded as checkpoint 18's.
**715 + 195 = 910 before this checkpoint, 720 + 195 = 915 after it**, lint clean, and both CI
dialects green — SQLite and MariaDB, the latter running the same suite on every pull request (Q37).

## Checkpoint 10 — the frontend opened, and a route that was already decided

**The route is `/sign-in`, and that was not a free choice.** The boilerplate already ships
`middleware/000.gateway.global.js`, a global route middleware holding

    const SIGN_IN_PATH = '/sign-in'

which redirects every unauthenticated request to `` `${SIGN_IN_PATH}?redirect=${to.fullPath}` ``.
So §10.2's "every other screen sends somebody here when theirs has gone" is **already true** rather
than something a later checkpoint wires — and any other path would have quietly broken it. The unit
found this by reading the tree rather than by picking a plausible name, which is the difference
between a route that works and one that looks right.

`pages/sign-in/index.vue` is the routable leaf, paired with `SignInPageContext` beside it as a `.js`
sibling that `nuxt.config.js`'s existing `pages:extend` hook strips from the route table. The page
is a title and nothing else — no fields, no validation, no submit. Components are 12, the UI is 15,
the wiring is 16.

### Reachability was established, not asserted

A `nuxt build` was run and the generated route table read out of the client bundle: `path:"/sign-in"`
appears with its page chunk, and **only the two `index.vue` routes appear**, which is what proves the
`.js` sibling really is stripped rather than merely believed to be.

**Curling a running server would have proved less, and the reason is worth keeping.** With
`ssr: false` nitro serves the same SPA fallback for every path, so a `200` on `/sign-in` is
indistinguishable from a `200` on `/nonsense`. The weaker check is the one that looks more like
real verification.

### The endpoint was wrong for this audience in every tracked env file

The boilerplate shipped `http://localhost:3900/graphql-customer`, and `.furo-env.test` pointed at
`/graphql-stub`. Both now read the staff endpoint, verified against the backend rather than assumed
— `StaffGraphqlServerEngine` declares `graphqlEndpoint: '/graphql-staff'` and `server/index.js`
listens that engine on `4900`:

    ENDPOINT_URL   = http://localhost:4900/graphql-staff
    WEBSOCKET_URL  = ws://localhost:4900/graphql-staff

`WEBSOCKET_URL` also gained its field in both `RuntimeConfig` and `PublicRuntimeConfig`;
`plugins/000.furo.js` had always read it off `runtimeConfig.public`, but only `ENDPOINT_URL` was
ever declared. And `.furo-env.development` is gitignored, so it does not exist until somebody copies
the example — `NuxtFuroEnvLoader.loadEnv()` returns `{}` for a missing file **silently**, which would
leave `ENDPOINT_URL` undefined at dev time with nothing said.

### A unit's finding I checked and did not take

The unit reported the tree doc "stale" for saying **Middleware: None** while the repo ships two
global middleware files. **It is not stale — the word is overloaded.** That section reads "A frontend
row holds neither a DB client nor a Redis client, and no compose file is placed here", which is §8's
sense of middleware, the one whose table lists MariaDB. Nuxt route middleware is a different thing.

The doc was right and the reading was wrong, so nothing was corrected. **What was genuinely missing
is different and now recorded:** the tree doc named the directories without saying that one of them
decides this feature's route. `.hora/tree/expense-note-frontend-staff.md` now carries what the
boilerplate ships and which parts are load-bearing, plus the overload itself, so the next reader does
not repeat the misreading.

### Two gaps this checkpoint did not close, both recorded rather than worked around

- **There is no `gateway` layout.** The `hof-nuxt` digest expects an auth page to take one; this
  repository ships only `layouts/default.vue`, a bare `<slot />`. The page uses the default
  implicitly. Creating a layout is checkpoint 12's or 15's, not this one's.
- **`composables/useRedirect.js` is a bare exported function**, which the tree doc's own "shared logic
  is a class, never a composable and never a bare function" rules out. Pre-existing boilerplate,
  untouched, raised as **Q40** — it becomes a decision at checkpoint 16, which is what wires the
  post-sign-in redirect.

**9 suites, 34 tests, lint clean.** Baseline was 8 and 27.

## Checkpoint 11 — the third pass over the use cases, and what it found that the first two could not

**Run interactively by the main session.** Its real output is
`expense-note-frontend-staff/ai/contexts/uiux-context.md`, which did not exist — only the blank
questionnaire ships with the skill. Both the UI generator (checkpoints 12 and 15) and the auditor
(checkpoint 18) read that file, so **an answer written into it becomes both the instruction and the
standard it is later judged against.** Every answer is therefore tagged with where it came from:
`[spec]` with the section named, `[user]`, `[tree]`, or **`[chosen]` — arbitrary, so checkpoint 18
does not audit a preference as though it were a rule.**

### Three answers the spec could not give, put to the user

| | Decided | Why it was not inferable |
|---|---|---|
| devices | **desktop-first, usable on a phone browser** | §4 rules out a phone **app**. That says nothing about a phone **browser**, and the two are different questions |
| accessibility | **WCAG 2.2 AA** | the spec is silent. This is the standard checkpoint 18 fails against, so it had to be stated before 12 generates anything |
| tone | **plain and neutral** | §5 inherits nothing, so there was no house voice to match |

**One thing that looked like a question and was not.** The interface language is **English**, and it
is derived rather than chosen: §6 names the categories "transport, meals, supplies, other", so the
domain vocabulary the UI displays is English. §1's `Question language | English` is about questions
put to the user during specification and is **not** evidence for the interface — reading it that way
would have been a guess wearing a citation.

### The finding: both of §10's use cases end outside this feature

This is the checkpoint that asks whether *a person can actually do the use case on a screen* — 2
asked whether the spec supports it, 9 whether the API does. Walking both:

| §10 use case | The path, as far as it goes | Where it stops |
|---|---|---|
| signs in on a Monday morning, "**and every screen after that knows who they are**" | open any path → the gateway sees no token → `/sign-in?redirect=<path>` → form → `signIn` → token in memory → redirected back | **there is no screen after that.** The only routes are `/sign-in` and `/`, and `/` is still the boilerplate stub with an empty template. The clause is about screens this feature does not own |
| "finishes on a shared machine, **signs out**, and the next person is asked to sign in" | — | **there is nowhere to put the control.** §11.2 puts `signOut` on the **expense-entry** screen, and §10.2's screen is explicitly "for a member of staff who is **not** signed in". `#sign-in` has no screen a signed-in person ever sees |

**This is not a spec defect and was not sent back to checkpoint 2.** The kit says a use case with no
path through the interface goes back, because either the interface or the use case is wrong. Neither
is wrong here: the paths exist at **version** level — §11.2 carries the sign-out control, and the
later screens are what "every screen after that" refers to. What is true is narrower and sharper:
**§10's use cases are not closable at `#sign-in`'s own gate.** They close when `#expense-entry`
exists, or at the whole-version sweep.

**And that corrects something I wrote at checkpoint 9.** The record there says the screen half of
these two use cases "is checkpoint 18's". **It is not** — `#sign-in`'s checkpoint 18 cannot close
them either, because the control and the destination both belong to a feature that does not exist
yet. The right statement is the one above, and Q41 records it so checkpoint 18 does not report a
missing sign-out button as a defect of this feature.

**Why the earlier passes could not have caught it.** Checkpoint 2 read the use case against the
spec, where §11.2's `signOut` row makes the path complete. Checkpoint 9 read it against the API,
where the `signOut` operation exists and works. **Only the question "which screen has the button"
reaches it** — and that question is this checkpoint's alone.

### What the context file records that nothing else does

- **No design tokens exist.** `assets/css/variables.css` declares `:root { }` and a comment saying
  the boilerplate deliberately makes no decision. So there is no answer to "what is the primary
  colour" until a checkpoint declares one — which is why checkpoint 10's page carries no `<style>`
  block at all: with no tokens, any colour would be a literal, and `05-frontend.md` forbids those.
- **Eighteen project UX rules**, lifted from `D:\ORT\rules\05-frontend.md` rather than invented, so
  the generator produces code that passes review instead of code that gets rejected.
- **Three things that must never be built**, each with its spec citation: a sign-up or
  password-reset link (§3 — there is no such screen and no such flow), a "remember me" checkbox (§9.6
  — the access token lives in memory and the refresh token is httpOnly, so it would control nothing),
  and any UI for `renewAccessToken` (§10.2 — it belongs to the client layer and happens invisibly).
- **One copy rule that is a requirement, not a preference:** an unknown address and a wrong password
  are refused **identically**. "No account found for that email" is a defect, and a generator left to
  its instincts writes exactly that.

## Checkpoint 12 — the screen broken into components, and what reading five skills bought

**The exit condition is that every component either already exists or has a stated reason for being
new.** After Q42 was settled the answer is: **four exist, one is new.** But the value of this
checkpoint was not the breakdown — it was that five digests, read against the installed library
rather than the skill prose, found eight things that would each have produced working-looking code
that was wrong.

### The breakdown

| Part of the screen | Component | Status |
|---|---|---|
| the page and its context | `pages/sign-in/index.vue` + `SignInPageContext` | **exists** — built at checkpoint 10 |
| email address input | **`FuroEmailField`** inside **`FuroControlBlock`** | **exists** — furo-vue |
| password input | **`FuroPasswordField`** inside **`FuroControlBlock`** | **exists** — furo-vue |
| submit control | **`FuroButton`**, `variant: 'default'`, driven by `loading` | **exists** — furo-vue |
| the refusal message | — | **NEW.** Reason below |

**The one new thing, and why it has to be new.** §10 requires an unknown address and a wrong
password to be refused **identically**, so the refusal is a single **form-level** message and not a
per-field error. `FuroControlBlock`'s `errorMessages` attaches to one control, and the
`hof-cp-control-block` skill does not cover a form-level message or name anything that does. The
block can be made to render one mechanically — null label, empty slot — but that is unsanctioned
use, so this is a small component of this application's own rather than a library one borrowed
sideways.

### Eight facts that contradict the skills or the library's own manifest

**Every one was verified in the installed source, and every one would have compiled.**

| # | What the skill or manifest says | What the source does |
|---|---|---|
| 1 | a `FuroTextField` can take `type="email"` | **`type` is destructured out of the fallthrough attributes and silently discarded.** Using the dedicated `FuroEmailField` is not a preference, it is the only thing that works |
| 2 | `FuroControlBlock` is a "label, control, **hint** and error wrapper" (manifest) | **there is no hint prop and no hint slot.** The parcel is `label` / `controlId` / `errorMessages` / `required` / `orientation` |
| 3 | `FuroPasswordField` has a "reveal toggle" (manifest) | **it does not** — a bare `<input type="password">`, `"slots": []`. Nobody should design a show-password affordance around it |
| 4 | a button variant named `primary` | **no such variant.** `default | secondary | destructive | outline | ghost | link`; the primary-looking one is `'default'`, and a guess would have rendered unstyled |
| 5 | disabled and loading "set aria-disabled / aria-busy" (manifest) | loading sets **`aria-busy` only**, never `aria-disabled`, while also setting the native `disabled` |
| 6 | — | **a loading button has no accessible name**: its label is `visibility: hidden` and its spinner is `aria-hidden="true"` |
| 7 | — | **`autocomplete` is untouched by the library.** `username` and `current-password` are entirely the caller's to pass, and a password manager needs both |
| 8 | — | **`aria-describedby` is not wired** from the error region to the control it describes — a gap the library's own source comments on |

**Facts 6 and 8 are WCAG 2.2 AA failures that the library hands us**, against the target the user set
at checkpoint 11. Neither is ours to fix upstream and both are ours to compensate for at checkpoint
15: the submit needs an accessible name that survives its pending state, and the refusal message
needs associating with the fields it refuses. Recorded here so 15 builds them in rather than 18
finding them.

**Fact 2 is the second manifest-versus-source disagreement in one library** (with 3 and 5), which is
worth noticing as a pattern rather than three separate surprises: `components.json` describes
intent, and the `.vue` file is what ships.

### The defect this checkpoint actually turned on, found by three digests independently

**Not one of the 52 components had a single working colour, dimension or z-index**, because nothing
imported the library's stylesheet — `nuxt.config.js` loaded only the app's two files, `variables.css`
is an empty `:root {}`, `furo-nuxt` 2.x ships no CSS at all, and furo-vue's Nuxt module installs an
icon renderer and no stylesheet.

**`--color-ring` is the only focus indicator `FuroButton` has.** Its stylesheet removes the native
outline and rebuilds the ring from that property, so undefined it removed the outline and rebuilt
**nothing**: no visible focus for a keyboard user.

Fixed in `2899424` by loading `furo.css` first, and **verified out of the built bundle** rather than
asserted — `--color-ring: var(--palette-blue-500)`, `--palette-blue-500: #3b82f6`,
`--color-destructive: var(--palette-rose-500)`.

**No test could have failed on it, and that is the point worth keeping: an undefined CSS custom
property is not an error, it is an empty value.** Nothing throws, nothing warns, the build succeeds,
the components render. The only way to find it is to ask what a property resolves to, and the only
reason anybody asked is that three digesters were told to verify the skill against the installed
package instead of summarising it.

It also corrected the tree doc, which attributed those stylesheets to `furo-nuxt` and described a
`@layer` system this repository does not have — that is `crm-kit-frontend`'s. The correction is kept
beside what it replaced.

## Checkpoint 13 — one half applicable, one half not, and the kit made me say which

**The exit condition has two halves and the kit is explicit: "State which of the two, do not assume
both."** That instruction did real work here.

| Half | Verdict |
|---|---|
| this feature's backend error codes map to user-facing messages | **applicable.** 13 codes, all mapped |
| logic used by more than one component or page exists as a class under `app/modules/` | **not applicable**, and nothing was built for it |

**The n/a is evidenced rather than asserted.** `#sign-in` has one screen; `components/` holds only a
`.gitkeep`; `pages/index.vue` is still an empty stub; and the feature's other two operations have no
second call site either — `signOut`'s control belongs to §11.2's screen (Q41) and `renewAccessToken`
is transparent client-layer work no screen calls (§10.2). The skill's own trigger is "reusing
general logic across multiple files", and nothing clears it. **`app/modules/` still does not exist,
which is the right outcome**: inventing a shared module to have something to show would be premature
extraction, and the checkpoint explicitly permits n/a.

### The i18n fork, decided from what was already recorded

The skill offers two **mutually exclusive** mechanisms and says an app uses one or the other: a
locale-path variant (`ERROR_LOCALE_HASH` + `t()`) that needs an i18n layer **even for a single
language**, and a static-string variant that needs none.

**Static strings, no new dependency.** Not a fresh judgement — `ai/contexts/uiux-context.md` §7
already records English, single language, no localization layer, and that was derived at checkpoint
11 from §6 naming the categories "transport, meals, supplies, other". The decision was made two
checkpoints ago; this one only had to notice it applied.

### An always-on rule forced a better design than the skill's

The skill's static variant resolves in **two** hops — code → semantic name via `ERROR_CODE_MAP`,
then name → message — and writes `ERROR_CODE_MAP` as a `Map`.

Two rules kill that. `hoc-properties` prohibits `Map` and **names a string-keyed one as the exact
circumvention it is banning**. And the skill's own `dictionaries.md` says "no reverse map", which
`ERROR_CODE_MAP` is. So the hash is keyed by the code directly, one hop — and `ERROR_CODE_HASH`
still earns its place by supplying the computed keys, so no dotted string is typed twice.

**A skill contradicting itself, resolved by a rule that reached both halves.** Category 1 of the
arbitration taxonomy, and the cheapest kind.

### The three messages worth recording

- **`204.M001.001`** — "That email address and password do not match." One code for two outcomes, so
  the message names neither. A reading pins that exact text, so restoring "No account found for that
  email" fails a test rather than surviving review. §10's criterion is now held in three places: one
  `throw` site in the backend, one code in the contract, one string here.
- **`203.M001.004`** — quotes **no figure**. The limit is 72 **bytes**, measured with
  `Buffer.byteLength` because bcrypt truncates there, so "72 characters" is false for any non-ASCII
  password. A helpful-sounding number would have been a lie, and the honest message is vaguer.
- **`204.Q001.002`** — the account row exists, its secret row does not. **The one case where "try
  again" would be a lie**, because a missing row does not heal on a retry. It points at whoever
  issues accounts, which §4 puts outside the product — and not at a screen, because there is none.

### The conflict-proof file, and a rule ESLint does not enforce

`BaseAppGraphqlCapsule` is a base class every operation's capsule derives from, so the unit reported
it rather than editing it and the main session wrote it. It carries the **single** resolution point:
a context that mapped a code to a message itself would be a second place where §10's identical
refusal can quietly stop holding.

`getErrorMessage()` is inherited from furo and **returns a code despite its name** — confirmed in the
base, which answers `null`, one of four transport codes of its own, or the backend's. The name
cannot be changed, so the docblock warns.

**And one reading was written and deleted before committing.** It generated its cases with
`Object.entries(ERROR_MESSAGE_HASH).map(...)`, which `testing.md` bans — "complex case-generating
loops are banned". **ESLint does not catch that one**, lint was clean, and 86 tests passed. It was
removed because the rule says so and because the completeness it was testing already lives in the
reconciliation reading in `constants-error.js`, which is where it belongs. A green suite is not
evidence that a test is allowed to exist.

10 suites, **73 tests**, lint clean. Baseline was 9 and 34.

## Checkpoint 14 — the four clients, and the first exit condition this feature could only half meet

**Reached in part, deliberately recorded as such.** The condition has two clauses and they had
different fates.

| Clause | Verdict |
|---|---|
| a client exists for every operation, **matching `.hora/contracts/1.0.0/` exactly** | **met**, and verified harder than by eye |
| **it works against the stub from checkpoint 4** | **not met, and unmeetable here** |

### The half that was met, and why the verification counts

Each document was extracted **out of the Payload file as it stands on disk** — not transcribed from
a report — and validated against the pinned contract with `graphql@17.0.2`:

```
SignInMutationGraphqlPayload              VALID
SignOutMutationGraphqlPayload             VALID
RenewAccessTokenMutationGraphqlPayload    VALID
SignedInStaffMemberQueryGraphqlPayload    VALID
```

**And the checker was shown to produce the negative answer first**, because four VALIDs from a
validator that always returns VALID would look identical. The first negative control was **wrong and
was corrected**: it tripped on "Variable `$input` is not defined" rather than on the field, so it
proved only that the checker rejects *something*. The corrected control asks for `refreshToken` on
`SignInResult` and is refused with "Cannot query field … Did you mean accessToken?" — the error the
check exists to catch.

**A negative control that fails for an unintended reason is barely better than none**, and it is the
same trap as the push check that could not distinguish "pushed" from "no such ref".

### The half that could not be met

**The backend cannot boot on Windows** — Q24, root-caused this session to a single line in renchan's
`DeepBulkClassLoader`. Nothing listens on a socket, so no client can be driven against the stub.
**Not faked, and no mock server was stood up and called a stub.**

The nearest honest evidence, which is a step short and is recorded as such: the same four documents
also validate against the **backend's own staff schema** — the schema the stub server would serve —
and each stub resolver returns exactly the fields these Capsules read. **No request was ever sent.**

This is the second exit condition this feature has reached in part rather than passed, after §10's
two use cases at checkpoint 11. Both are recorded with the reason rather than rounded up.

### The trap in the three no-argument operations

`signOut`, `renewAccessToken` and `signedInStaffMember` take **no argument at all**, which is the
case a skill's examples skip. furo's `invokeRequestWithFormValueHash` wraps unconditionally into
`{ input: valueHash }` — an `input` those three documents never declare. Each of their Payload tests
asserts `variables` is `{}`, so a later checkpoint sending an `input` fails a test rather than a
runtime request.

**The digest found this by reading the installed generator rather than the skill's prose**, which is
the third time that practice has paid this gate.

### `types/graphql-schema.d.ts` — what it is, since the distinction matters

The file did not exist and now holds the **whole** contract, 23 types. Asked directly whether that is
a second authority for the contract we just agreed to keep single:

**No, but it is a second representation.** It is a type projection — mechanically derived, consumed
only by the type checker, and **unusable for validation**, so nothing can pass against a stale copy
of it the way a vendored SDL would allow. That is precisely the property that made vendoring the SDL
unacceptable and makes this acceptable. It can still drift if the contract moves and nobody
regenerates, and that is its recorded risk.

It holds the whole contract rather than four operations because the generator rewrites it whole:
hand-trimming would guarantee a merge conflict when `#expense-entry` regenerates the same file.

### One forbidden-name standoff, resolved rather than suppressed

`id-denylist` bans `data`; furo's `content` getter requires `data` as a fixture key; and
`quote-props` rejects quoting it to escape. A module-level `RESPONSE_CONTENT_FIELD = 'data'` used as
a computed key **suppresses no rule** and names the thing better than `data` did. Both errors were
reproduced before the workaround was chosen rather than assumed.

**22 suites, 140 tests, lint clean.** Baseline was 10 and 73.

## Where the decisions live, when control flow does not hold them

**Asked for at the gate, and it cannot be recovered from the tree afterwards.** A reader counting
branches in this feature sees a small number and concludes the logic is thin. The opposite is true:
the load-bearing decisions were deliberately moved *out* of control flow, into data, getters,
`where` clauses and hooks — each time for a stated reason, and each time with an alternative that
would have scored better on a branch count and been worse.

| Where | The decision it holds | Why not a branch, and what was rejected |
|---|---|---|
| `generateValidationEntries()` — an array of `[() => boolean, errorClass]` tuples | which checks run, **in which order**, and which error each produces | Order is the decision: presence before format before length, so the first failure is the most specific thing wrong. As `if` statements the order would be implicit in the nesting and a new rule would edit an existing branch. Rejected: one `if` per rule (five branches, OCP violation); a single regex doing all five (one branch, five indistinguishable refusals — and §10 needs the address cases to differ from the password cases) |
| `schemasToSkipFiltering` — a three-element array | **which operations are reachable without a session** | The framework maps listed entries to `null` and calls `filter?.(…)`, so a listed operation gets no `Unauthenticated`, no `Unauthorized`, no `DeniedSchemaPermission`. This array *is* §7's Authentication row, and a wrong entry is a public endpoint. Rejected: the boilerplate's `'*'`, which is what made the audience open (audit finding 1); and a per-resolver opt-out, which spreads one security decision over four files |
| `refuseRejectedPassword()` — **one method, one `throw`** | that two different internal causes are one indistinguishable refusal | §10 requires an unknown address and a wrong password to be refused identically. Both paths funnel into a single method that records the §7 failure and throws `InvalidCredentials` — there is exactly **one** `throw this.errorHash.InvalidCredentials.create()` in the resolver, so the two outcomes cannot diverge by construction, and neither can their effect on §7's count. Rejected: different codes (fails 11 tests, and the criterion). **Corrected at checkpoint 13:** this row previously read "distinct names, one shared code", describing two error names converging on one code. That is not what was built and the truth is stronger — two names sharing a code relies on somebody remembering to write the same string twice, whereas one throw site has nothing to remember |
| `#spendRefreshToken`'s `where` clause — `{ tokenHash, usedAt: null, revokedAt: null, expiredAt: { [Op.gt]: now } }` | **whether a refresh token may be spent at all** | Four conditions evaluated by the database in one guarded `UPDATE`, which a control-flow count reads as **zero**. It is the most security-relevant decision in the feature. A caller-side check would have scored as branches and been *weaker*, because a caller-side check is skippable by construction and this one is not. It is also what makes §10's identical-refusal structural: expired, revoked and spent all reach `updatedCount === 0` |
| `RotatingSessionResult#shouldRollBack()` — `hasError() && !hasRevokedSeries()` | that a rotation has **three** outcomes, not two | A refusal that revoked must commit; a revocation that itself failed must roll back. Written as a predicate on the result rather than a branch at the call site, because the caller cannot see which of the three it is. Rejected: a second transaction and revoking outside the caller's transaction (both deadlock on the same row under `SERIALIZABLE`); splitting the spend into its own committed transaction (breaks spend/issue atomicity) |
| the two rate-limit subclasses' **overridden getters** | §7's two limits — the model, the keyed field, the instant field, the window, the threshold | Base plus two thin concretes rather than one parameterized class, because the two do not differ only in values: the sign-in limit normalizes its key and the renewal limit must not, which is an overridden *method*, not a parameter. Parameterizing would also have put §7's numbers at every call site, so each resolver would restate the spec |
| Sequelize **hooks** — the address normalizers | that an address is lower-cased once, wherever it enters | Not a branch anywhere, and not in any resolver. The trap is recorded at checkpoint 5: `bulkInsert` bypasses hooks entirely, so a seeded address with a capital fails no insert and no test — it just makes the account unfindable. Held by a test that normalizes the way `signIn` will and requires a row back |
| the **renewal limit's absence of a table** | how many times a series renewed in the last hour | Checkpoint 2's finding paying off: each rotation already inserts a §9.7 row carrying `sessionKey` and `generatedAt`, so a series' rows inside the window *are* its count. Rejected: a counter table, which would have been a second source of truth for a fact the data already holds |

**The pattern, stated once.** Every row above trades a branch for a declaration, and in six of the
eight the declaration is also the thing that makes a §10 criterion structural rather than agreed —
true because there is only one path, not because three paths were each written to return the same
answer. That is the property a branch count cannot see and a reviewer should.

## Where an equipped skill and an always-on rule disagreed, and what won

**Captured at checkpoint 8, not at the gate, because this list cannot be reconstructed from the
files afterwards.** A reader of the merged tree sees the outcome and not the disagreement: nothing
in the code records that a skill said otherwise. The evidence lives only in the unit reports, and
those are transient.

**Q10 is the standing ruling** — where an equipped skill and an always-on rule in `D:\ORT\rules\`
conflict, the rule wins. What follows is every time that ruling was actually exercised in
checkpoints 3 to 6, as the units reported it.

**A correction to a figure I gave verbally first: I said "eleven times in checkpoint 6 alone".
The real count is 18 across checkpoints 3 to 6, of which 15 are in checkpoint 6.** I said eleven
from memory rather than from the reports. The larger number is not a better result — it is the
same conflicts, counted properly.

### The five that changed structure, not style

| # | The question | The skill said | The rule said | Built |
|---|---|---|---|---|
| 1 | the validator's run entrypoint | `validate()`, which **throws** | `validateInput()`, which **returns** the error or `null` | the rule's. The skill explicitly punted — "use whatever the actual base names it" — so a rule that decides beat a skill that declined to |
| 2 | where validators live | `app/validator/forResolver/<endpoint>/` | `app/tools/validator/resolvers/<audience>/` | the rule's. Also keeps one `tools/` tree rather than opening a second top-level directory |
| 3 | who calls `.create()` on the error | the entry carries a constructor the base "raises" | the base `.create()`s the first failing entry's error | the rule's; follows from #1 |
| 4 | the error-code family for a credential refusal | `205.*` is **auth** | `205` is **external**; `204` is database, and its own worked example is `OrderNotFound: '204.M018.001'` | the rule's. All of this feature's credential refusals are `204` |
| 5 | SDL layout for a new audience | **unsettled** — the skill says so outright | `schemas/` split by audience as directories | a directory of numbered files. Three features append to this audience, so each adds a file instead of editing a shared one |

### The rest — convention, and each one a real fork

| # | The question | Built, and why |
|---|---|---|
| 6 | `@augments` versus `@extends` | `@augments`. The framework and boilerplate write `@extends`; `jsdoc.md` and every file this project has *written* use `@augments`. Arose in five separate units |
| 7 | GraphQL type declarations | one `types/StaffGraphQL.d.ts`, `namespace server.graphql.staff`, per `graphql-resolvers.md` — against the type skill's one-file-per-resolver under `types/resolvers/` |
| 8 | lifting shared members into the app base engine | duplicated in the concrete engine, as the tree does. The skill advises lifting; `/hora-build` classes a base class as conflict-proof, so lifting would rewrite three existing files so one new one could be added |
| 9 | index-name abbreviation | always `SHORT_COLUMN_NAME`, no length threshold. The migration digest set a ~50-character floor "and not before" |
| 10 | multiple `addIndex` calls | sequential `await`, never `Promise.all` — the digest said the opposite |
| 11 | constant file layout | `.cjs` master plus ESM bridge for §7's shared figures; a module-level constant for the encipher's own cost factor. Two units read one digest oppositely and **both were right** — the distinguishing test is shared category versus one module's knob |
| 12 | member order in a resolver | constructor, then `static create`, then static getters, per `javascript-style.md` — against the mutation digest's `schema`-first list |
| 13 | method order within a resolver | caller above callee, so `validateInput` sits above `createInputValidator` — the digest numbered them the other way |
| 14 | model seam naming | `StaffMemberSecretModel`, not `…Ctor`: `hoc-accessors` reserves the `Ctor` form for a class you instantiate, and a Sequelize model is called statically |
| 15 | case-field names in tests | `params` / `factoryParams` / `expected`. `hoc-jest` says "do not invent names like `params`" — the rule's allowed list is closed and excludes the skill's `input` / `override` |
| 16 | the Act variable | `actual`, not the skill's `received` — except in an inheritance test, where the rule's **own example** writes `received` |
| 17 | `toStrictEqual` | forbidden by the rule, permitted by the skill. Decisive detail: every `toStrictEqual` in this repository is in a file from the initial boilerplate commit, and every test this project has *written* uses `toEqual` — so the tree was not counter-precedent |
| 18 | hoisting fixtures under a `describe` | nothing hoisted; every fixture inside its case, duplication preferred. The skill **mandates** hoisting and the rule bans it |

### Two things worth saying about this list

**Nine of the eighteen are test conventions, and that is not noise.** `hoc-jest` and the always-on
`testing.md` disagree about case-field names, the Act variable, matchers, and hoisting — four
pervasive choices, each touching every test file. A build that followed the skill would have a
visibly different test suite from one that followed the rule, and nothing in either document
announces the conflict.

**The arbitration step is itself a property of this setup.** A build with no equipped skills has
no such step: there is one authority and no reconciliation. So a comparison between this and a
conventional build is partly measuring the existence of the arbitration, not only the style it
settles on. Three of the five structural outcomes — #1, #2 and #4 — would have been *different
code* under the skill, and nothing in the merged tree records that a choice was made.


### The eighteen are one category of three, and the frontend gate found the other two

**Written at checkpoint 12, because the list above is titled for a disagreement that turns out to be
only one of the kinds that happen.** All eighteen entries are *an equipped skill versus an always-on
rule*, and Q10 settles every one of them. That is worth stating precisely, because it changes what
the list is evidence of:

**An arbitration with a standing authority is cheap; an arbitration without one is the only kind
that needs a person.**

Q10 is a standing authority. So once a skill-versus-rule conflict is **noticed**, resolving it is
mechanical — the rule wins, every time, with no judgement exercised. **The entire cost of those
eighteen is detection.** They are not eighteen judgement calls; they are eighteen detections against
a rule somebody had already written. Those are very different products, and conflating them
overstates what the arbitration step did.

The frontend gate produced the other two categories:

| | What it is | Adjudicator | Cost | Instances |
|---|---|---|---|---|
| **1. skill versus rule** | the skill says one thing, `D:\ORT\rules\` another | **Q10** — the rule wins | detection only | 18, listed above |
| **2. skill versus boilerplate** | twenty component skills document `@openreachtech/furo-vue`; `furo-boilerplate-nuxt 2.1.0` ships neither it nor any component | **none.** A boilerplate is not a rule, a skill is not a rule, and neither `specs/` nor `D:\ORT\rules\` says which component library a frontend uses | **a person** | 1 — **Q42** |
| **3. skill stricter than rule** | `hof-prohibits` forbids, in template position, both a chopped ternary and the `.map()`/`.filter()` that `javascript-style.md` positively *requires* over loops | not needed — the strict side is safe | **nothing** | 1 so far |

**Category 2 is why Q42 went to the user rather than into a unit**, and the reason is better than
"it changes the stack": there was nothing to appeal to. No rule reached it, the spec is silent, and
two readings led to materially different work.

**Category 3 is the only one of the three that is a positive finding about the skill set**, and it
is worth counting separately for a reason the other two do not have: an implementer who knew only
the always-on rules would write a `.map()` into a template and be **correct by the rules while being
wrong**. The skill is carrying knowledge the rules cannot express. Free to obey, and nothing detects
it except reading the skill.

### The clearest case of the metric and the quality pointing opposite ways

Kept verbatim because it is the one worth quoting: **`#spendRefreshToken`'s guard was written
narrow at checkpoint 5** — `{ tokenHash, usedAt: null }` — **the hole was found by a unit reading
the guard against §10 rather than by a failing test**, and the fix widened the query to
`{ tokenHash, usedAt: null, revokedAt: null, expiredAt: { [Op.gt]: now } }` **instead of adding a
caller-side check.**

A caller-side check would have scored as branches on a control-flow count and been **weaker**,
because a caller-side check is skippable by construction. The chosen fix is four conditions
evaluated by the database, which a control-flow count reads as zero — and it is the most
security-relevant decision in the feature.

## What was found in code that was already committed, lint-clean and passing

**Every item below was discovered in code that the suite was green on.** The column that matters is
the last one: whether any test could have failed on it. Where the answer is no, the finding is the
case for having a verification or audit step at all — no branch count, no coverage figure and no
passing suite can produce it.

**Tally: of 18 items, 12 could not have been caught by any test.** Three could have been caught by
a test that did not exist. One could only have been caught against a database dialect this project
never tests on. Two were caught by an existing test that was too weak, and are counted in the
three.

### Found by the verifier at checkpoint 3, first pass (verdict: not met)

| # | What | Could a test have caught it? |
|---|---|---|
| 1 | four units shipped with **no tests**, while the tree held a tested sibling for every shape they introduced | **No.** A test cannot fail on its own absence |
| 2 | the README said "the three servers" and "Each GraphQL endpoint answers a health check out of the box" — the second **false the moment this audience opened**, and the exact sentence that would lead the next reader to restore the `healthCheck` Q23 removed | **No.** Nothing points a test suite at prose |
| 3 | `validate-unique-error-code.js` had no `staff/actual/` case | **No** — the gap *was* a missing test |
| 4 | two JSDoc claims the code did not support: a present-tense claim that counting code normalized before querying, when no such code existed; and a circular-dependency justification that was untrue for that pair | **No.** A confidently wrong comment is invisible to execution |
| 5 | the `// TODO: Must fulfill this method.` convention dropped, so a pre-release `git grep` listed three of four unimplemented `findUser`s | **No.** A discoverability convention, not a behaviour |
| 6 | `PaginationInput.sort` declared non-optional, though an omitted GraphQL input field arrives `undefined` | **In principle** `tsc` — but Q31 records that `jsconfig.json` produces 1630 errors of which 1010 are phantom, so in practice **no** |

### Found by the verifier at checkpoint 3, second pass

| # | What | Could a test have caught it? |
|---|---|---|
| 7 | the engine test declared four mocks and two context instances at **describe scope** and fed **one** `cases` array to **four** sibling `test.each` calls — eight tests mutating two shared objects while installing spies on them, on the authentication filter | **No.** It did not fail; it was fragile. Nothing fails until something else changes |
| 8 | `collectMiddleware()` asserted only five `expect.any(Function)`, so a reorder, a swapped parser or a dropped mount all passed | **A test existed and was too weak.** A better one catches it — and now does, position by position |
| 9 | `StaffGraphqlShare.createAsync()` never asserted the broker, so **dropping the broker left the suite green** | **Yes, by a test that did not exist.** Now 6 fail |
| 10 | the README tagline still said "two GraphQL endpoints" — missed by the pass that had just fixed the paragraph and the table eighteen lines below it, and found by grepping for number words | **No.** And it was found by a *different method* than the fix that missed it |

### Found by a unit reading code against the spec at checkpoint 5

| # | What | Could a test have caught it? |
|---|---|---|
| 11 | `#spendRefreshToken`'s guard was `{ tokenHash, usedAt: null }`, so a refresh token **revoked but never spent — exactly what `signOut` leaves** — was marked spent and issued a fresh pair for a dead series. §10's "stops working the moment its holder signs out" was false | **Yes, by a test that did not exist** — 8 now fail without the fix. But it was found by **reading the guard against §10**, not by anything failing. The suite was green with the hole in it |
| 12 | checkpoint 5's own record claimed §7's per-series limit bounded the revocation write on the unauthenticated path. It does not: a refused renewal generates no row, so a replay never advances its own count | **No.** A prose claim in a record, found by a later unit checking it rather than inheriting it |

### Found by the security audit at checkpoint 8

| # | What | Could a test have caught it? |
|---|---|---|
| 13 | **MEDIUM** — the staff endpoint allowed **any origin**. `signIn` callable cross-origin with a readable response; no origin control at all behind `sameSite` | **No.** It was intended behaviour, copied from the boilerplate. A test asserts what you meant, and this is what was meant |
| 14 | the validator capped the password at 72 bytes and the address at **nothing**, so a 300-character address reached an INSERT into `varchar(191)` | **Only against MariaDB** — and the dev dialect is **SQLite, which enforces no `varchar` length at all**, so a 300-character address inserts cleanly there. On this project's actual test setup, **no** |
| 15 | a comment asserted §7's limit bounded the work an unknown address can buy — true per address, false in aggregate, since §7 mandates address keying and a caller rotating the address is bounded by nothing | **No.** Prose again, and the second instance of this exact class |
| 16 | a session whose refresh half could not be delivered was **minted anyway** — over the framework's unconditional WebSocket channel there is no express response, so the cookie write is a silent no-op and `signIn` returned a working access token while discarding the refresh token | **Yes, by a test nobody thought to write**: pass a null response on a *success* path. Every existing null-response case was on a refusal path. Now 4 fail |
| 17 | the audience carried upload middleware — 10 files × 10 MB parsed before any resolver or filter — for a feature no section declares, beside a 10 MB JSON limit for an email and a password | **No.** "This middleware serves nothing declared" is a scope question. Nothing misbehaves |
| 18 | `RotatingSessionResult#hasRevokedSeries()` answers true when zero rows were revoked, because the predicate means "attempted without error" | **No, and no test should** — it has no consequence today. Recorded so a future caller does not read it as "rows changed" |

### Found at the frontend gate

| # | What | Could a test have caught it? |
|---|---|---|
| 19 | **every deployed environment logged an email address on every sign-in.** `live`, `staging` and `production` declared no `logging` key, Sequelize defaults to `console.log`, and the address travels in a `WHERE` clause — including the `COUNT(*)` §7's limit runs on every attempt | **No.** `development` already set `logging: false` and the whole suite runs there. The defect lived only in the configuration of environments the suite never opens |
| 20 | **neither of §10's use cases can be completed on a screen this feature builds** — §11.2 puts `signOut` on the expense-entry screen, and §10.2's screen is for somebody *not* signed in | **No.** Checkpoint 2 asked the same question against the spec and 9 against the API; both answered correctly. Only "which screen holds the control" reaches it |
| 21 | **not one of the library's 52 components had a working colour, dimension or z-index.** Nothing imported `furo.css`; `--color-ring` is `FuroButton`'s only focus indicator, and its stylesheet removes the native outline and rebuilds the ring from that property | **No** — see below. The build succeeds, lint is clean, the components render |
| 22 | seven further source-verified facts contradicting the skills or the library's own `components.json` — `type="email"` silently discarded, no `primary` variant, a hint prop and a password reveal toggle that do not exist, `autocomplete` untouched, a loading button with no accessible name | **No.** Every one of them compiles and renders |

### Three distinct reasons no test could fail, which is not the same as "tests are incomplete"

**The four checkpoints that produced findings produced them for three different reasons**, and the
distinction is worth more than the findings:

| | The mechanism | Instance |
|---|---|---|
| **1. Right where the suite looks** | the code is correct in the environment the suite runs in, and wrong only in the configuration of environments it never opens | the logging defect (19) |
| **2. Right against the referent asked** | the same sentence yields a different answer against the spec, against the API, and against the screen. Each pass answered its own question correctly | §10's use cases (20) |
| **3. Legal but empty** | nothing is malformed, so nothing is detectable | the undefined tokens (21) |
| **4. Tested through a door production does not use** | the suite reaches the code by a different route than the running product does, so the failing path is never entered | the Windows boot defect (Q24) |

**The third has no class of automated detector at all**, and that is the one worth stating plainly:

> **An undefined CSS custom property is not an error, it is an empty value.**

Build, lint, unit tests, type check and audit all detect **malformed** things. A legal-but-empty
value passes every one of them. Fifty-two components had no working colour and nothing anywhere
reported anything.

**And a missing definition that *subtracts* is worse than one that omits.** The library's stylesheet
removes the native focus outline and rebuilds the ring from `--color-ring`. Undefined, it did not
fail to add a ring — it left the removal in place with nothing behind it. An omission degrades to
the browser default; this one degraded past it.

**Mechanism 4 is the one the backend gate could not have found, and it is worth its own paragraph.**
`@openreachtech/renchan`'s `DeepBulkClassLoader` passes a raw absolute path to `await import()`,
which on Windows is `D:\...` — protocol `d:` — and the ESM loader refuses it. **No renchan server
starts on Windows**, and the same line appears in two packages, covering models, GraphQL resolvers,
post-workers and REST routes.

**915 tests pass anyway, for two independent reasons that both have to be true.** Jest supplies its
own module registry and intercepts `import()`, so a specifier Node's loader would refuse is resolved
by jest instead. And **no test exercises `loadClasses()` at all** — the suite reaches models and
resolvers by importing them directly, never through the loader that boots them.

So the suite is not weak here and no extra assertion would have helped: **it is pointed somewhere
else entirely.** A green suite says what it says about the code it runs, and says nothing whatever
about the path production takes to reach that code. Adding tests does not close this class; only
running the thing the way it actually runs does — which is what a live acceptance gate is for, and
which this exercise has never once been able to do.

### The same shape bit the measurement, not just the product

**A method note rather than a finding, because it cost no defect — but it is the third instance of
mechanism 3 in one day and the first where the thing fooled was a check.**

Push state was being verified with

    git log --oneline origin/<branch>..<branch>

and empty output read as "nothing unpushed". **That range yields empty when `origin/<branch>` does
not exist at all**, so *fully pushed* and *never pushed* produce byte-identical output. Four
frontend commits — including the focus-ring fix — read as landed while existing only on one machine.

**It is the focus-ring defect wearing different clothes.** An undefined custom property reads as an
empty value; a non-existent remote ref reads as an empty diff. In both cases the tool answered the
question it was asked, correctly, and the question could not distinguish the two states it existed
to distinguish.

Stated generally, and worth more than either instance:

> **When a check can return the same answer for "satisfied" and "not applicable", it is not a
> check.** Establish existence before comparing.

The corrected form establishes the ref first — `git rev-parse --verify --quiet origin/<branch>`
before `git rev-list --count`, or `git ls-remote --heads origin <branch>` for the remote's own
answer rather than a cached copy of it.

**And the asymmetry is not chance.** Twice in one day, both times a local tree read as a pushed one,
never the reverse. A missing thing looking like an absent difference fails in exactly one direction:
toward believing the work is done.

### The checkpoint that produced no code created the standard a later finding failed against

Checkpoint 11 wrote no code at all. It asked the user for an accessibility target and recorded
**WCAG 2.2 AA**, tagged `[user]`.

**Without that, the missing focus ring is not a defect.** It is a styling gap for checkpoint 15 to
notice or not — no standard, nothing failed, a matter of taste. The answer given at 11 is what makes
it a 2.4.7 failure at 12.

Which is also the argument for the tagging that file carries. **A preference and a requirement look
identical in a document, and only one of them can be failed against.** `[user]` and `[spec]` can;
`[chosen]` cannot, and says so in as many words, so checkpoint 18 does not audit somebody's taste as
though it were a rule.

### What this says

**Three distinct discovery methods produced these, and none of them is a test.**

1. **Reading code against the spec sentence it implements** — items 11, 14, 16. The most valuable, and the only method that found the two real security holes.
2. **Reading prose against the code it describes** — items 2, 4, 10, 12, 15. Five of eighteen, and *nothing* else can find them: a comment, a README and a `.hora` record are all invisible to execution, and all three were wrong in ways that would mislead the next reader into undoing a deliberate decision.
3. **Reading a test against what it would fail on** — items 7, 8, 9. A green test that cannot fail is worse than a missing one, because it reports coverage it does not have.

**And one thing the tests did do, which is worth saying plainly:** every fix above now has a test that fails without it, verified by reverting each fix and reading the failure. The suite could not find these, and it is what keeps them found.
