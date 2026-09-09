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
