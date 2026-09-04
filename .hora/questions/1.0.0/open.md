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

### Follow-up, once 0.2.0 is published

Move this project's pin from `^0.1.0` to `^0.2.0`, and **delete this question's "not yet
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
