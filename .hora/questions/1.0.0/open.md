# Open questions — 1.0.0

Append-only. Answered by editing `specs/`, never by editing an entry here.

## Q1. Nothing in the data model holds a session
<!-- spec: data-model, sign-in -->
<!-- blocking: yes -->
<!-- category: unmet-usecase -->

§9 declares three tables — `staff_members`, `expense_categories`, `expenses` — and none of
them holds a session. §10's `SignInResult(staffMemberId)` carries nothing the client can
authenticate with on the next request.

Two of §10's own acceptance criteria cannot be met by the design as written:

- "a session survives a page reload" — nothing is persisted for a reload to find
- "stops working the moment its holder signs out" — a stateless signed cookie could survive
  the reload, but could not be revoked on sign-out without something to revoke against

The reading was put up as a check and confirmed: it is a hole in the spec, not in the reading.

**Which session mechanism to use is a design decision, so it is not settled here.** It reaches
both the data model and the operation list, and it changes the pinned contract:

- a server-side session row keyed by an opaque cookie needs one table, and `signIn` /
  `signOut` / `signedInStaffMember` stay as the three operations §10 declares
- the two-token flow the equipped cookie-authentication convention describes needs an access
  token on `SignInResult`, a refresh-token cookie, its own token tables, and a **fourth
  operation** to renew — which would then need a kind and a caller stated in §10.1 like every
  other operation

**Evidence found in the tree, which settles nothing on its own:** the backend row's
`.env.development` already carries `AUTH_REFRESH_TOKEN_TTL_DAYS`, `AUTH_COOKIE_SECURE`,
`AUTH_COOKIE_SAME_SITE`, `AUTH_COOKIE_PATH` and `AUTH_COOKIE_DOMAIN`, plus a comment stating
that the access-token lifetime is a fixed 15-minute module constant. That is the two-token
flow of the equipped cookie-authentication convention, filled in during setup.

**So the environment layer has taken a position the spec has not.** This is recorded as
evidence for whoever answers, and it is deliberately not read as the answer: which mechanism
the product uses is intent, and intent is not inferred from a file somebody filled in
(`../../../.claude/skills/hora/references/structure.md`, invariant 2). If the two-token flow
is the decision, §9 still needs its token tables and §10.1 still needs a fourth operation with
a kind and a caller.

**Routed to `/hora-spec`, stage 4 (data, API and execution).** `/hora-plan` does not write a
design decision into `specs/`.

- [ ] unresolved

## Q2. `amount` and `totalAmount` are typed `Int!` in the contract
<!-- spec: expense-entry-operations, monthly-summary-operations -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

The equipped GraphQL schema convention types money and decimal fields `String!`, emitted via
`BigNumber#toFixed(2)`. That rule exists to protect decimals, and this product has none: §9.3
declares `amount int`, and §4 permanently excludes a minor unit and any second currency.

- [x] resolved
      Decided in conversation: `Int!`. `String!` would send `"1200.00"` for a currency with no
      minor unit, and every screen would strip the decimals back off. The departure from the
      house rule is deliberate and recorded here so it is not read as an oversight.

## Q3. `spentOn` is typed `String!` as an ISO `YYYY-MM-DD` date
<!-- spec: expense-entry-operations, monthly-summary-operations -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

§9.3 declares `spent_on date` — "the day the money was paid, not the day it was recorded" —
so there is no time of day to carry. The convention's declared `DateTime` scalar would carry
a midnight nobody set.

- [x] resolved
      Decided in conversation: `String!`, ISO `YYYY-MM-DD`. §12's month-boundary criterion
      turns on which day an expense falls on, and a timezone shift applied to an invented
      midnight can move that day across a month boundary. A dedicated `Date` scalar was
      offered and not taken — it adds a scalar and a serializer both repositories must agree
      on, for a field that is already unambiguous as text.

## Q4. The expense row exposes `status` in 1.0.0
<!-- spec: expense-entry-operations -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

§4's approval seam states that an expense "carries a status from the first migration, holding
one value now, and every read goes through it". Whether the *contract* exposes that field to
the frontend in this version is a separate decision from whether the column exists.

- [x] resolved
      Decided in conversation: expose it now. 1.1.0's approval feature then adds no field to
      the row — it only adds values the field can hold. The frontend ignores the single value
      `recorded` for now.

## Q5. The rest of the contract was derived from the parenthesized contents
<!-- spec: sign-in-operations, expense-entry-operations, monthly-summary-operations -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

§10.1, §11.1 and §12.1 name every input and result with their contents in parentheses
(`ExpensesInput(pagination)`), which is a shape to derive rather than an unknown one to
invent. Recorded here is what was derived and how.

Derived from the equipped GraphQL schema convention, and from the SDL already in the backend
row (`server/graphql/schemas/<audience>.graphql`, one file per audience, `healthCheck` only):

- `<Operation>Input` / `<Operation>Result` one-to-one per operation, never shared between two
- non-null `!` everywhere except a genuinely optional field — `memo` alone, per §9.3
- ids typed `Int!`, following the convention's own entity example, though the columns are
  `bigint`
- `pagination` is the convention's `PaginationInput!` / `Pagination!` pair — `limit`, `offset`
  and `totalRecords` non-null, `sort` nullable
- the expense row is one reused type, `Expense`, named after §6's own term. `ExpensesResult`
  and `MonthlyExpensesResult` both carry it, which is what keeps §12's "the total equals the
  sum of the entries shown" checkable against one shape
- `expenses` sorts most recent first, from §11.2's "their own entries, most recent first"
- `timestamps` are the convention's `DateTime!` scalar
- **the three operations the spec writes as `SomethingInput()` take no argument at all.** The
  convention gives an operation with no input no argument, and it prohibits a
  single-character key — so no empty input type is declared and no placeholder field is
  invented to make one legal
- **the `Sort` / `SortInput` pair is `key` / `direction`**, taken from the framework's own
  `RequestSort` in the backend row rather than invented. No operation in 1.0.0 lets the
  caller choose a sort, so the pair is declared for the convention's pagination shape and
  left unused

**Not derived, and deliberately left out: everything the session mechanism decides** (Q1).
The contract file marks that surface open rather than guessing at it.

- [x] resolved
      Recorded. Nothing above was invented; each line traces to the spec or to the convention.

## Q6. Neither implementation row has a `main` branch
<!-- spec: none -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

Both rows were set up with a fresh `git init` and opened straight onto `release/1.0.0`, so
neither `expense-note-backend` nor `expense-note-frontend-staff` has a `main` branch — not
locally, and not on origin. `origin/HEAD` points at `release/1.0.0` in both.

Nothing is wrong today: branching a freshly-initialized row from its current `HEAD` is the
documented path, and the hora repository itself does have `main`.

Two consequences, both at merge time rather than now:

- **the eventual merge has no target.** `release/1.0.0` merges into `main` in every declared
  row before the hora repository may merge at all
- **the merge-order check cannot answer.** `git -C <row> fetch origin main` errors rather than
  returning the 0 or 1 that check reads, so "has this row been merged?" is unanswerable as
  written. It is not the "no remote configured" case the rule covers either — the remote
  exists and holds one branch

Raised now rather than at the merge, because the fix is somebody creating `main` on each row
(from the boilerplate commit, most likely) and that is cheapest before there is history to
reconcile.

- [x] resolved
      `main` created in both rows at each boilerplate commit and pushed, in this session:
      `ec7ec25` in the backend, `beb2fa6` in `frontend-staff`. `release/1.0.0` is exactly one
      commit ahead of it in each, so the eventual merge fast-forwards.

      **One correction to the reasoning this was fixed on.** It does not mirror the hora
      repository, as was claimed when the fix was proposed: there, `main` is the initial commit
      and the empty `Release 1.0.0` marker sits **after** it, on the release branch. In both
      rows the marker is the **root** commit, so a `main` at the boilerplate commit has that
      marker as an ancestor. Harmless — `main` is an ancestor of `release/1.0.0` either way —
      but the branch-opening marker does not sit where `commits.md` puts it, because the
      hand-done setup committed it before the boilerplate rather than after.

      Both checks now answer instead of erroring: the merge-order check returns 1 (not yet
      merged, correct) and the hotfix check returns 0 (nothing new on main) in each row.

## Q7. `.hora/` commits reach `release/1.0.0` without a pull request or CI
<!-- spec: none -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

Pushing this session's `.hora/` commits to `release/1.0.0` made GitHub report:

```
remote: Bypassed rule violations for refs/heads/release/1.0.0:
remote: - Changes must be made through a pull request.
remote: - Required status check "test (20.x)" is expected.
```

**Nothing was violated.** The rules come from active organization-level rulesets on
`openreachtech` — `Approve (Bypass for admins only)`, `Approve (Bypass for all members)`,
`CI (Bypass for admins only)`, `Main Guard`, `Trunk` — and their names state that bypassing is
granted to members and admins. GitHub prints "Bypassed rule violations" whenever a permitted
actor bypasses, so this is the configuration working as set up.

**What is undecided is whether it should keep happening.** The organization's default for a
branch is a pull request plus a green `test (20.x)`, and `commits.md` has `.hora/` committed
straight to `release/<version>` at each gate boundary. Both are deliberate, and they disagree.
Every gate of every feature will land the same way — commits with no review and no CI run.

Two readings, neither recommended:

- **it is fine as it stands.** `.hora/` holds no application code, so a CI suite has nothing to
  say about it, and a pull request per gate boundary would be review theatre
- **it should go through a pull request like anything else.** The rulesets are organization
  policy rather than this project's choice, and a bypass that becomes routine stops being
  noticed

Raised because a bypass notice that nobody reads is the same as no rule.

- [x] resolved
      **The organization's process wins.** Decided by the user, in their own words: "git thì
      vẫn theo quy trình, làm xong tạo nhánh merge release, sau đó làm PR, tin nhắn slack để
      review thì tốt hơn."

      So from here on, `.hora/` landings included:

      - every change goes on its own branch, cut from `release/1.0.0`'s tip, under the names
        `commits.md` already gives (`feature/<feature-id>`, `install/`, `update/`, `retake/`).
        A `.hora/`-only landing at a gate boundary is named for what the gate produced
      - a pull request into `release/1.0.0`, its body in the ORT shape — `# Why` with
        `* Close #n` where an issue exists, then `# How` as bullets
      - `test (20.x)` runs and a review happens. **Nothing merges through the bypass**
      - one Slack message per pull request, handed to the user to post

      This commit is the first to follow it: raised on `release/1.0.0`, then moved onto
      `update/questions-with-q7` and opened as a pull request rather than pushed.

      **This is not a departure from Hora Kit, though it first looked like one.** `commits.md`
      has `.hora/` "commit to `release/<version>` directly", which reads as *needs no
      `feature/` branch* rather than *may bypass the trunk's own process* — and its "Merging
      into a trunk branch" says the merge itself is not the kit's to state, because it belongs
      to the project's own git conventions. So the pull request fills a blank the kit
      deliberately left. Nothing about the kit is wrong here, and nothing was filed.

      **A work branch still prints the notice, and that is the configuration, not a failure of
      this process.** There are three rulesets, and they do not target the same refs:

      | Ruleset | Targets | Rule | Bypass |
      |---|---|---|---|
      | `Approve (Bypass for admins only)` | `main`, `dev`, `staging`, `release/**` | pull request | admins |
      | `CI (Bypass for admins only)` | `main`, `dev`, `staging`, `release/**` | required status checks | admins |
      | `Approve (Bypass for all members)` | **`~ALL`** | pull request | **all members** |

      The third one covers every branch, so pushing a second commit to a work branch trips it —
      a branch cannot be pushed to "through a pull request". Its bypass is granted to all
      members precisely so ordinary branch work is possible. **Creating a branch does not trip
      it; updating one does**, which is why the first push of a branch is silent and the next
      is not.

      **What the decision actually buys is the first two.** Those guard `release/**`, their
      bypass is admin-only, and they are the ones a direct push was skipping: no review, and no
      `test (20.x)`. Landing through a pull request satisfies both. A notice printed while
      pushing a work branch is noise from the `~ALL` ruleset and means nothing was skipped.

      **It reaches every branch, not only a `.hora/` one.** Clarified by the user after the
      first answer: "mỗi lần mà làm xong 1 tính năng, có thay đổi sẽ phải tách từ nhánh release
      ra thành nhánh feature, xong tạo merge và gửi slack… nếu trong commit.md mà k nhắc gì đến
      chứng tỏ là mình sẽ cần follow theo quy trình đó vì đấy là chung của cả công ty."

      So `commits.md`'s "merged back into it" is **not** permission to merge locally and push.
      A `feature/<feature-id>` branch reaching its gate boundary — checkpoint 9 in the backend
      row, checkpoint 17 in a frontend row — opens a pull request into that row's
      `release/<version>`, waits for CI and a review, and gets a Slack block handed over. The
      same holds for an `install/`, `update/` or `retake/` branch.

      **What stays the kit's, and wins wherever it speaks:** the branch names, the
      `Release <version>` marker, the gate-boundary timing for `.hora/` (after 2, 9, 17 and
      18), `.hora/` never sharing a commit with implementation, and app merging into `main`
      only after every declared row has.

## Q8. §9's credential cluster is stricter than the equipped skill, on purpose
<!-- spec: data-model -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

**Read this before "correcting" §9 against `hor-cookie-authentication`.** The equipped skill and
§9 disagree, the skill is the incomplete one, and a later session reading a published skill
against a spec somebody wrote will resolve it the wrong way round unless told otherwise.

### What the equipped skill (0.1.0) gets wrong

- `references/migrations.md` lists four tables and **omits `<actor>_secrets` and
  `<actor>_secrets_bk` entirely**
- `references/token-models.md` names `<Actor>Secret` in the cluster list but does not say it
  takes the backup mixin, gives it no columns, and shows no code for it
- so `signIn`'s own sample calls `findPasswordHashByEmail({ email })` while no table the
  reference lists holds an `email` column

Reading the two references against each other made `<Actor>Secret` look like a phantom. It is
not: it is real, every actor has one, and it takes the backup mixin like the password hash.

### Where §9's nine tables actually came from

**Two running implementations, not the reference** — `hora-simple-crm-jiro-backend` and
`simple-file-management-backend`, both of which carry `customer_secrets`, `customer_secrets_bk`,
`admin_secrets` and `admin_secrets_bk` with models to match. `<actor>_secrets` there is
`<actor>_id` BIGINT not null, `email` STRING(191) not null, `saved_at` DATE(3) not null, plus
`id` and the timestamps from the migration factory.

**That is also what settled `saved_at`.** It sits *beside* `created_at` / `updated_at` rather
than replacing them — the real migration spreads the factory's timestamps right after it. The
reference's column lists name auth-specific columns only, which is why `id` never appears in
them either.

### The two places §9 is deliberately stricter, and must stay

| | The running projects | §9 | Why §9 wins |
|---|---|---|---|
| `staff_member_secrets.staff_member_id` | plain index | **unique** | the table is 1:1 — its history is in §9.8 — and this project's foreign-key rule gives a 1:1 satellite a unique index |
| `staff_member_secrets.email` | **no index at all** | **unique** | §9's own acceptance criterion says two members of staff can hold the same address in no circumstance. Only an index makes that true. In those two projects, that criterion would be false |

**Neither is a mistake to reconcile toward the implementation.** Both are written into §9 as
decisions with their reasons, so the migration checkpoint 3 writes will differ from the CRM's
on purpose.

### Fixed upstream, not yet published

`hora-skills-ort-renchan` issue **#36**, fixed by **PR #37**, merged into that repository's
`release/0.2.0`. Its diff adds both secrets tables to the migration list, gives `<Actor>Secret`
its own section declaring the mixin, and states the rule the reference had never written down:
**both credential tables take a backup mixin; the token tables take none**, because a token is
deleted or revoked rather than rewritten. That last line is why §9.6 and §9.7 have no `_bk`.

**This project pins `^0.1.0`, so none of that is equipped yet.** Until the pin moves:

- **the spec is the authority on the credential cluster, and the equipped skill is known
  incomplete on this point.** A checkpoint finding them in disagreement must not rewrite the
  spec
- checkpoints implement from `specs/`, not from the skill, so nothing is blocked by the delay

### Follow-up, once 0.2.0 is INSTALLABLE — which is later than published

**0.2.0 was published on 2026-09-07, and this project still cannot pin it.** `.npmrc` sets
`min-release-age = 7`, so npm refuses a version until it is seven days old — **2026-09-14** for
this one. The original wording of this follow-up said "once 0.2.0 is published", which is the
wrong condition and would have had somebody try the bump and read npm's refusal as a broken
registry rather than as the quarantine working. The condition is **published plus seven days**.

Move this project's pin from `^0.1.0` to `^0.2.0` then, and **delete this question's "not yet
published" and "stricter than the skill" halves** rather than leaving them to confuse — the
omission will be gone, and only the two deliberate index divergences will still be worth
recording. Tracked on the reviewing session's side as well as here, so it does not live in one
head.

- [x] resolved
      Recorded. §9 carries the nine tables; the divergences are deliberate and written as
      decisions in §9 itself.

## Q9. §9's referential-integrity criterion is the model's work, not the resolver's
<!-- spec: data-model -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

Found at `#data-model`'s checkpoint 1, reading §9's criteria closely enough to build from.

**"an expense always names a member of staff who exists, and a category that exists"** cannot be
met by a migration. This project forbids DB-level `references` constraints — a foreign key is a
plain `BIGINT` with an index, and integrity is enforced in application code
(`hor-cookie-authentication`'s `migrations.md` restates it for the credential cluster). So the
criterion needs code, and which code decided whether checkpoint 1 could pass at all:

- **the model's** — then a test can assert a bad id is refused with only `#data-model` built,
  and the criterion is checkable at its own feature's gate
- **the resolver's** (`recordExpense`, `#expense-entry`) — then it is unobservable until a
  feature built later exists, which is a `forward-reference` and would have gone back to
  `/hora-spec` at stage 2

- [x] resolved
      Decided in conversation: **the model validates it.** The `Expense` model checks on write
      that the owner and the category exist, so §9's criterion stays where it is and is tested
      at checkpoint 6 against `#data-model`'s own code.

      Two consequences worth having on record. There is an existence query on every expense
      write — accepted at this size (§7: 20 members of staff, 50 foreseen). And a
      both-places option was offered and declined: having the resolver re-check what the model
      already refuses states one rule twice, which is what the charter's one-piece-of-
      information rule exists to discourage. `#expense-entry`'s resolver surfaces the model's
      refusal rather than repeating the check.

## Q10. Where the equipped skills and the always-on ORT rules disagree, the rules win
<!-- spec: none -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

Found at `#data-model`'s checkpoint 3, from the digests of the matched skills. Three direct
contradictions between `hora-skills-ort-*` and the always-on rules in `D:\ORT\rules\`. Both are
ORT-authored, so neither is self-evidently the mistake.

| | The equipped skill | The always-on rule |
|---|---|---|
| several `addIndex` calls | `hor-sequelize-migration`: "One index: `await` it directly. Multiple: wrap them in `Promise.all([...])`" | `migrations-and-seeders.md`: "**Indexes: sequential `await queryInterface.addIndex(...)` — never `Promise.all`**" |
| index names | keep the full `COLUMN_NAME`, shorten only past ~50 characters | always build from an abbreviated `SHORT_COLUMN_NAME` |
| a subclass's JSDoc | `hoc-jsdoc` uses `@extends` | `jsdoc.md` uses `@augments` |

- [x] resolved
      Decided in conversation: **the always-on rules win.** They are this machine's standing
      standard, loaded into every session, and stated as overriding. So this project writes
      sequential `await`s, abbreviated index names, and `@augments`.

      **Recorded because the consequence is a false-looking mismatch.** Anyone comparing these
      nine migrations against `hor-sequelize-migration`'s own examples will see `Promise.all`
      there and sequential awaits here, and the natural conclusion is that ours is wrong. It is
      not. The same holds for the index names.

      **Amended at the first unit, because "always" turned out to have a reach nobody
      intended.** Unit 2 applied the rule literally and produced
      `expense_categories_n_unique` — the single-word column `name` abbreviated to `n` — then
      flagged it rather than deviating quietly. Every example the rule gives is a long
      composite (`customer_cart_id` → `cci`, `path_group_close_status_target_value_id` →
      `pgcstvi`), and the rule states its own purpose: it "keeps index names within the DB
      identifier length limit". A one-letter abbreviation serves that purpose not at all, and
      the equipped skill names this exact outcome a mistake.

      **So: abbreviate a composite column name, and leave a single word whole.**
      `staff_member_id` → `smi`; `name`, `email` and `status` stay as they are. That reads
      "always, for the reason given" rather than "always, literally", and it is recorded here
      so nobody re-litigates it as a deviation from Q10 rather than part of it.

      Two further divergences the digests found, neither acted on here because neither reaches
      `#data-model`:

      - `hoc-jsdoc` carries no Sequelize `/** @type {*} */` cast idiom, and the `@typedef`
        placement and `<Class>Params` naming rules exist in it only under headings marked
        *frontend only*. All three live in the always-on `jsdoc.md` instead, which is what this
        project follows
      - `hor-type-interface` declares resolver types in `graphql.<category>` under
        `types/resolvers/`, while the always-on `graphql-resolvers.md` uses
        `server.graphql.<audience>` in `types/<Audience>GraphQL.d.ts`. **This one lands on
        `#sign-in`'s checkpoint 3, not here**, and is left for that feature to settle

## Q11. §9.2 omits two columns the master-table convention asks for
<!-- spec: data-model -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

Found by unit 2 of `#data-model`'s checkpoint 3, reading `hor-database-design` against §9.2.

The convention says a reference master table carries `id`, `name`, `display_name`,
`display_order` and `is_active` — `name` being the machine-facing system key and `display_name`
the user-facing label, free to reword or localize without touching logic. **§9.2 declares only
`name` and `display_order`.**

Two consequences, and the second is the one that matters:

- with no `is_active`, retiring a category means deleting a row that expenses still point at
- with no `display_name`, `name` holds `transport / meals / supplies / other` and **the label a
  member of staff sees has to come from somewhere other than the database** — in practice four
  hard-coded strings in the frontend

**That last point sits awkwardly against §9.2's own seam claim**, which says making categories
operator-editable later "adds a screen and changes no other table". Adding a display label later
*would* change this table, so the claim is overstated as written.

- [x] resolved
      Decided in conversation: **raise it, do not fix it now.** Checkpoint 3 builds what the
      spec says — two columns — and this entry is the record.

      **Where it bites is the frontend gate**, when a screen needs four labels and finds none in
      the database. It can be settled before then with the finding already on record, and
      deciding it now would cost a stage 4 re-entry, a spec pull request and unit 2's three
      files redone, for a gap that changes nothing about the backend.

      Note for whoever settles it: the honest options are to add both columns, or to correct
      §9.2's seam sentence so it stops claiming more than the design delivers. Leaving both as
      they are means the sentence stays wrong.

## Q12. The backend row ships no base model class, and every model needs one
<!-- spec: none -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

Found by unit 2 of `#data-model`'s checkpoint 3, and it blocked the checkpoint for every unit
at once.

`hor-sequelize-model` is unambiguous: a model extends **`BaseAppRenchanModel`** at
`sequelize/baseModel/BaseAppRenchanModel.js`, which extends `RenchanModel` from
`@openreachtech/renchan-sequelize` — and *"a model must never extend Sequelize's `Model`
directly, nor even `RenchanModel` directly"*, because routing every model through one app-level
base is what lets shared behavior be added in one place.

**renchan-boilerplate 1.11.0 ships no `sequelize/baseModel/` directory at all.** Nothing in
`.hora/tree/expense-note-backend.md` recorded its absence, and no unit was assigned it — so
without this, all nine models fail to resolve their import and `sequelize/_.js` fails at
activation.

- [x] resolved
      **The implementer reported it rather than creating it, which is correct** — a shared
      ancestor class is the conflict-proof case, not something a unit writes on its own
      (`hora-build`'s "Conflict-proof files are reported, not written directly"). The main
      session created it: `RenchanModel` subclass, empty body, nothing shared to put in it yet.

      Recorded for two reasons. It is a gap in the **boilerplate** rather than in this project,
      so the next row created from renchan-boilerplate 1.11.0 hits it identically. And
      `.hora/tree/expense-note-backend.md` says `sequelize/models/` ships empty without
      mentioning that the base class those models must extend is missing too — worth adding
      when that record is next rewritten.

## Q13. A row created on Windows loses the executable bit on every shell script
<!-- spec: none -->
<!-- blocking: no -->
<!-- category: lacked-environment -->

Found when CI failed on every branch of `expense-note-backend`, including branches whose only
change was a YAML comment.

`package.json` invokes `./test.sh` and `./test-live.sh` directly, and on a Linux runner the
mode bit decides whether that is permitted. Both test jobs died at
`sh: 1: ./test.sh: Permission denied`, exit code 126; only ESLint passed.

**The boilerplate is not at fault.** `renchan-boilerplate` records `100755` for both test
scripts. This repository recorded `100644` from its initial commit, because `core.fileMode` is
`false` here — git's **default on Windows**, where git can neither see nor record an executable
bit. `docker.sh`, written later by hora on the same machine, never had it either.

**What makes this worth a general entry rather than a local fix.** Nothing looks wrong on the
machine that creates the row: on Windows the bit is meaningless, every script runs, and the
tree looks correct. The failure appears only on the runner, in a job whose error names a
permission rather than a missing setup step — a long way from the cause. **Any hora row created
on Windows has this, on every shell script its boilerplate ships.**

- [x] resolved **for this repository**
      `git update-index --chmod=+x test.sh test-live.sh docker.sh` — it writes the index
      directly rather than reading the filesystem, which is why it works while `core.fileMode`
      stays `false`. Backend PR #3, on its own branch off `release/1.0.0` because it blocks
      every other branch. `git ls-files -s` now reports `100755` for all three.

      **Unresolved in general, and this is the half worth carrying.** A `.gitattributes` cannot
      fix it — git has no attribute for the executable bit. So one of two things has to change,
      and the choice belongs to whoever owns the row-creation step rather than to this project:

      - **row creation sets the mode explicitly** after cloning a boilerplate, for every `.sh`
        it ships. Fixes it once, at the source, for every future row
      - **CI invokes the scripts as `bash test.sh`** rather than `./test.sh`. Makes the mode
        irrelevant, but has to be done in every repository's workflows rather than once

      Also recorded in `.hora/tree/expense-note-backend.md`, since that file lists these
      scripts and a reader would otherwise have no reason to suspect them.

## Q14. A test that says "when the env value is unset" never unsets it
<!-- spec: none -->
<!-- blocking: no -->
<!-- category: upstream-defect -->

Found when PR #3 restored the executable bit and six previously-invisible test failures
appeared. Two defects, stacked, and one had been hiding the other.

### Ours, and fixed

`2705bfb` changed `.env.development` from `AUTH_COOKIE_SECURE=` to `AUTH_COOKIE_SECURE=false`.
The engine's getter reads `env.AUTH_COOKIE_SECURE !== 'false'`, so that turns the flag off, and
six boilerplate tests across three engine suites assert it stays on. Restored to the upstream
value (backend PR #5).

**Decided under the authority granted for implementation choices**, on three grounds: the
engine's own comment says *"Turning it off is for plain-HTTP verification hosts alone"* and
local development is not one, since browsers treat `http://localhost` as a secure context and
accept `Secure` cookies there; §7 puts TLS in front and asks nowhere for the flag off; and
`tests/__tests__/` is 14 suites and 139 tests green with the value restored.

**What was deliberately not done: changing the assertion to `false`.** That would convert a
test of the contract into a test of whatever the env file happens to say.

### Upstream's, recorded not fixed

The test's `describe` reads *"to keep the secure flag on when the env value is unset"* — and
**the test never unsets anything.** No `AUTH_COOKIE_SECURE` reference, no `process.env`, no
spy. It reads the ambient `.env.development` and depends on that file leaving the value blank.

So the assertion is true only by coincidence of configuration. **Any project created from
renchan-boilerplate that legitimately sets `AUTH_COOKIE_SECURE` breaks six tests in files it
did not write**, and the failure message points at the engine rather than at the env that
caused it. The honest form stubs the value inside the test.

- [x] resolved **for this repository**, and left open upstream
      This is a defect in renchan-boilerplate 1.11.0 whatever this project chooses, and it sits
      beside Q13: both are the same shape — **the boilerplate is correct on the machine it was
      written on**, and breaks elsewhere for reasons its own error messages do not name. Q13 is
      the executable bit lost on Windows; this is a test that reads ambient configuration and
      calls it "unset".

      Neither is this project's to fix upstream, and both are worth carrying to whoever owns
      the boilerplate.

## Q15. The npm scripts cannot run on Windows as written
<!-- spec: none -->
<!-- blocking: no -->
<!-- category: lacked-environment -->

Found at `#data-model`'s checkpoint 5, initializing the local database so the seeder's test
could run.

`npm run db:refresh` fails with `'export' is not recognized as an internal or external
command`. The script begins `export NODE_ENV=development; ...`, and npm on Windows spawns
`cmd.exe`, which has no `export`. **The same applies to `dev`, `test` and `test:live`** — every
one of them opens with `export`.

So on Windows, none of the following work as documented: refreshing the database, starting the
development server, or running either test suite through npm.

**Same family as Q13.** The row was created on Windows and its scripts assume a POSIX shell,
exactly as Q13's shell scripts assumed a POSIX file mode. Neither is visible on the machine
that wrote them: on Windows `export` fails loudly but only when somebody runs it, and the
executable bit failed silently until CI.

- [x] resolved **as a workaround, not a fix**
      The local SQLite database was initialized by running the underlying commands through
      bash directly — `rm -f sequelize/storage/*.sqlite3`, then `npx sequelize-cli db:migrate`,
      then `db:seed:all` against `dev-master/` and `development/` in turn, with `NODE_ENV`
      exported in the bash session rather than by the script. That works because this session
      has a POSIX shell available; it is not a fix for anyone whose only shell is `cmd.exe`.

      **The real options, and the choice is not this project's to make** — the scripts come
      from the boilerplate:

      - **`cross-env`**, or the equivalent, so a script sets its variable portably. One
        dependency, and every script keeps its shape
      - **`"script-shell": "bash"`** in `.npmrc`, which makes npm use bash on every platform.
        No dependency, but it requires bash to be installed and silently changes how every
        script in the project is interpreted
      - **leave it**, and document that this row is developed on POSIX only

      Worth carrying to whoever owns the boilerplate alongside Q13 and Q14. All three are the
      same shape: **correct on the machine they were written on.**

## Q16. `.env.live` is tracked in git, and it is the file production values go into
<!-- spec: none -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

Found by `#data-model`'s checkpoint 8 audit, in files **outside** the change set it was auditing.
Recorded so that checkpoint's clean result is not read as a clean bill for the repository.

`git ls-files` reports both `.env.development` and `.env.live` as **tracked**, and
`.gitignore:5` carries only a bare `.env`, which matches neither variant.

**No secret is exposed today, and saying otherwise would overstate it.** `.env.development`
holds local-container values (`password` as a database password against a container published
on `127.0.0.1`), and `.env.live` ships with **every value empty** — verified. Both are
templates, and tracking a development env file with local-only values is a common and
defensible choice.

**The exposure is latent, and it is `.env.live` specifically.** That is the file a deployment
fills with real production credentials, and it is tracked — so the first person to fill it in
commits them, with nothing in `.gitignore` to stop it and no error to warn them. The failure
mode is a single ordinary `git add`.

Alongside it: `sequelize/config.cjs` carries hardcoded credentials in its `live` and `staging`
blocks and declares no `ssl` / `tls` option on any non-local connection. The `live` values are
the local container's and match `docker-compose.development.yml`; `staging` points at a
placeholder host. Neither is a production secret today, and the `live` block is what this
project's own MariaDB verification connects through.

- [ ] unresolved
      **Not this feature's to fix** — `#data-model` touches neither file, and both come from the
      boilerplate. Recorded because the audit's "no HIGH, no MEDIUM" verdict covers the 42
      files it was handed and nothing else, and a reader could take it more broadly.

      What would resolve it, for whoever owns the server and config surface:

      - **`.env.live` untracked**, with a `.env.live.example` holding the empty keys instead —
        the shape stays in the repository, the values cannot be committed
      - or `.gitignore` widened to `.env*` with the example file force-added, which is the same
        thing said the other way round
      - and, separately, a decision on TLS for non-local connections, which §7's "TLS in front"
        addresses for the client edge but not for the database hop

      Left open rather than resolved, because unlike Q13 / Q14 / Q15 nobody has decided it yet.

## Q17. An eslint selector enforces more than its own message describes
<!-- spec: none -->
<!-- blocking: no -->
<!-- category: upstream-defect -->

Found at `#data-model`'s checkpoint 8, linting the audit fixes.

`@openreachtech/eslint-config` declares:

```js
{
  selector: 'IfStatement[test] AwaitExpression',
  message: 'Do not use await in if condition',
}
```

**That is a descendant selector, so it matches an `await` anywhere inside an `if` statement —
including its body — not only in the `test` the message names.** Two ordinary guarded awaits
tripped it:

```js
if (staffMemberId !== null) {
  await this.verifyStaffMember({ ... })   // <- flagged, and there is no await in the condition
}
```

`await` inside an if-block body is unremarkable code. Any project on this config hits this the
first time it guards an asynchronous call, and the message sends the reader looking at a
condition that is already fine.

**A selector matching only the condition would be `IfStatement > .test AwaitExpression`** — the
child combinator scoping it to the `test` node the message is about.

- [x] resolved **in this project, by restructuring rather than by disabling**
      `no-restricted-syntax` is the most protected rule in the disable order — never disabled
      over the others — so the code moved instead: the null guards left the hook bodies and
      became early returns inside `verifyStaffMember` / `verifyExpenseCategory`. The result is
      arguably better (one responsibility per method, no branching around the await), but it
      was **forced by a rule whose message describes something narrower than what it enforces**,
      not chosen.

      **Fourth of the same family as Q13, Q14 and Q15** — correct on the machine it was written
      on. This one differs in that nothing is environment-specific: the selector is simply
      wider than intended, everywhere, for everyone.

      Worth carrying to whoever owns `@openreachtech/eslint-config`, alongside the other three.

## Q18. Two prohibitions collided, and the disable procedure was not used
<!-- spec: none -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

Found at the same point, and recorded because the outcome was *not* to take the sanctioned
escape hatch.

Sequelize **requires** a `beforeBulkUpdate` hook to rewrite `options.attributes` in place — it
reads the values back out of that same object (`node_modules/sequelize/lib/model.js:1939-1943`,
verified). But:

- `no-param-reassign` (`props: true`) forbids assigning to a parameter's property
- `no-restricted-properties` forbids `Object.assign` outright — "Never use `Object.assign()`"

`/hora-build` has a procedure for exactly this shape: cut an `adhoc/` branch, disable the
lower-ranked rule for that one file, and raise an `eslint-exception` question. By the protection
order that would have been `no-param-reassign` (tier 3) rather than `no-restricted-properties`
(tier 2).

- [x] resolved **without an exception, because one was not warranted**
      `Reflect.set(attributes, 'email', ...)` satisfies both rules and is not a workaround for
      either: it mutates in place, which is what the framework's contract requires, and it is a
      standard way to set a property. A comment states why the obvious two forms are unavailable.

      **The disable procedure is for when no version of the code satisfies both rules.** A
      code solution existed, so reaching for the exception would have spent a rule-disable on a
      problem that had an answer. Recorded so the absence of an `eslint-exception` entry here
      reads as a decision rather than an oversight.

## Q19. The Sequelize activator cannot be driven from plain node on Windows
<!-- spec: none -->
<!-- blocking: no -->
<!-- category: lacked-environment -->

Found at `#data-model`'s checkpoint 9, trying to walk the use cases through the real models.

`SequelizeActivator` loads each model by dynamic-importing the path `rootPath.to()` returns — a
bare Windows absolute path like `D:\ORT\...`. Node's ESM loader refuses it:

```
ERR_UNSUPPORTED_ESM_URL_SCHEME: On Windows, absolute paths must be valid file:// URLs.
Received protocol 'd:'
```

**Jest resolves it and the entire 242-test suite runs**, so nothing is broken for the tests.
What cannot work is any **standalone script** that activates Sequelize outside jest — a
one-off walk, a data-inspection script, a manual reproduction.

- [x] resolved **by going around it, not through it**
      Checkpoint 9's walk ran in SQL against the migrated database (via the `sqlite3` driver
      under CommonJS, which the loader restriction does not touch) rather than through the
      models. That covers the data requirements the walk exists to check; the model-layer
      behaviours were already covered by the test suite, which is the environment that does
      resolve those imports. **The split is stated in the checkpoint's own record rather than
      implied**, so nobody reads one run as having done both.

      **Fifth of the family after Q13, Q14, Q15 and Q17** — and the second whose cause is
      Windows specifically. A fix would be `pathToFileURL()` around the path before the dynamic
      import, in `renchan-sequelize`; that is the package's to make, not this project's.

      Worth knowing before somebody spends an hour on it: the failure names a URL scheme, so it
      reads as a problem with the script rather than with the loader's treatment of a drive
      letter.

## Q20. The backend row carries seven high-or-critical advisories, and nothing gates on them
<!-- spec: none -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

Found starting `#sign-in`, checking the tree was clean before writing code.

`npm audit` in `expense-note-backend` reports **16 vulnerabilities: 1 critical, 6 high, 7
moderate, 2 low.** The app repository's `lint.yml` runs `npm audit` and gates on it; **the
backend row's three workflows — eslint, sqlite, mariadb — do not run it at all.** So none of
these fails a pull request, which is why every backend PR so far passed green with them
present.

**They split into three groups, and only one of them reaches a deployed product.**

| | What | Fix |
|---|---|---|
| **five, including the one critical** | `sqlite3`, and `tar` / `cacache` / `node-gyp` / `make-fetch-happen` beneath it. `tar`'s is arbitrary file creation via hardlink | **`sqlite3@6.0.1`, a semver-major bump.** `sqlite3` is a **devDependency** — the dev and test dialect — so none of these ships |
| **one, and it is the one that matters** | **`multer` 2.2.0, high: denial of service via a crafted multipart field name.** Reached transitively through `@openreachtech/renchan@2.9.2`, which is a **runtime** dependency | **not fixable from this project's own ranges.** It needs renchan to bump multer, or an `overrides` entry here |
| one | `js-yaml`, high, CPU use on empty merge sources. Transitive, dev-reachable | in range |

**The `js-yaml` advisory is the same one that was fixed in the app repository** (its PR #11).
It is still here, in the backend, unfixed — fixing it in one repository does not reach the
other.

- [x] resolved
      **Cleared on the backend row: `npm audit` now reports 6 moderate and nothing above,
      down from 16 with one critical and six high.** Verified against `release/1.0.0` at
      `866a754`, and the lockfile confirmed to carry `multer` 2.3.0, `renchan` 2.9.3,
      `body-parser` 1.20.8 and `sqlite3` 6.0.1.

      **One line above is wrong, and correcting it matters more than the fix.** This entry said
      `multer` was "not fixable from this project's own ranges" and would need an `overrides`
      entry or a bump by renchan. **Both were unnecessary.** `@openreachtech/renchan` declares
      `multer: ^2.1.1`, so the patched 2.3.0 was always inside a range this project already
      had — a lockfile move reached it, with `package.json` untouched and **no `overrides`
      block added**, which matters because an `overrides` entry in a sample application is a
      divergence every later reader would have to explain.

      **How the error was made, since it is the reusable part:** `npm ls multer` was read,
      showing multer transitive beneath renchan, and "transitive" was treated as "pinned by its
      parent". It is not — a transitive dependency moves freely within whatever range its
      parent declares. The check skipped was a single command: read renchan's own declared
      range. The lesson is not about multer; it is that a dependency's reachability has to be
      read off the declaring `package.json`, never inferred from the shape of a tree listing.

      **The `sqlite3` semver-major went ahead and was right to be flagged first.** 5.1.7 →
      6.0.1 cleared the critical and all four remaining highs, and it turned out **reductive**:
      6 installs a prebuilt binary instead of building through node-gyp, so the toolchain
      subtree left the tree — 1,153 lockfile deletions against 140 insertions. Both the SQLite
      and the MariaDB suites passed on it, which are the suites that actually exercise that
      driver. **The suites approved it rather than an argument on paper**, which is the only
      reason a major bump on the test database driver should ever be taken.

      **The six that remain are all moderate and none is fixable from here** — `qs` and `uuid`
      transitively, and `express`, `sequelize`, `sequelize-mig` and `@openreachtech/renchan`
      directly. They need major bumps on the framework, which is **renchan-boilerplate's
      decision, not this project's**: every row created from 1.11.0 carries the same six, so
      clearing them here would fork this project from the boilerplate while leaving the
      boilerplate exposed.

      **The gap that let sixteen accumulate is still open, and it is the finding worth
      keeping.** None of the backend row's three workflows runs `npm audit`, while the app
      repository's `lint.yml` does — which is exactly why every backend pull request passed
      green with a critical present. Adding the step is right and belongs **upstream**, not as
      a local divergence. Sixth of the family after Q13, Q14, Q15, Q17 and Q19.

      **Original assessment, kept rather than rewritten:** A dependency change is its
      own `install/` or `update/` branch (`commits.md`), and two of the three groups are
      decisions rather than mechanics:

      - **`sqlite3` to 6.0.1 is semver-major on the test database driver.** It is what every
        suite runs against, so the bump is a decision with a real blast radius, not a lockfile
        nudge. Worth doing — the critical is in that group — but worth doing deliberately, with
        the suites as the check
      - **`multer` cannot be fixed from here.** The honest options are an `overrides` entry
        pinning a patched multer under renchan, or renchan itself bumping. The second is right
        and the first is available if waiting is not acceptable. **This is the only one of the
        seven that a deployed product is exposed to**, since `sqlite3` is dev-only
      - `js-yaml` is a three-line lockfile move, the same one already made in the app repository

      **The reason to record rather than carry silently:** the whole-version sweep points the
      security audit at the repository entire, not at one feature's change set, and its
      dependency check will raise all of this. Better dated now, with the dev-versus-runtime
      split already worked out, than discovered at the gate before a release.

## Q21. §10's first use case ends on a clause no gate of its own can check

<!-- spec: sign-in -->
<!-- blocking: no -->
<!-- category: spec-assumption -->

Found at `#sign-in`'s checkpoint 1, reading §10 closely enough to build from.

The use case reads: "a member of staff opens the app on a Monday morning, signs in with the
address and password they were issued, and is signed in — **every screen after that knows who
they are**."

Checkpoint 1's exit condition asks that each of a feature's requirements, use cases and
acceptance criteria be "checkable against a product in which this feature and its `depends`
are built and nothing later is". With `#data-model` and `#sign-in` built and nothing after
them, **`#sign-in` has exactly one screen — the sign-in screen** (§10.2). There is no "screen
after that" to check the clause against; the two that follow belong to `#expense-entry` and
`#monthly-summary`.

**Not routed to `/hora-spec`, and this is the reason.** A forward-reaching *acceptance
criterion* is a stop — `/hora-plan` raises it `blocking: yes` and checkpoint 1 does not pass
while it stands. This is not one. §10's eight acceptance criteria are the things actually
checked, and **every one of them is local**: identical refusals, a session surviving a reload,
no password in a response or a log, `signedInStaffMember` refused without a session, the three
renewal failures, series revocation on reuse, renewal with no cookie, and the eleventh failed
attempt. None mentions another screen.

**The reading checkpoint 1 passed on, recorded so no later gate re-litigates it:** the clause
describes the *mechanism* — the access token goes on a request header, so any operation a
later screen calls carries proof of the session, and `signedInStaffMember` is what answers who
is holding it. Checked that way it is §10's fourth criterion read forwards, and it holds with
one screen built.

**What this costs if the reading is wrong:** nothing until `#expense-entry` opens its screen,
at which point the clause becomes checkable for real and the acceptance sweep will check it.
The risk is not that it fails — it is that a sweep reads the clause as unmet at 1.0.0 without
knowing a gate already considered it.

## Q22. Nothing in the product can issue the first account

<!-- spec: none -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

Found at `#sign-in`'s checkpoint 2, walking the first use case — which begins with an address
and password "they were issued".

**No account exists, and no route creates one.** `sequelize/seeders/` holds two seeders, both
for `expense_categories`; there is **no `development/` seeder for `staff_members`,
`staff_member_secrets` or `staff_member_password_hashes`**. §4 rules a sign-up operation out
of scope on purpose — "accounts are issued by an operator outside the product, so build no
sign-up operation and no sign-up screen" — so the absence of the operation is a decision, not
a gap.

**What is missing is the mechanism the decision implies.** "An operator outside the product" is
not a procedure anybody can run: the operator would have to compose a bcrypt digest by hand and
insert three rows across three tables in the right order.

**The build side is unambiguous and needs no spec change.** `#sign-in`'s tests read real rows
by the always-on testing rule — data from `seeders/development/*`, a seeder added where it is
missing, never a mocked row — so **development seeders for those three tables are checkpoint 5
of `#sign-in`**, and checkpoints 17 and 18 need them too: an end-to-end run signs in as a real
member of staff or it proves nothing.

**The deployed side is what stays open.** A seeder is development data and has no business
carrying a real password into a real deployment. What a deployment gets is 20 members of staff
whose accounts must come from somewhere, and the spec names no script, no operation and no
procedure. §8 declares MariaDB, so hand-written SQL is the implied answer.

**Not raised to `/hora-spec`, because it is not this version's blocker.** §4 already routes
password *reset* to a later version, needing a mail sender this version does not declare, and
issuing an account is the same shape of problem. Recorded so it is a decision at 1.0.1 rather
than a discovery on the first day of use. Adjacent to Q16 (`.env.live` tracked): both are
"what the deployed product needs that no gate here exercises".

## Q23. The pinned contract declared an operation the spec never did, and the same commit forbade it

<!-- spec: sign-in -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

Found at `#sign-in`'s checkpoint 3, reading the contract before writing the staff SDL.

`.hora/contracts/1.0.0/staff-graphql.graphql` declared `healthCheck: Boolean!` as a field of the
staff `type Query`. **The spec never mentions a health check, in any section.** §10.1 holds four
operations, §11.1 five and §12.1 one; `healthCheck` is none of them.

**`#sign-in`'s own task file forbids it by name**, and both files were written by the same
`/hora-plan` commit (`731b9d4`, "Plan 1.0.0 into four features, and pin the staff contract"):

> **the staff audience adds NO `healthCheck`.** The backend row ships one for its own
> audiences, and copying it across out of habit is the obvious thing to do — do not. §10.1 is
> the complete operation list, and §7's Authentication row is written against it staying that
> way

So the derivation contradicted itself inside one commit: the constraint anticipated exactly this
mistake, and the contract beside it made it. **It surfaces only when somebody builds from both
artifacts at once**, which is checkpoint 3 and no earlier gate.

**Why it is not cosmetic.** §7's Authentication row reads "every operation authenticates by one
of two credentials, and which one is part of adding it", naming three exceptions —
`signIn`, `signOut`, `renewAccessToken`. A `healthCheck` has neither credential, so it would
have to join `schemasToSkipFiltering` as a fourth, and §7's rate-limiting row would stop being
true as well ("these are the two operations reachable without a session, and the only two
limited at 1.0.0"). One convenience field would falsify two sentences of the spec.

**Resolved by correcting the contract, not by raising it and stopping.** `/hora-build` says
wanting to change a contract mid-checkpoint means raising a question rather than changing it —
that rule guards a contract against an implementation that finds it inconvenient. This is the
other case: the contract contradicted the document it was derived from, and `specs/` is the
authority over every derivation under `.hora/`. The field was removed and a comment left in its
place saying why, so the next reader does not restore it. **Nothing in `specs/` was touched.**

**The consumer had not read it yet.** The contract is pinned for both sides, so changing one
normally desyncs the frontend — but no frontend work has begun, `#sign-in`'s frontend gate
starts at checkpoint 10, and a health check appears in no screen's call table in §10.2, §11.2 or
§12.2. Had a screen been built against it, this would have been a stop rather than a fix.

**What stays open, and is the reusable part:** nothing checks a pinned contract against the spec
it was derived from. `/hora-plan` verified the operations exist; it did not verify that no
operation exists which the spec does not declare. Both directions want checking, and only one is.

## Q24. Q19 understated itself — the product's own entry point cannot boot on this machine

<!-- spec: none -->
<!-- blocking: no -->
<!-- category: lacked-environment -->

Found at `#sign-in`'s checkpoint 3, trying to build the new audience's server the way
`server/index.js` does.

**Q19 says the failure is confined to "any standalone script that activates Sequelize outside
jest — a one-off walk, a data-inspection script, a manual reproduction". `server/index.js` is
such a script.** It is the product's single entry point, its first statement is `await
activate()`, and it dies there:

```
Error [ERR_UNSUPPORTED_ESM_URL_SCHEME]: On Windows, absolute paths must be valid file:// URLs.
Received protocol 'd:'
```

### Root cause found at checkpoint 14, and the scope was wider than this entry claimed

**This entry said "on this machine". It is not this machine, and it is not this project — it is a
one-line defect in `@openreachtech/renchan` that stops any renchan server booting on Windows.**

`DeepBulkClassLoader.loadClasses()` builds absolute filesystem paths with `path`, then:

```js
const exported = await import(it)      // lib/tools/DeepBulkClassLoader.js:61
```

`pathToFileURL` is never imported by that file — only `fs` and `path`. On Windows an absolute path
is `D:\...`, whose scheme Node reads as the protocol `d:`, and the ESM loader refuses it. On Linux
and macOS an absolute path begins `/`, which Node tolerates, so the defect is invisible everywhere
except Windows.

**Proved rather than diagnosed**, in isolation and without modifying anything installed — importing
one real model file both ways:

```
the absolute path the loader passes: D:\ORT\...\sequelize\models\StaffMember.js
raw absolute path        -> ERR_UNSUPPORTED_ESM_URL_SCHEME
pathToFileURL(path).href -> IMPORTED
```

So the fix is `await import(pathToFileURL(it).href)`, and it is confirmed to work.

**The same defect exists twice, at the same line number, in two packages:**

| Package | Version | File |
|---|---|---|
| `@openreachtech/renchan` | 2.9.3 | `lib/tools/DeepBulkClassLoader.js:61` |
| `@openreachtech/renchan-sequelize` | 2.2.2 | `lib/tools/DeepBulkClassLoader.js:61` |

**And the blast radius is every class-loading path a server has at startup**, not only models:

- `RenchanModelsLoader` / `SequelizeActivator` — the Sequelize models
- `GraphqlResolversLoader` — **the GraphQL resolvers**
- `GraphqlPostWorkersLoader` — the post-workers
- `RestfulApiRoutesBuilder` — the REST routes

So it is not that this product's entry point happens to die early. **No renchan application can start
on Windows at all**, and it would die at the next loader even if the first were fixed.

### Why a green suite says nothing about it

**Jest never reaches the failing path, for two independent reasons**, which is why 915 passing tests
coexist with a server that cannot start:

1. Jest supplies its own module registry and intercepts `import()`, so a specifier that Node's ESM
   loader would refuse is resolved by Jest instead.
2. **No test exercises `loadClasses()` at all.** The suite reaches models and resolvers by importing
   them directly, never through the loader that boots them.

This is another instance of the pattern recorded at the frontend gate: the check and the failure do
not meet. The suite is not weak here — it is pointed somewhere else entirely.

### Where it lands

**Upstream, in `@openreachtech/renchan` and `@openreachtech/renchan-sequelize`**, as a one-line change
in each. Not fixable in this project except by patching `node_modules`, which would be undone by the
next install and is not this feature's to do.

Until it is fixed, **nothing in this product has ever been exercised through a running server on this
machine.** Every gate has been met in-process — checkpoint 4 said so explicitly, and checkpoint 14
reports the same split. That is a real and stated limit on what any verification here can claim, and
it is worth keeping attached to this entry rather than rediscovered.


Traced to `@openreachtech/renchan-sequelize/lib/tools/DeepBulkClassLoader.js`, which
`await import(it)`s a bare `D:\...` path that `rootPath.to()` returned.

**This is not `#sign-in`'s doing and predates the branch.** The base commit's `server/index.js`
carries the identical first line, so the customer and admin servers cannot start on this
machine either — nobody had noticed, because jest's resolver tolerates the path and the suites
are what everyone runs. Q19 found the mechanism and dated it correctly; what it got wrong was
the blast radius, calling it a scripting inconvenience when it is the deployment entry point.

**What it costs, in order:**

- **checkpoint 17** builds the local end-to-end environment, and **checkpoint 18** drives the
  product through a browser. Neither can happen on this machine while the entry point cannot
  start. That is two gates away, not far
- the frontend gate needs a backend to talk to, so checkpoints 14 to 16 lose their live target
- it is invisible to every gate before those, because lint and jest both pass

**Not a code defect in this repository, so not a `retake/`.** It is an upstream defect in
`renchan-sequelize` — a path handed to the ESM loader that must be a `file://` URL on Windows —
and the honest fix is upstream, where every row created from the boilerplate gets it. Seventh of
the family after Q13, Q14, Q15, Q17, Q19 and Q20.

**What was done instead, so checkpoint 3 was not blocked on it:** the audience's schema was
verified by running the framework's own `SchemaFilesLoader` over the real directory and
`makeExecutableSchema` over the result — the same code path minus the socket. What could not be
exercised is `listen(4900)` and an HTTP probe of the endpoint.

## Q25. This repository's README is the boilerplate's, inherited verbatim

<!-- spec: none -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

Found at `#sign-in`'s checkpoint 3, correcting the sentences the staff audience falsified.

`expense-note-backend/README.md` is titled **`# renchan-boilerplate`** and opens "A running
skeleton for a renchan application… This repository is the starting point for an application
built on it." It is the boilerplate's own README, never rewritten for this product.
`README.ja.md` mirrors it.

**What was fixed here, and why only that.** Six statements had become false because this feature
added an audience — the server count, the health-check blanket, the `TODO`-marker count, "both
audiences share one implementation", the cookie-name list, and the concept paragraph. Those were
this change set's to fix, and they are fixed in both languages, written without counts so the
next audience falsifies nothing.

**What was left alone:** the title, the framing, and the table of contents. Turning this into the
product's README is a rewrite — the always-on `npm-package.md` rule routes a README through the
readme skill, with a stated section order and a `docs/` split — and doing it inside a checkpoint
would be scope nobody approved, on a file whose whole structure that skill governs.

**Why it is worth a question rather than silence.** A reader arriving at this repository is told
it is a boilerplate and that what is left is "the application's own schema, resolvers and
models" — which ten tables, a staff audience and 310 tests have since become. **The health-check
sentence was the concrete cost**: it was still promising a health check on every GraphQL endpoint
hours after `healthCheck` was removed from this audience as a defect (Q23), and it is exactly
what would have led the next reader to restore it. Documentation that describes a different
project does not merely go stale; it argues against the code.

Belongs in the same conversation as `#expense-entry`'s or `#monthly-summary`'s documentation
work, or as an `update/` branch of its own once the version's features are in.

## Q26. §2.1 declares one server and the deployed product will carry four

<!-- spec: none -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

Found at `#sign-in`'s checkpoint 3, opening the staff audience.

§2.1 reads: "One server, one consumer. **There is no REST server**: nothing here is a file
transfer, a redirect or a third party that cannot speak GraphQL, so the default stands." The
built product will ship four servers, all started by `server/index.js`: the staff GraphQL
endpoint this feature adds, plus the boilerplate's customer GraphQL (3900), admin GraphQL (5800)
and RESTful API (8001).

**The tree record already settled that they stay.** `.hora/tree/expense-note-backend.md` says
"the boilerplate ships `customer` and `admin`. `expense-note` declares one server,
`staff-graphql` (spec 2.1), so implementation **adds** a `staff` audience" — adds, not replaces.
Removing them is in no section of §10, so it is not this feature's work, and doing it here would
be scope nobody approved.

**But three undeclared endpoints is a security surface, not a tidiness question.** Each answers
`healthCheck` and nothing else today, so the exposure is small — what is undeclared is the
*surface*, and §2.1's "one server, one consumer" is false of the thing that gets deployed. §7's
Authentication row governs "every operation", and three of the four servers serve operations no
section of the spec declares, authenticating by neither credential.

**Where it lands, and why it is recorded now.** Checkpoint 8 audits *this feature's change set*,
which will not contain them. The whole-version sweep points the same audit at the repository
entire and will raise all three. Recording it here means the sweep meets a dated finding with the
scope reasoning already worked out, rather than discovering it before a release. Adjacent to Q16
(`.env.live` tracked) and Q22 (nothing can issue the first account): all three are "what the
deployed product carries that no gate of a feature exercises".

## Q27. The upload middleware survives the reasoning that removed the raw-body capture

<!-- spec: none -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

Found by verification at `#sign-in`'s checkpoint 3.

`StaffGraphqlServerEngine.collectMiddleware()` omits the `rawBody` capture the other three
engines carry, and the engine says why: nothing in this repository or in `renchan` reads
`rawBody`, it is laid in for a webhook that must verify a signature over raw bytes, and this
audience serves none — §10.1 is its complete operation list.

**The same reasoning applies verbatim to the middleware immediately above it**, which was kept:

```js
graphqlUploadExpressWithResolvingContentType({
  maxFileSize: 10000000, // 10 MB
  maxFiles: 10,
}),
```

§10.1, §11.1 and §12.1 declare no upload anywhere in this audience, and §4 puts a receipt
photograph out of scope for 1.0.0 with object storage undeclared. So 10 MB × 10 files of
multipart parsing sits in front of an endpoint whose `signIn` skips the authentication filter
entirely — reachable with no credential at all, which is the one place unused parsing capacity
is worth naming.

**Not acted on, deliberately.** Checkpoint 8 is the security audit and owns this call with its own
criteria; changing it here would pre-empt the gate whose job it is. What makes it worth recording
rather than leaving to be noticed is the asymmetry verification found: **the omission was reasoned
and the retention was not.** One of the two decisions was made and the other was inherited, and
only the first left an argument behind.

## Q28. A stub-served field has no authentication filter, so a stub is a public endpoint

<!-- spec: sign-in -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

Found at `#sign-in`'s checkpoint 4, digesting the stub-API skill before writing any stub.

**The authentication filter is built from the `actual/` resolvers alone.** In
`@openreachtech/renchan/lib/server/graphql/resolvers/GraphqlResolversBuilder.js`:

```js
const schemas = this.extractSchemas({
  schemaHash: actualResolverSchemaHash,      // actual ONLY — the stub hash is not read here
})
const filterSchemaHash = await this.buildFilterSchemaHash({ engine, actualSchemas: schemas })
```

and then, per schema, over the **union** of actual and stub:

```js
const filter = this.filterSchemaHash[it]                 // undefined for a stub-only field
const resolver = this.actualResolverSchemaHash[it]
  ?? this.stubResolverSchemaHash[it]                     // actual supersedes stub automatically
```

`generateResolverResolveCallback` then calls `await filter?.(envelope)`. **`undefined` means no
filter runs** — no `Unauthenticated`, no `Unauthorized`, no `DeniedSchemaPermission`. The same
mechanism `schemasToSkipFiltering` uses to make an operation public (`.hora/tasks/1.0.0/sign-in.md`,
checkpoint 3) applies to every stub-only field, without anybody listing it.

**So `signedInStaffMember` is public for exactly as long as it is a stub** — the one operation of
the four deliberately kept *out* of the skip list, because §10 requires it refused without a
session. Its criterion is unmeetable at checkpoint 4 by construction, since a stub returns
hardcoded data and cannot refuse. **That is checkpoint 6's to satisfy, not checkpoint 4's**, and
checkpoint 4's exit condition asks for hardcoded data in as many words.

**Three consequences, in rising order of cost:**

1. **Checkpoint 8's security audit reads this feature's change set**, which will contain four
   publicly-reachable operations where §7 permits three. The audit should meet that as a dated
   finding rather than a discovery.
2. **The frontend gate builds against the stub.** Checkpoints 12 to 14 develop a client and a
   screen against an endpoint that **never refuses**, and checkpoint 16 swaps them onto one that
   does. A client with no unauthenticated path — no redirect to the sign-in screen, no retry
   through `renewAccessToken` — passes every frontend checkpoint and breaks at 16. §10.2 says
   every other screen "sends somebody here when theirs has gone", so that path is the feature,
   not an edge case.
3. **The production shape is the one worth naming.** The engine loads both resolver directories
   in every environment, so a deployment where checkpoint 6 missed an operation serves that
   operation **publicly, with hardcoded data, and nothing fails.** No error, no log line, no
   failing test — the endpoint simply works and lies. This is the argument for the stub skill's
   own advice to *move* a stub into `actual/` at checkpoint 6 rather than leave it beside the
   real one: the framework's `?? ` fallback means a leftover stub is invisible until the actual
   one is deleted, and then it is invisible in the other direction.

**Recorded rather than acted on, because there is nothing to fix here.** This is how the
framework's stub mechanism works, and the mechanism is what makes the frontend gate independent
of the backend gate finishing — which is the whole reason checkpoint 4 precedes checkpoint 6.
What it needs is to be **known** at three later gates, which is what this entry is for.

## Q29. `test.sh` runs a whole test phase against directories that do not exist

<!-- spec: none -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

Found at `#sign-in`'s checkpoint 3, reading the runner while digesting the backend-testing skill.

`expense-note-backend/test.sh` runs the suite in **two phases**, and the distinction between them
is real and useful:

```sh
function testWithEmpty () {          # the database carries MASTER seeds only
  jestCommand "$@" tests/empty/__tests__/
  jestCommand --detectOpenHandles tests/empty/_orders/
}

function testWithSeeded () {         # then development seeds are added
  npm run db:seed:dev
  jestCommand "$@" tests/__tests__/
  jestCommand --detectOpenHandles tests/_orders/
}
```

**Neither `tests/empty/__tests__/` nor `tests/empty/_orders/` exists.** The phase survives only
because every invocation carries `--passWithNoTests`, so two of the four jest runs do nothing and
say so quietly. **Not a defect** — an unused capability, and `--passWithNoTests` is what makes it
harmless rather than a broken runner.

**It is directly relevant to this feature, which is why it is recorded now rather than left.**
§10's first acceptance criterion is that "an address with no account and a correct address with
the wrong password are refused identically". Today `staff_members` is empty, so "an address with
no account" is trivially any address. **Once Q22's development seeders exist, that stops being
true**: every seeded address has an account, so the test has to pick an address deliberately
absent from the seeder — a value whose meaning depends on a seeder file the test does not name.

The `empty` phase is where an assertion that genuinely needs an unseeded table belongs, and it
runs before `db:seed:dev`. **Checkpoint 6 should choose per test which phase it wants**, rather
than defaulting everything into `tests/__tests__/` because that is where the other files are.
Adjacent to Q22.

## Q30. The reconciliation between shared test doubles and the no-hoisting rule is undecided

<!-- spec: none -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

Raised by the agent digesting the backend-testing skill, which flagged its own resolution as
non-authoritative rather than presenting it as settled.

**The equipped skill wants shared, itself-tested doubles** under `tests/mocks/` and `tests/tools/`.
**The always-on rule wants nothing hoisted**: "The instance under test, every stub/fixture it
needs, and `factoryParams` all belong in the case… never declared as a shared `const` above or
inside a `describe`. Duplicating a value across cases is accepted and preferred."

These do not obviously contradict — one is about a shared *class* in its own file, the other about
a `const` at describe scope — but the boundary between them is exactly where an agent will guess.

**The working resolution, used at checkpoint 3 and recorded here so it is visible rather than
implicit:** a one-line stub is inlined into each case, duplicated as the rule prefers;
`tests/mocks/` is reserved for a genuine shared mock **class** that is itself tested. Neither
`tests/mocks/` nor `tests/tools/` exists in this repository yet, so nothing has had to choose.

**Why this is a question and not a ruling.** Q10 settled that the always-on rules beat an equipped
skill, and named the cases it was deciding — the `@augments` spelling, index-name abbreviation and
`Promise.all`. **It did not decide this one**, and the digest that reconciled it said so plainly
instead of quietly writing a rule into a file agents read as authority. Worth settling before
checkpoint 6, which writes eight resolver tests and is the first place a shared double would be
tempting.

## Q31. `checkJs` is on and `jsconfig.json` never resolves the test globals, so two thirds of its output is phantom

<!-- spec: none -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

Found at `#sign-in`'s checkpoint 4, running the type check an implementer flagged as owed —
`server.graphql.staff.*` is a new ambient namespace and no lint rule enforces a JSDoc type name.

**`npx tsc -p jsconfig.json --noEmit` reports 1630 errors, and 1010 of them are not real.** They
are `Cannot find name 'describe'` / `'test'` / `'expect'` — one family, across every test file in
the repository. `@types/jest@30.0.0` is both declared in `devDependencies` and present in
`node_modules/@types/`, so this is not a missing dependency.

**It is a config gap, and the diagnosis is one line.** `jsconfig.json` carries `checkJs: true`,
`moduleResolution`, `module`, `target` and `exclude: ["node_modules"]` — and **no `types` field.**
Adding one clears the family outright:

| run | total errors | the missing-globals family |
|---|---|---|
| as configured | 1630 | 1010 |
| `--moduleResolution bundler --module esnext` | 1630 | 1010 |
| **`--types jest,node`** | **409** | **0** |

The middle row is there because a modern module resolution was the obvious hypothesis and it is
**wrong** — identical numbers. Only the explicit `types` list moves it.

**Why this is worth an entry rather than a shrug.** `checkJs: true` says somebody intended these
files to be type-checked. What the configuration actually produces is 1010 phantom errors that
**mask 409 real ones** — and any editor reading `jsconfig.json` shows every developer the same
1010. A check nobody can read is a check nobody runs.

**Among the 409 that were masked, some are in code `#sign-in` is about to build on.**
`app/session/SessionClerk.js` carries three `TS2322: Type 'unknown' is not assignable to type
'Error | null | undefined'`, and `app/session/BaseSessionResult.js` an `Object is possibly
'null'`. Checkpoint 5 has to extend `SessionClerk` with the access-token read that
`StaffGraphqlContext.findUser` owes, so knowing those exist beforehand is worth more than
discovering them while adding a method.

**What this run established about `#sign-in` itself, which was the reason for running it:**

- **zero errors in every source file checkpoint 3 created** — the engine, the context, the share,
  the model and `types/StaffGraphQL.d.ts`
- **zero errors anywhere naming `server.graphql.staff`**, so the ambient namespace the always-on
  `graphql-resolvers.md` mandates does resolve. That was the open question
- where this feature's files do appear, **the error shapes are identical to the siblings they
  mirror and strictly fewer**: `StaffGraphqlServerEngine` 14 against `CustomerGraphqlServerEngine`
  20, of the same kinds; `SignInAttempt` and `StaffMemberSecret` carry the same two, one each. So
  no new *kind* of type error was introduced

**Not fixed here, for two reasons.** `jsconfig.json` is repository-wide configuration and nothing
in §10 touches it, so it is scope nobody approved. And **tsc is not part of this project's
toolchain**: `review-and-tooling.md` names ESLint, Jest and the spell checker, and the always-on
`jsdoc.md` reaches for tsc only to *emit* declarations (`tsc --emitDeclarationOnly`), never to
check. So this changes no gate — it changes what a developer sees in an editor, and what a future
run of the check would be able to tell them.

The fix, when somebody takes it, is `"types": ["jest", "node"]` in `jsconfig.json`, on an
`update/` branch of its own, with the 409 triaged separately. Adjacent to Q13, Q14, Q15, Q17,
Q19, Q20 and Q24 — the family of things the boilerplate ships that no gate here exercises.

## Q32. bcrypt ignores a password past 72 bytes, and nothing caps one

<!-- spec: sign-in -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

Raised by the agent writing the password encipher at `#sign-in`'s checkpoint 5, which declined to
add a guard nothing had asked for rather than adding one quietly.

**bcrypt truncates its input at 72 bytes.** So two passwords sharing their first 72 bytes produce
the same digest and verify interchangeably. `bcryptjs` v3 exposes a `truncates(password)` helper
to detect it, and **nothing in this product calls it**:

- **§7 and §9.5 cap nothing.** §7 says only "a password is a one-way hash and is never stored,
  returned or logged in any other form"; §9.5 declares the digest column and says nothing about
  the plaintext's length
- **the encipher does not guard it**, deliberately — its assignment said not to add methods
  nothing needs, and refusing a password is user-visible behaviour no section describes
- **no input validator guards it either**, because `#sign-in` has none. That was decided at
  checkpoint 5: §10's eight criteria say nothing about input validation, and the contract types
  both `SignInInput` fields `String!`, so GraphQL refuses a missing one before a resolver runs

**What it actually costs.** Somebody who sets a 100-character password from a password manager has
its last 28 characters ignored — and would get in by typing only the first 72. That is a real
weakening the person did not consent to, and it is invisible: nothing fails, nothing logs, and the
sign-in works.

**Why it is not urgent.** No account exists that anybody chose a password for. §4 rules sign-up
out of scope for 1.0.0, and the only credentials in the tree are development seeder fixtures. So
the exposure arrives with the first real account, which arrives with whatever mechanism Q22 is
still open about. **The two questions want deciding together.**

**Three ways it could go, and none is obviously right:**

| | what it costs |
|---|---|
| refuse a password over 72 bytes | user-visible, and a refusal no section of the spec describes. Needs a §7 or §10 sentence to sit behind it |
| pre-hash the password (SHA-256, then bcrypt the digest) | removes the limit with no user-visible change, and is standard practice — but it changes what the stored digest is a digest *of*, so adopting it later than the first real account means a reset for everybody. It also diverges from what the always-on testing rule assumes when it asserts a bcrypt digest of a password |
| accept it | the documented behaviour of the algorithm the project chose, and 72 bytes is a long password. But "documented" is not the same as "somebody decided it" |

**Where it lands.** Checkpoint 8's security audit reads this feature's change set and will meet the
encipher; it should meet this dated rather than discover it. And whichever way it goes, **option
two stops being cheap the moment a real password is stored**, which is the only reason this is
worth writing down now rather than at 1.0.1.

## Q33. Two tests in the suite pass by accident, and seeding development data is what exposed it

<!-- spec: none -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

Found at `#sign-in`'s checkpoint 5, seeding the three staff-account tables. **Neither finding is
caused by those rows** — both were latent and became visible because `sequelize/seeders/development/`
stopped being empty for the first time.

### 1. A master-seeder test depends on suite order

`tests/__tests__/sequelize/seeders/master/expense_categories.js` asserts the **whole table**:

```js
const actual = await ExpenseCategory.findAll({
  order: [['displayOrder', 'ASC']],       // no `where` — every row in the table
})

expect(actual).toEqual(expected)          // exactly four objectContaining entries
```

and `tests/_orders/Expense/Expense.js` creates categories and leaves them behind:

```js
await ExpenseCategory.create({
  id: params.ExpenseCategoryId,           // 10000311, 10000313, … left in the table
  name: `expense category ${params.ExpenseCategoryId}`,
  displayOrder: 1,
})
```

**It passes only because `test.sh` runs `tests/__tests__/` before `tests/_orders/`.** Reverse
those two lines, run `__tests__` alone against a database an `_orders` run has already touched, or
parallelize the two categories, and it fails with a large diff. Reproduced, and confirmed
unrelated to the new rows: a fresh `db:refresh` makes it pass again.

**Why it is worth recording rather than shrugging at.** The always-on testing rule's whole stance
is that "a test must fail when the implementation is **wrong**". This one can fail while the
implementation is **right**, which is the same defect wearing the other face — and it will do so
at the least convenient moment, since nothing in the runner declares the ordering it depends on.
The rule also says plainly: "Never call `Model.findOne` / `update` / `findAll` directly inside a
test to fetch or verify." A seeder test is the one place that instruction is awkward, and scoping
the read to the seeded id block is what resolves it.

**It is `#data-model`'s file and not `#sign-in`'s to change.** The fix is a `where` on the
`1000000x` block, or a `toEqual`-plus-length pair scoped to the seeded ids.

### 2. `db:seed:dev` was never idempotent; now it can bite

sequelize-cli's seeder storage here is the default `none`, so `db:seed:all` re-runs **every**
seeder on every invocation. While `development/` held nothing but `.directorykeeper.cjs`, running
it twice was harmless. Now a second `db:seed:dev` without a teardown between dies on
`SQLITE_CONSTRAINT: UNIQUE constraint failed: staff_members.id`.

**The same fragility already existed for `db:seed:master`** — a second run collides on
`expense_categories.id` — so this is the shape of the runner, not something the new rows
introduced. `npm test` and `db:refresh` both tear down before seeding and are unaffected. What
breaks is `./test.sh --seeded <path>` run twice in a row.

**Recorded because it is now reachable.** An agent or a person who runs the seed step twice while
iterating gets a constraint error naming a table they did not touch, and the honest diagnosis is
two directories away.

### 3. Settled while recording these: the error-path seeded rows stay

The agent asked whether the two members of staff holding no complete credential — one with no
address, one with a digest and no address — model anything real, since §9.4 and §9.5 each say one
row per member of staff and it offered to drop them for a uniform ten.

**They stay, and Q22 is the reason.** §4 rules sign-up out of scope, so "accounts are issued by an
operator outside the product" — and Q22 records that the product offers that operator no mechanism
at all, leaving hand-written SQL across three tables in the right order. **A half-issued account is
therefore not a hypothetical in this product; it is the most likely way one goes wrong.** The
seeder coverage convention asks for exactly such rows, and §10's identical-refusal criterion means
`signIn` has to refuse a credential-less account the same way it refuses a wrong password — which
is a behaviour needing a row to test against.

## Q34. A detected reuse is the one real security event in this flow and it leaves no trace

<!-- spec: sign-in -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

Raised by the agent closing `SessionClerk`'s gaps at `#sign-in`'s checkpoint 5, which declined to
invent an audit line rather than guessing at one.

A refresh token presented after it is spent means the cookie was copied — §9.7 exists to detect
exactly that, and §10 requires the whole series revoked when it happens. **The product now detects
it, revokes the series, and records nothing anywhere.** The refusal reaches the caller and the
event reaches nobody.

**Why no log line was added, which is the substance of the question.** Every identifier that would
make such a line useful is barred:

| identifier | why it cannot go in a log |
|---|---|
| the refresh token | §9.7 stores only a digest and the plaintext must never come back out |
| its digest | still a credential-equivalent for the row it names |
| the `sessionKey` | names the series, so it is the credential's handle |
| the member of staff's id or address | §7: "None of the three is ever written to a log line" |

So the honest options are a line carrying nothing identifying — which cannot be investigated — or
a table, which is a data-model change no section asks for. **The spec asks for no audit trail at
all**, and §8 declares no log aggregation.

**Not urgent, and here is the bound.** The security *response* is complete: the series is revoked,
so a stolen cookie stops working and so does the session it was stolen from. What is missing is
only the ability to know it happened. At 20 members of staff on an internal system, the person
affected notices they were signed out.

**Where it would land if taken.** A `staff_member_session_events` table, or §7 gaining a sentence
that permits a `sessionKey` in a log line at a stated retention. Both are 1.0.1 or later.
Checkpoint 8's security audit should meet this dated rather than raise it as an omission.

## Q35. §9.6 says an expired access token is deleted, and nothing deletes one

<!-- spec: sign-in -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

Found at `#sign-in`'s checkpoint 5, giving `SessionClerk` its access-token read.

§9.6 reads: "An expired access token is **deleted**, not flagged — there is no `revoked_at` here,
because the lifetime is the revocation."

**Only one path deletes one.** `SessionClerk#deleteAllAccessTokens({ sessionKey })` runs on
sign-out and on a series revocation. **An access token that simply expires is never deleted** —
`findAvailableAccessToken` refuses it through the model's `isAvailable({ pointsAt })` and leaves
the row where it is.

**Nothing available can prune them.** §8: "Redis is not declared, because this version runs no
background job. Every write finishes inside its own request, and nothing here leaves the process."
So there is no scheduled sweep to put this in. Deleting on read would turn the authentication hot
path — every operation of every screen — into a write, which is worse than the rows.

**What it costs, sized rather than asserted.** An access token lives fifteen minutes, so a working
day is roughly 32 per person; §7 foresees 50 members of staff, and §7's retention row keeps an
expense 7 years. That is on the order of a million rows accumulating in `staff_member_access_tokens`
over the retention period — not a performance problem for an indexed lookup on a unique column,
and not nothing either. **The table grows without bound and nothing in the product ever shrinks
it.**

**Not a code defect and not this feature's to fix.** §9.6 states an intent that §8's own decision
makes unimplementable at 1.0.0, so the two sections disagree quietly. **The honest reading is that
§9.6's "deleted" describes what sign-out does and overstates itself for the expiry case** — which
is a sentence, not a mechanism, and correcting a sentence in `specs/` needs approval this
checkpoint does not have.

**Where it lands.** Either §9.6 gains a clause saying an expired row is left until its series ends,
or 1.1.0 declares the job that sweeps it — which the approval feature planned for 1.1.0 may bring a
scheduler for anyway. Adjacent to Q26 and Q31: things the deployed product carries that no gate of
a feature exercises.

## Q36. A worked example in an equipped skill is corrupted at source, behind an eslint-disable

<!-- spec: none -->
<!-- blocking: no -->
<!-- category: upstream-defect -->

Found at `#sign-in`'s checkpoint 6, digesting `hor-resolver-validator` before writing the first
input validator this repository has had.

`@openreachtech/hora-skills-ort-renchan@0.1.0`, in
`hor-resolver-validator/references/validator-pattern.md` around line 233, the test example reads:

```js
    /* eslint-disable */
},
      {
        input: {
          originObjectCategory    const cases = [
      {
        input: {
          originObjectCategoryId: 1,
        },
        expected: true,
     Id: '1',
        },
```

**An entire `const cases = [` block has been pasted into the middle of the identifier
`originObjectCategoryId`**, splitting it across ten lines — `originObjectCategory` … `Id: '1',`.
The array is also unbalanced. It is not a formatting quirk; the code cannot parse.

**The `/* eslint-disable */` immediately above it is what makes this worth recording rather than
just fixing locally.** Whatever produced the corruption, the disable comment means no linter in any
consuming project will ever report it — so the example can sit broken indefinitely while reading as
deliberate.

**The cost to a consumer.** This is a *worked example in a reference file*, which is exactly the
thing an implementer copies. An agent handed this skill and told to follow its test shape would
reproduce unparseable code, and the disable comment would travel with it.

**It also contradicts this project's own standards twice over**, independently of being broken:
`D:/ORT/rules/testing.md` names the case fields `params` / `expected` (this uses `input`), and the
disable comment itself is refused by `eslint-comments/no-use` outside the three files
`expense-note-backend/eslint.config.js` exempts by name — under a comment telling maintainers never
to add a fourth (Q18 and checkpoint 3 both met that rule already).

**Not worked around, because nothing here consumes it.** The digest at
`.hora/digests/hor-resolver-validator.md` records the section as thin, states the defect, and tells
the implementer to follow `D:/ORT/rules/testing.md` for test shape instead — which is what the
always-on rule requires anyway (Q10). So `#sign-in` loses nothing.

**Belongs upstream**, with whoever maintains that package: the file wants repairing and the
`eslint-disable` wants removing rather than the example being left to lint-silence. Eighth of the
upstream family after Q13, Q14, Q15, Q17, Q19, Q20 and Q24 — and the first one that is a defect in
the *guidance* rather than in code or tooling.

## Q37. Every id the API returns is `BIGINT` in the database and `Int!` in the contract, and no test asserts its type

<!-- spec: none -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

Raised by the agent implementing `signedInStaffMember` at `#sign-in`'s checkpoint 6, which declined
to add a coercion nothing had asked for and flagged it instead.

Every primary key in §9 is `BIGINT` (`MigrationAttributeFactory.ID_BIGINT`), and every id the pinned
contract returns is `Int!` — `SignInResult.staffMemberId`, `SignedInStaffMemberResult.staffMemberId`,
and at `#expense-entry` also `RecordExpenseResult.expenseId`, `CorrectExpenseResult.expenseId`,
`RemoveExpenseResult.expenseId` and `Expense.id`.

**Under SQLite — which every suite runs on — Sequelize hands a `BIGINT` back as a JS number.
Under MariaDB, which §8 declares as the real store, it can hand it back as a string.** So the
resolvers are only ever exercised against one of the two representations.

**Checked, and it is not a defect.** Two reasons, both arithmetic rather than opinion:

- **`graphql-js` coerces a numeric string for an `Int!` output field**, so the contract holds under
  either representation. A resolver returning `"10110001"` and one returning `10110001` produce the
  same response.
- **Overflow is unreachable.** `Int!` permits up to `2147483647`. This project's allocator hands out
  a 3-digit prefix plus 5 digits — 8 digits, so at most `99999999`, comfortably inside it. And those
  are *seeder and fixture* ids; production ids autoincrement from 1, and §7 foresees 50 members of
  staff with a few hundred expenses each. Nothing approaches 2.1 billion rows.

**So no coercion was added, and that was the right call.** A `Number(...)` on the response path would
be an undocumented transformation that nothing in the tree does, and it would hide the dialect
difference rather than resolve it.

**Corrected at checkpoint 9 — the claim that only SQLite is ever tested was false.** This entry
said `package.json` carries a `test:live` script that "nothing in this project's gates ever runs".
`.github/workflows/test-with-mariadb.yml` runs `npm run test:live` against a real MariaDB service
on **every pull request**, and always has. The proof is a failure rather than a reading: the
MariaDB run of this branch refused
`StaffGraphqlContext › .findUser() › … accessTokenRecord.id: 10100406` while the SQLite run of the
same commit refused a *different* case — which is how Q38's race was diagnosed. Both dialects
execute the same 910 tests at every gate.

**And the passing suite says more than this entry allowed.** Tests assert seeded ids as JS numbers
(`id: 10110001` inside a `toEqual`), and `toEqual` does not equate `'10110001'` with `10110001`. The
MariaDB run passes those assertions, so on this driver and at these magnitudes a `BIGINT` id arrives
as a number, exactly as it does on SQLite. That is evidence, not a guarantee about every magnitude.

**What remains genuinely open is therefore narrower, and is the part worth keeping.** No test
asserts the *type* of a returned id on either dialect, so the two representations are
indistinguishable to the suite rather than untested by it. If a coercion ever does turn out to be
needed it belongs in one place for every operation rather than per resolver. Adjacent to Q24, Q31
and Q38.


## Q38. The id-block convention cannot protect a table the product itself writes

<!-- spec: none -->
<!-- blocking: no -->
<!-- category: convention-gap -->

Found at `#sign-in`'s checkpoint 9, diagnosing an intermittent CI failure. **Fixed in this
repository; raised here because the convention it breaks is not this repository's.**

`hor-bank-id` allocates an exclusive row-id prefix per feature so that two writers never choose the
same id, and `migrations-and-seeders.md` asks every seeder to reserve its own block. `#sign-in` was
allocated `101`. The convention holds perfectly against another writer **choosing** an id. It does
nothing against a writer that **derives** its id from the current maximum — which is what an
auto-increment column does.

    step 1  a test inserts explicit id 10100401     -> table max = 10100401
    step 2  a concurrent signIn mints an access token, auto-incremented,
            and is assigned max + 1                 -> 10100402
    step 3  the same test inserts explicit 10100402 -> UNIQUE violation

So the block's **own first insert** is what hands the auto-increment writer the block's second id.
No choice of prefix escapes it, because the collision is generated by the block rather than
suffered by it. Three tables were exposed — `staff_member_access_tokens`, `sign_in_attempts` and
`staff_member_refresh_tokens` — every one of them a table the product writes on the `signIn` or
`renewAccessToken` path.

**Two properties made it expensive to find, and both are worth recording.**

1. **It is a race, so it moved.** The failing case differed between runs and between dialects, which
   reads as a flaky fixture rather than a systematic fault. The mechanism only became visible after
   the thrown error was printed: Jest reports a `SequelizeUniqueConstraintError` by its `message`,
   which is the bare string `Validation error`, so the log showed a **blank** message above a stack
   in the insert path and named neither the constraint nor the column. The useful text sits in
   `error.original.message` — `UNIQUE constraint failed: staff_member_access_tokens.id` — and
   nothing surfaces it.
2. **A partial fix made it worse.** Removing the ids from two files first took the run from
   intermittent to failing every time, because the collision moved to the suites that still held
   explicit ids in the same tables. The invariant holds everywhere or nowhere.

**The invariant, stated for whoever owns the convention:** an explicit row id is safe only in a
table **nothing but the tests writes**. `staff_members` qualifies — §4 rules sign-up out of scope, so
no product path inserts one, and its 45 explicit ids were kept. A table on any product write path
must leave the id to the database, and a case needing identification should use a natural key it
already has (here `access_token`, which is unique, indexed, and what the lookup uses anyway).

**Where it lands.** `hor-bank-id`'s allocation is still correct and still needed; what it lacks is
the sentence saying which tables it can protect. Same for `migrations-and-seeders.md`'s id-block
rule. Neither is this repository's file. Adjacent to Q33, which recorded two other ways this
shared-database test tree depends on order.

## Q39. Both backend CI workflows passed their test flags to npm instead of to the test script

<!-- spec: none -->
<!-- blocking: no -->
<!-- category: convention-gap -->

Found at `#sign-in`'s checkpoint 9, while reproducing Q38. **Fixed in this repository; raised
because both files came from the backend boilerplate.**

Every Jest step in `test-with-sqlite.yml` and `test-with-mariadb.yml` was written without npm's
`--` separator:

    npm test --seeded --maxWorkers=3 tests/_orders/

npm reads `--seeded` and `--maxWorkers=3` as its own configuration and forwards only the positional
argument. `npm test --dry-run` prints what actually ran:

    > ./test.sh tests/_orders/

**So neither flag had ever reached the script, on either dialect, silently.** Two consequences:

| declared | what happened |
|---|---|
| `--maxWorkers=3` | never applied. `test.sh` appends its own cap only inside the `--empty` / `--seeded` branches; with no mode argument the run falls through to the bottom, where `jestCommand "$@"` carries no cap — so the concurrency CI declares is not the concurrency it uses |
| `--seeded` / `--empty` | never distinguished. The fall-through branch runs `db:seed:dev` unconditionally, so the phase meant to prove behaviour against **master seeds alone** would have run with development seeds loaded |

The second matters more than the first, and it reaches a decision already recorded: checkpoint 5
chose `development/` over `dev-master/` for the staff-account seeders precisely because
`tests/empty/**` runs before `db:seed:dev`, which is what gives §10's "an address with no account"
criterion a phase with no accounts in it (Q29). **On CI that phase would have been seeded.** The
split was real in `test.sh` and absent in the workflow that calls it.

**It cost nothing yet, for a reason that is luck rather than design:** `tests/empty/` does not exist
in this repository, so the phase whose semantics were broken currently holds no tests. Q29's
reasoning was sound about where such a test belongs and wrong about the phase being ready for it.

**Where it lands.** The `--` is added to all ten steps here. The frontend repository is unaffected —
its workflow runs a bare `npm test`. Any other ORT backend on this boilerplate carries the same two
files. Adjacent to Q26, Q31 and Q38: things the built or deployed product carries that no gate of a
feature exercises.


## Q40. The boilerplate's one composable is a bare function, and the redirect this feature needs is inside it

<!-- spec: none -->
<!-- blocking: no -->
<!-- category: convention-gap -->

Raised at `#sign-in`'s checkpoint 10 by the unit opening the frontend, which flagged it rather than
using it or rewriting it. **It becomes a decision at checkpoint 16**, which is what wires what
happens after a successful sign-in.

`.hora/tree/expense-note-frontend-staff.md` records this project's ruling plainly: **"shared logic is
a class under the app's own folders — never a composable and never a bare function."** The
`hof-nuxt` digest's Composables section is overridden by it, and that override is already written
down.

`composables/useRedirect.js` is exactly what the ruling excludes — `export default function
useRedirect ({ defaultPath = '/' } = {})`, returning `{ redirectTo }` closed over two inner function
declarations. It reads `?redirect=` off the route query and navigates, defaulting to `/`.

**It is not dead code, which is what makes this a decision rather than a tidy-up.** The gateway
middleware redirects an unauthenticated visitor to `` `/sign-in?redirect=${to.fullPath}` ``, so the
query parameter this composable exists to read is written on every such redirect. Something has to
consume it after `signIn` succeeds, or a member of staff sent to the sign-in screen from a deep link
lands on `/` instead of where they were going.

| The options | What it costs |
|---|---|
| use it as it is | the one place this project breaks its own class-only rule is the sign-in path, and the next frontend feature has a precedent for adding composables |
| replace it with a class under `app/` | consistent, testable the way everything else here is, and `#sign-in` pays for a boilerplate file it did not write |
| leave it and write the redirect fresh in the page context | two implementations of one behaviour, which is worse than either |

**Not blocking and not this checkpoint's.** Recorded now because the unit that found it was not the
unit that will have to choose, and because the reasoning is invisible from the file itself — nothing
in `useRedirect.js` says a project rule forbids its shape.

Adjacent to Q38 and Q39: a convention this repository inherited rather than wrote.


## Q41. Neither of §10's use cases can be completed on a screen this feature builds

<!-- spec: sign-in -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

Found at `#sign-in`'s checkpoint 11, the pass that asks whether a person can actually perform each
use case **on a screen**. **This is not a spec defect** — it is a statement about what this
feature's acceptance can and cannot claim, recorded so checkpoint 18 does not report it as one.

§10's two use cases:

> - a member of staff opens the app on a Monday morning, signs in with the address and password they
>   were issued, and is signed in — **every screen after that knows who they are**
> - a member of staff who has finished on a shared machine **signs out**, and the next person to open
>   the app is asked to sign in rather than landing in somebody else's account

**Both end outside `#sign-in`.**

| | Why |
|---|---|
| "every screen after that" | there is no screen after that. `#sign-in` owns exactly one screen, §10.2's, and the only other route is `/` — still the boilerplate stub with an empty template. The clause refers to screens `#expense-entry` and `#monthly-summary` own |
| "signs out" | **§11.2 puts the `signOut` call on the expense-entry screen.** §10.2's screen is explicitly "For: a member of staff who is **not** signed in", so there is nowhere in this feature for a sign-out control to live — a signed-in person never sees this feature's only screen |

**The spec is coherent and nothing should change in it.** The paths are complete at **version**
level: §11.2 carries the control, and the later screens are what "every screen after that" means.
What is not true is that `#sign-in` can demonstrate either one end to end.

**Where they actually close.** At `#expense-entry`'s gate, or at the whole-version sweep — which is
the run `_plan.md` already reserves for behaviour spanning several features, and which is the only
run that judges §14's version-wide criteria.

**This corrects a claim in checkpoint 9's own record.** That record says the screen half of these
two use cases "is checkpoint 18's". It is not: `#sign-in`'s checkpoint 18 cannot close them either,
for the reason above. The claim was written before anybody asked *which screen has the button*.

**Why neither earlier pass could have caught it, which is the part worth keeping.** Checkpoint 2
read the use cases against the **spec**, where §11.2's `signOut` row makes the path complete.
Checkpoint 9 read them against the **API**, where the `signOut` operation exists, is tested and
works. Both were right about what they were asked. **Only "which screen has the control" reaches
this**, and no gate before 11 asks it.

**What checkpoint 18 should do with this.** Not report a missing sign-out control as a defect of
`#sign-in`, and not mark §10's use cases as passed either. Record them as reached-as-far-as-the-
feature-goes, with the remainder owed to `#expense-entry`. Adjacent to Q40, also deferred to a later
frontend checkpoint.


## Q42. Twenty equipped component skills document a package the boilerplate does not ship

<!-- spec: none -->
<!-- blocking: no -->
<!-- category: convention-gap -->

Found at `#sign-in`'s checkpoint 12, the first checkpoint in this project that needed a text field.
**Resolved for this repository by a user decision; recorded because the cause is upstream and every
project built from the same boilerplate meets it identically, at the same checkpoint.**

### What disagrees with what

`furo-boilerplate-nuxt 2.1.0` gives this row `@openreachtech/furo-nuxt ^2.0.0`, which depends on
`@openreachtech/furo ^1.11.0`. **Neither package contains a single `.vue` file** — verified by
counting, not inferred — and `components/` ships empty but for a `.gitkeep`.

Every one of the **twenty** `hof-cp-*` skills opens with the same clause: *"in a repo that consumes
`@openreachtech/furo-vue`"*. Button, text field, control block, select, table, dialog, toast,
tabs, and the rest. Their examples import from that package — nine such imports in the two skills
read closely.

**`@openreachtech/furo-vue` is referenced nowhere in this project**: not `package.json`, not the
boilerplate, not `.hora/tree/expense-note-frontend-staff.md`, not `specs/`. Only the skills name it.

So the skills and the boilerplate disagree about what the stack is, and **the disagreement is
invisible until a checkpoint needs a component.** Checkpoints 1 to 11 all passed without touching
it. Checkpoint 12 cannot.

### Why this is a different kind of arbitration from the eighteen already logged

The eighteen recorded above are all **an equipped skill versus an always-on rule in
`D:\ORT\rules\`**, and Q10 settles every one of them: the rule wins. There is a standing authority,
so the arbitration is mechanical once the conflict is spotted.

**This one has no adjudicator.** A boilerplate is not a rule, a skill is not a rule, and neither
`specs/` nor `D:\ORT\rules\` says anything about which component library a frontend uses. Q10 does
not reach it. That is precisely why it went to the user rather than being decided in a unit — there
was nothing to appeal to, and the two readings led to materially different work:

| | |
|---|---|
| add `@openreachtech/furo-vue` | twenty skills become applicable; the stack gains a dependency the boilerplate did not choose |
| hand-build the components | the declared stack is untouched; a text field, a password field and a button get reinvented, and every table, select and dialog after them |

**Decided by the user: add it.** The evidence that supported it — three sibling ORT frontends
already depend on it (`crm-kit-frontend`, `hora-ecosystem`, `hora-kit-homepage`), it is actively
maintained at 1.3.2, it carries no lifecycle install scripts, and it ships 52 components including
`FuroEmailField` and `FuroPasswordField`, purpose-built for exactly this screen.

### Where it lands, and why not here

**Upstream, and one of two places.** Either `furo-boilerplate-nuxt` should ship what the skills
assume, or the skills package should declare the dependency it documents. This project fixed its own
instance in one commit; **that fix reaches one repository.** The next project created from the same
boilerplate hits the same wall at its own checkpoint 12 and has to make the same call with the same
absence of an adjudicator.

**The same shape, independently, in a second place:** one `js-yaml` advisory has now been fixed
separately in the app repository and the backend row and is still open in the frontend — three
repositories, one advisory, three fixes, because the fix lives in a lockfile rather than in
`renchan-boilerplate` and `furo-boilerplate-nuxt`. Different subject, identical propagation. Not
chased here; noted because two instances make it a pattern rather than an incident.

Adjacent to Q38 and Q39, both also conventions this repository inherited rather than wrote.


## Q43. The spec says the access token is held in memory; the boilerplate persists it to `localStorage`

<!-- spec: sign-in -->
<!-- blocking: yes -->
<!-- category: contradiction -->

Found at `#sign-in`'s checkpoint 13, reading `app/constants.js` before wiring anything. **Raised
before any code depends on it, which is the only reason it is cheap to settle.** It decides how
checkpoint 16 wires the session, and it has a security dimension, so it is marked blocking for 16
rather than for the feature.

### The contradiction, both sides quoted

§6, Terminology:

> **access token** — the credential the client sends on a request header, and what proves a session
> for every operation except the three §7 names. Lives fifteen minutes (§9.6). **Held in memory,
> never in a cookie.**

The boilerplate, at `node_modules/@openreachtech/furo-nuxt/lib/tools/AccessTokenClerk.js`:

```js
static createStorageClerk () {
  return StorageClerk.createAsLocal()      // -> window.localStorage
}

static get STORAGE_KEY () {
  return 'access_token'
}
```

and this project's own `app/constants.js` already carries `STORAGE_KEY.ACCESS_TOKEN: 'access_token'`
to match.

**`localStorage` is neither memory nor a cookie**, so the clause is not violated on its letter by the
"never in a cookie" half — and is plainly violated on the "held in memory" half.

### Why it is not a wording quibble

**The two designs differ in what an XSS gets.** The refresh token is an httpOnly cookie precisely so
that script cannot read it (§9.7). An access token in `localStorage` is readable by any script on the
origin, so a single XSS yields a working credential for up to fifteen minutes. Held in memory, it
does not survive to be read by a later injection and is not reachable from another tab.

**They also differ in what a reload does, and §10 has a criterion about exactly that.** "A session
survives a page reload":

| | How a reload is survived |
|---|---|
| `localStorage` (boilerplate) | the token is simply still there — no network call |
| memory (§6) | the token is gone; the client presents the refresh **cookie** to `renewAccessToken` and gets a fresh one |

**§10.2 describes the second one.** It says `renewAccessToken` "belongs to the GraphQL client layer,
which calls it when an access token has expired, transparently, on whatever screen happens to be
open", and that "a client that never renews leaves every session dying after fifteen minutes with
nothing on screen to explain it." That machinery exists **because** the token is not persisted. If it
were in `localStorage`, the renew path would be needed only after fifteen minutes rather than after
every reload.

So §6's "held in memory" is not an offhand phrase — the rest of the feature is shaped around it.

### What it collides with

**`middleware/000.gateway.global.js` already calls `AccessTokenClerk.create().existsToken()`**, which
reads `localStorage`. Under an in-memory design that call answers `false` on every fresh page load,
so the gateway would redirect a signed-in member of staff to `/sign-in` after every reload — before
the client layer has had a chance to renew. **The gateway is boilerplate this feature did not write
and checkpoint 10 deliberately did not touch**, and it is where the two designs actually meet.

### Why this has an adjudicator, unlike Q42

Q42 had none — no rule reached it. **This one does: the spec is the authority, and it is explicit.**
So the direction is settled even though the work is not: the access token is held in memory, and the
reload path goes through `renewAccessToken` against the refresh cookie.

**What is genuinely open is only the mechanism**, and it is checkpoint 16's:

1. hold the token in a module-level store or an app-share singleton, and give the gateway a check
   that tolerates "no access token yet, but a refresh cookie may exist" — which means the gateway
   can no longer decide by `existsToken()` alone
2. keep `AccessTokenClerk` for its interface but substitute an in-memory `StorageClerk`, since
   `create()` accepts `storage` as an injected parameter and defaults it — the seam is already there
   (`StorageClerk` also ships a `createAsSession()`, which is **still not memory**)

Option 2 is the smaller change and uses a seam the library deliberately exposes. Neither is decided
here.

**Not fixed at this checkpoint, deliberately.** Nothing yet stores or reads an access token in this
application — checkpoint 10 built a route and a title. Recording it before 14 and 16 build on the
boilerplate's assumption is the whole value; discovering it afterwards would mean unpicking the API
client and the gateway together.

Adjacent to Q40, which is the other boilerplate behaviour this feature has to decide about at 16.


## Q44. No `.vue` file in this repository can be unit-tested

<!-- spec: none -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

Found at `#sign-in`'s checkpoint 15 by the unit building the screen, which wrote a mount test for its
one new component, hit this, **deleted the test rather than encode the defect**, and reported it.
Verified independently from the main session before recording.

### What happens

`jest.config.js` transforms `.vue` with `@vue/vue3-jest` (29.2.6), and `.babelrc` compiles the test
file to CommonJS with `@babel/preset-env`. **vue3-jest's output is not interop-flagged** — it carries
no `__esModule: true` — so babel's `_interopRequireDefault` wraps the whole module object rather than
unwrapping it, and a default import yields the namespace instead of the component.

Probed directly, on the real component:

```
PROBE keys:          [ 'default', 'render' ]
PROBE has .props:    undefined
PROBE has .default:  object
```

So `import AppRefusalMessage from '.../AppRefusalMessage.vue'` gives `{ default, render }`.
`.props` is undefined, every prop falls through as a plain attribute, and a mount renders nothing
useful.

### Why it has never been noticed

**No `.vue` file has ever been tested in this repository** — `grep` over `tests/` finds not one import
of a `.vue`. The boilerplate ships none, and every test so far targets a class. So nothing regressed;
this is a latent gap that the first component to want a test walked into.

### Why it was not fixed at checkpoint 15

Four reasons, and the last is the one that decides it:

1. The exit condition was met without it.
2. The one new component, `AppRefusalMessage`, **holds no logic** — no context class, no computation,
   no decision — so there is no behaviour a test would assert.
3. `jest.config.js` is a shared file, and changing test infrastructure mid-checkpoint affects every
   suite in the repository.
4. **There is nothing to verify a fix against.** A change to the transform with no component whose
   test currently fails is a change whose correctness cannot be demonstrated — which is exactly the
   class of "green but meaningless" this feature has spent the day recording.

**Where it lands.** `#expense-entry` adds components that *do* hold behaviour, and its checkpoint 13
or 15 is the natural place: there will be a real failing test to fix against, which is the condition
this checkpoint lacked. The workaround to avoid — writing `.default` into test imports — encodes the
bug into every test file and should not be taken.

Adjacent to Q42: another thing the boilerplate hands every project built from it.

## Q45. furo's controls assume a CSS reset that nothing in the stack ships

<!-- spec: none -->
<!-- blocking: no -->
<!-- category: convention-gap -->

Found at `#sign-in`'s checkpoint 15, and **worked around on one screen rather than fixed**, because
the fix is project-wide and does not belong to a feature.

`FuroTextField` and its siblings are written for `box-sizing: border-box` — `width: 100%` plus padding
plus a border. **Nothing supplies it:**

- `@openreachtech/furo-vue` ships no `box-sizing` anywhere in `lib/assets/css/` — grepped, not assumed
- `@openreachtech/furo-nuxt` 2.x ships **no stylesheet at all** (Q42's neighbour: 1.x shipped
  `0100.reset.css`, and the sibling `crm-kit-frontend` still loads it — but it is on furo-nuxt 1.x)
- this application's `assets/css/main.css` is 7 lines of iOS input sizing and declares no reset

**So every furo control overflows its container, at every viewport**, in any project on
`furo-boilerplate-nuxt 2.1.0` that uses `furo-vue`. Checkpoint 15 worked around it with three local
`box-sizing: border-box` declarations scoped to the sign-in screen.

**Why it was not fixed centrally here.** A reset in `main.css` changes the box model of every element
in the application at once. On a repository with one screen that is nearly free, and that is exactly
why it is tempting — but it is a project-wide structural decision arriving as a side effect of
building a form, and the same reasoning that kept `@layer` out of checkpoint 15 applies unchanged.

**Where it lands.** Either `main.css` gains a reset — the removal condition for the three local
declarations, which should be deleted the moment it does — or furo-nuxt 2.x restores the stylesheet
it used to ship. The second is upstream and is the better fix, since every consumer of `furo-vue`
has this problem.

Adjacent to Q42 and Q44: three separate things the frontend boilerplate leaves to each project to
discover independently.
