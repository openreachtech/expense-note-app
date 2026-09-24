# hof-e2e-test-specification
<!-- hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-e2e-test-specification/ -->

**Read the source above whenever this leaves a question open.**

Author and maintain the **E2E test specification** — the durable, maintained list of scenarios
stating what must be true of the product, flow by flow, derived from the API surface. Use when
creating it for the first time, adding scenarios for a new capability, or reconciling it after a
flow changed.

The boundary, in one line: **this specifies what must be true; it does not say how to click it.**

## Two principles

- **A scenario is a claim about the product, not a script for a robot.** Executable by someone with
  no access to the codebase; still correct after the screen is rebuilt. Both properties are lost the
  moment a selector, a wait, or an internal state appears in it.
- **Coverage is derived, not remembered.** The inventory comes from what the product exposes — the
  API surface, the provocation catalogue, the role list — so a missing scenario appears as a
  **difference** rather than waiting for somebody to notice an absence.

## Where the specification lives

```
<project-root>/ai/specs/e2e/
├── index.md            the flows, their identifier prefixes, and how many scenarios each holds
└── <flow>.md           one file per flow
```

- **A third kind of document** — **maintained**: amended when the product changes, read by every
  later run. Its own directory keeps it from being treated as a dated report or as fixed facts.
- **One file per flow** — the unit that changes, that a reader reviews, and that two people edit at
  once.
- **Results never go in here.** Which scenario passed on which day belongs to the dated report.

### As this repository actually stands (observed 2026-09-24 — not skill text)

| Thing the skill names | Actually here? |
| --- | --- |
| `expense-note-frontend-staff/ai/specs/e2e/` | **Does not exist.** `ai/` holds only `contexts/` (`uiux-context.md`, `uiux-context-expense-entry.md`, `uiux-context-monthly-summary.md`). No `index.md`, no flow file, no scenario identifier has ever been allocated |
| Phase 1 "project declaration" | **No document of that name.** Nearest: `specs/1.0.0/spec.md` (monorepo root) and the `ai/contexts/uiux-context*.md` files, which do carry roles and flows in `[spec]`-marked answers |
| Phase 1 "capability inventory" | **Does not exist.** The skill's sanctioned fallback does: the API surface at `expense-note-frontend-staff/app/graphql/client/` — mutations `correctExpense`, `recordExpense`, `removeExpense`, `renewAccessToken`, `signIn`, `signOut`; queries `expenseCategories`, `expenses`, `monthlyExpenses`, `signedInStaffMember` |
| Phase 1 seed set | Exists, in the **backend** repo: `expense-note-backend/sequelize/seeders/development/` (staff_members, staff_member_secrets, staff_member_password_hashes, expenses) |
| `scripts/check-scenario-coverage.mjs` (Phase 4) | **Does not exist** anywhere in this monorepo or in any installed `@openreachtech/hora-skills-ort-*` package. Gate 4 has no runnable check here |

So a scenario list produced inside a run rather than read from a maintained document is **consistent
with the tree** — no such document has ever been written. The skill's answer to a missing Phase 1
input is not to stop: **name what was missing together with what it costs** (Gate 1); and where the
capability inventory is absent, read the API surface directly and **say in the index that the
derivation was made that way** — because a coverage report reading "complete" against a hand-scanned
surface is the strongest false confidence this document can produce.

## Scenario identity

Identifier `<FLOW>-<nn>`, in the heading, and **the identifier is the join key** between this
document and every report that cites it.

- **Never reuse, never renumber** — not even to close gaps after a retirement.
- **Retire, do not delete.** The number stays spent.
- **A prefix is a namespace, not a label.** A renamed flow keeps its prefix; the index records the
  rename. Splitting a flow **moves** scenarios without renumbering them (the new flow gets a new
  prefix for its *future* scenarios); merging keeps both prefixes. A file holding two prefixes for a
  while is correct.
- **The success condition is the identity of the scenario.** Steps may change freely; the moment the
  success condition changes it is a different claim wearing an old number.

### Identifier bands

| Band | Source | Reading a gap |
| --- | --- | --- |
| `01`–`49` | the capability inventory | — |
| `51`–`79` | the provocation catalogue | a flow with nothing here has never been specified against failure |
| `81`–`99` | the role list | a flow with nothing here has no permission boundary written down |

Numbers are allocated in order and never renumbered to close gaps. Gaps mean retirement, and a gap
is information.

## Phase 1 — Gather the inputs

- **The project declaration** — core flows with their success conditions, the roles, the
  credentials, and the operations deliberately absent from the UI.
- **The capability inventory** — operation, entity and verb for everything the product can do. Where
  the project has no such matrix, **read the API surface directly**; it is the same source.
- **The data the scenarios start from** — the seed set the environment loads, so a precondition can
  name rows that will actually be there.

> **Gate 1** — all three are present, or the missing one is named together with what it costs. A
> specification written without the role list cannot contain a permission scenario, and must say so
> rather than appear complete.

## Phase 2 — Derive the scenario inventory

**Produce the list of scenarios before writing any of them.** Deriving and writing at once produces
the scenarios that were easy to think of, and no way to tell what is missing.

| Source | Produces | Question it answers |
| --- | --- | --- |
| the capability inventory | one scenario per reachable operation, grouped into flows | can a user do the things the product can do? |
| the provocation catalogue | a named scenario per failure worth surviving | does the product tell the truth when it cannot? |
| the role list | per role, that a restricted operation is neither offered nor accepted | is the boundary guarded on both sides? |

- **Group the operations into flows, then write scenarios per flow — not per operation.** One
  scenario covers several operations, and one operation appears in several scenarios. **Coverage is
  *at least one*, never *exactly one*.**
- Two clusterings, usually both present: **by the life of an entity** (created, read back, changed,
  read back, removed, restored) and **by the journey a person is on** (the cross-entity scenarios,
  which find the gaps between two teams' work).
- **Failure scenarios are first-class**, written and numbered like any other.
- **A flow is one scenario, not one assertion.**

### What makes an entity's coverage complete

| Rule | The scenario it forces |
| --- | --- |
| What can be created can be read back | Create it, then find it again from a fresh screen — not from the response of the create |
| What can be updated can be read back | Change it, leave, return, and see the change |
| What can be deleted has a restore, or a confirmation that names the consequence before it happens | Delete and restore; or delete and, before confirming, be told how many other things go with it and which records they are |
| A list is paginated and can say it is empty | Reach the second page; and see the empty list of something the actor legitimately has none of |
| An `execute`-style operation can say accepted, finished, and failed | One scenario for the successful run to completion; the failed one belongs to the failure band |
| A verb reachable by a role is reachable *only* by that role | Belongs to the permission band |

An entity that legitimately lacks a verb needs no scenario for it — but it needs a line in the flow
file's exclusion table saying so, because "no scenario" and "no capability" otherwise look the same.

### Selecting failure scenarios

- **Write a scenario for a failure the product promises to survive** — a dependency it retries, an
  input it validates, a conflict it detects.
- **Write a scenario for a failure whose silent version would read as a product bug.**
- **Do not write a scenario for a failure with no user-visible contract** — asserting one is
  inventing requirements.
- Every flow gets at least one entry in the failure band.

### Roles

- **What the role must reach** — its own area is reachable from where it lands after signing in. One
  per role. Catches the role that technically has permission and no route.
- **What the role must not reach** — per restricted operation, that it is **neither offered nor
  accepted**: both halves in one scenario. Hiding a control is not a guard, and guarding without
  hiding is an invitation.
- Too many combinations to be exhaustive → cover **the boundary that would hurt most if it were
  open** per role, name that choice in the index, and let the rest be a documented gap.

### Exclusions, and the two places they live

| Kind | Where | Meaning |
| --- | --- | --- |
| This flow will never cover this operation | the flow file's exclusion table | a decision, with its reason |
| No scenario for this yet | the index's coverage-gaps table | an acknowledged debt, with what it is waiting for |

**A reason is mandatory on an exclusion.** Exclusions are checked **in both directions**: one naming
an operation that no longer exists is reported, not ignored.

> **Gate 2** — every operation in the inventory appears in at least one scenario, or carries an
> exclusion **with a reason**. A reason-less exclusion is an oversight that has been written down.

### Two shapes that mean the derivation went wrong

- **One scenario per operation** — a list of API calls dressed up as a UI. Merge them into flows.
- **A flow with thirty normal-path scenarios** — it is several flows. Split it before writing;
  identifiers are permanent.

Rough shape: a handful of normal-path scenarios, at least one failure scenario, and a permission
scenario per role the flow distinguishes between.

## Phase 3 — Write each scenario

**The field labels are a contract, not decoration.** The identifier lives in the heading and the
operations live under `Covers`, because the coverage check reads exactly those two places.

| Field | Required | Rule |
| --- | --- | --- |
| identifier | yes | In the heading, `<PREFIX>-<nn>`, from this flow's band. Never reused |
| title | yes | One sentence naming **what the user achieves**. Not "test that…", not "verify…" |
| `Actor` | yes | A role from the declaration's role list, by its name there. Never "a user" |
| `Status` | yes | `active`, or `retired <date> — <reason>` |
| `Covers` | yes | The operation identifiers this scenario exercises, as they appear in the capability inventory. **This is the join to coverage** |
| `Preconditions` | yes | The state of the world before step 1 — who is signed in, and which **seeded** rows exist. Name the rows; do not describe creating them |
| `Steps` | yes | Numbered, what the user does |
| `Success condition` | yes | Exactly one, observable by the actor. The point at which the flow is complete |
| `Then also observable` | no | Consequences of that same completed flow, visible **elsewhere** in the product |
| `Failure variants` | no | Identifiers of the failure scenarios that branch from this one. Referenced, never inlined |
| `Notes` | no | Traceability to the project's own requirement or screen identifiers |

- **`Success condition` is singular on purpose.** Two conditions mean two scenarios.
- **`Then also observable`** is where a multi-service product is actually specified — a write that
  must reach a search index, derived list, counter or notification, stated in user terms ("the record
  appears in search results"). It does not create a second success condition.
- **Fields deliberately absent:** no per-step expected result; no priority or severity; no
  environment or setup section.

### The file — `<project-root>/ai/specs/e2e/<flow>.md`

```markdown
# Applications (`APP`)

What this flow is for, in a sentence or two, and where a user enters it.

## Excluded from this flow

| Operation | Reason |
| --- | --- |
| `purgeApplication` | Operations-only; reachable from the maintenance CLI, never from the UI |

## Scenarios

### APP-01 — A member submits an application and can see it afterwards

- **Actor**: member
- **Status**: active
- **Covers**: `createApplication`, `applications`, `application`
- **Preconditions**:
  - signed in as the seeded member account
  - the seeded catalogue holds the programme "Spring Intake", open for applications
  - this member has no application for that programme
- **Steps**:
  1. Open the programme "Spring Intake" from the programme list.
  2. Start an application.
  3. Fill the required fields with values a real applicant would give.
  4. Submit.
- **Success condition**: the application appears in the member's own list with status "submitted", and opening
  it shows the values that were entered.
- **Then also observable**: the programme's remaining-places count has decreased by one; the application is
  findable by the member's name in the administrator's search.
- **Failure variants**: APP-51, APP-52
- **Notes**: covers requirement REQ-114.

### APP-51 — Submitting fails while the application service is unreachable

- **Actor**: member
- **Status**: active
- **Covers**: `createApplication`
- **Preconditions**: as APP-01, up to and including step 3
- **Steps**:
  1. Make the application service unreachable.
  2. Submit.
- **Success condition**: the screen says the submission did not go through and what to do next; the entered
  values are still on screen, and nothing appears in the member's list.

### APP-81 — A member cannot reach another member's application

- **Actor**: member
- **Status**: active
- **Covers**: `application`
- **Preconditions**: two seeded member accounts, each with one submitted application
- **Steps**:
  1. Signed in as the first member, open the address of the second member's application directly.
- **Success condition**: the application is not shown, the screen says why, and the request is refused by the
  API as well as hidden by the UI.

### APP-04 — *retired*

- **Status**: retired 2026-03-11 — the paper-form upload it covered was removed from the product.
```

- **A scenario never spans two files.** One that seems to belong to two is either two scenarios or a
  sign the flows were cut in the wrong place.
- **`Preconditions: as APP-01, up to and including step 3`** is the one permitted shorthand, and only
  for a failure variant against its own normal-path scenario.
- **A retired scenario keeps its heading and its `Status`, and loses everything else.**

### The index — `<project-root>/ai/specs/e2e/index.md`

```markdown
# E2E Test Specification — index

| Flow | Prefix | File | Normal | Failure | Permission | Retired |
| --- | --- | --- | --- | --- | --- | --- |
| Applications | `APP` | [applications.md](./applications.md) | 6 | 4 | 2 | 1 |

## Coverage gaps

| Operation | Why not yet |
| --- | --- |
| `reindexProgrammes` | Specified once the operator screen exists; today it is only reachable from a job |
```

**The counts per band are the point of the table.** `Permission 0` on a flow with roles, or
`Failure 0` on anything, is visible at a glance and needs an answer. **Coverage gaps are recorded
here, not in the flow files**; the flow file holds only what a flow deliberately excludes, with its
reason.

### Steps and expectations

Two tests for every sentence: **could a person with no access to the codebase perform this?** and
**will it still be correct after the screen is rebuilt?**

- **Name things as the user sees them** — not by identifier, not by row position, not by route.
- **One action per step.** A step containing "and" is usually two.
- **Skip the navigation the product will change anyway.**
- **Describe a value by what makes it interesting, not by its literal** ("a quantity larger than the
  remaining stock"), unless the literal is the point.
- **Addresses appear only when the scenario is about the address** — a deep link that must survive a
  reload, or direct access to something a role may not reach.
- **Observable means the actor can see it** — on screen, or somewhere they can navigate to. Not the
  database, not the network panel, not the console. Where a consequence is visible only to an
  administrator, that scenario's actor is an administrator, or signing in as one is a step.
- **Quantify where a number is what makes it checkable** — "the count reads 23", not "the list
  updates".
- **A negative needs a place to be absent from.** "Nothing appears in the member's own list" can be
  checked; "no error occurs" cannot.
- **Say what the user has, not what the system did.**
- **Length is a signal.** More than about ten steps on a normal path means two flows, or the
  interface being specified rather than the outcome. Failure and permission scenarios are shorter
  still — most are a precondition, one or two steps, and an expectation.

### Asynchrony without waits

| The product promises | Write it as | What a failure means |
| --- | --- | --- |
| it appears on its own | "appears in the list without the user doing anything" | the live update is broken |
| it appears when the user next looks | "appears after opening the list again" | acceptable by design; say so, so nobody reports the absence of live update as a defect |
| it appears eventually, and the wait is visible | "the row shows *processing*, and later shows the result" | a wait with no terminal state |

**Never write a duration.** The harness owns timeouts. Where the product genuinely promises a bound,
that bound belongs in the declaration as a requirement, and the scenario cites it.

### Never in a scenario

| Never | Write instead |
| --- | --- |
| Selectors, CSS classes, test identifiers | The visible name of the thing |
| Explicit waits and durations | The state that must be observed, per the table above |
| Assertions on the database, an API response, or component state | What the actor sees as a result |
| Endpoint, table, component or class names | What the user did and got |
| "Displays correctly", "works as expected", "no errors" | The specific thing on screen |
| Conditional steps — "if a dialog appears, dismiss it" | Fix the preconditions, or split |
| "Repeat for each…" | Enumerate the cases that matter, or write one scenario about the set with a counted expectation |
| Another scenario's steps by reference | Spell them out. The single exception is a failure variant citing its own normal path |

> **Gate 3** — no scenario contains a selector, an explicit wait, or an expectation the user cannot
> observe; every scenario has exactly one success condition and one owner flow.

## Failure scenarios (the `51`–`79` band)

**What every failure scenario asserts — four claims:**

1. **Something appears.** (Where the failure is a wait rather than a refusal, this becomes **the
   wait ends**: the screen reaches a terminal state and stops promising a result that will never
   arrive.)
2. **It says what happened, in the user's terms.** Not a code, not "an error occurred" — what did
   not happen, and to what.
3. **It says what to do next.** Retry, choose another file, ask an administrator, continue another
   way.
4. **The user's work is still there.**

**Name a provocation without binding to the environment:** write **"while the application service is
unreachable"**, not "run the command that stops the container".

### The catalogue

| Provocation | Stated as | What the scenario asserts |
| --- | --- | --- |
| A dependency is unreachable | "while the *X* service is unreachable" | The failure is named on screen, the entered work survives, and the rest of the product remains usable |
| A dependency answers slowly | "while *X* is responding slowly" | Something says work is in progress; the screen is not frozen and the action cannot be fired twice |
| Input is invalid | "with a value that exceeds the limit / a required field left empty / the wrong kind of value" | The message is **beside the field**, the rest of the entry is kept, and nothing was saved |
| Input conflicts with existing data | "with a name that already belongs to another record" | The conflict is named, and the user is told which record it conflicts with |
| A concurrent change has occurred | "after another user has changed the same record" | The user is told their view is stale, and does not silently overwrite the other change |
| The actor lacks permission | "signed in as a role without that permission" | Not offered on screen **and** refused by the API — one scenario, both halves |
| The session has expired | "after the session has expired" | Told they are signed out, sent somewhere they can sign in, and returned to what they were doing — or told plainly that they cannot be |
| Accepted work fails later | "when the background work fails after being accepted" | The screen reports the failure and **stops waiting**; the user can retry or abandon |
| The action is repeated | "by submitting twice in quick succession" | One result, not two, and no crash on the second press |
| Work is abandoned mid-flow | "by leaving the screen with unsaved changes" | Warned before leaving, by in-app navigation **and** by closing the browser |
| An artifact cannot be rendered | "with a file the viewer cannot display" | Says the format is not displayable — never a broken image, which reads as "your file is damaged" |

Not every row applies to every flow. Pick the ones the flow can actually meet, and **record the
choice**.

**The four shapes an assertion must rule out:** **silence** (ruled out by claim 1); **a wait that
cannot end** (ruled out by asserting the terminal state — which is why "stops waiting" must appear
in the words); **the wrong state** — a failure rendered as empty ("there is nothing here" and "I
could not find out what is here" are different sentences; assert the second, by name); **a
misleading affordance** (ruled out by asserting absence, not just refusal).

**Restore the world.** A failure scenario ends by stating that the provoked condition is undone —
scenarios must be runnable in any order. Where a provocation cannot be undone within the scenario,
say so explicitly and place the scenario last in its flow.

**A failure scenario is not a bug report.** It states what the product **must** do, whether or not it
does today. If the product currently fails it, that is a finding for the review and for the
declaration's known-and-accepted list; the scenario itself does not soften to match.

## Phase 4 — Verify coverage

Run `scripts/check-scenario-coverage.mjs <project-root>`. It enumerates operations, reads the
scenario files, and reports operations no scenario mentions, exclusions that no longer match
anything, and duplicate identifiers. Same contract as the other checks in this family: **0 clean,
1 findings, 2 could not analyse this project — and 2 is not a pass.**

> **Gate 4** — coverage is clean, or every gap is explained in the index.

(Per the repository table above, this script is not present here.)

## Phase 5 — Maintain

**Amend or retire — the decision has one test: does the claim still hold in some form?**

| The change | What to do |
| --- | --- |
| The route through the UI changed; the user can still achieve the outcome | **Amend the steps.** Same identifier — the claim is unchanged, only the way there |
| The screen was redesigned | Usually **nothing**. A scenario that has to be edited for a redesign was specifying the interface |
| The outcome changed — success now means something different | **A new scenario.** The old one retires |
| The capability was removed | **Retire** |
| Two scenarios turned out to be the same claim | Retire one, note which absorbed it |

**Reconciliation belongs to the change that caused it, not to a calendar.**

| What changed | What it obliges |
| --- | --- |
| A new operation exists | Derive its scenarios, or add a coverage-gap entry saying what it waits for |
| An operation was removed | Retire the scenarios that covered it; remove any exclusion naming it |
| A new role exists | One scenario that its area is reachable, and one per boundary it must not cross |
| A flow's shape changed | Amend the steps of its scenarios; check whether the success conditions still hold |
| An exclusion's reason stopped being true | Delete the exclusion and derive the scenarios it was holding back |
| The seed set changed | Check every precondition that names seeded rows — one naming a row that is no longer seeded makes its scenario unrunnable, and it will be reported as unexecuted rather than as broken |

- **An unexecuted scenario has exactly two causes, and both need action:** it could not be performed
  as written (amend or retire it — do not leave it to be reported again), or the run skipped it
  (that belongs in the run's report; a scenario skipped by every run for months is covered by
  nothing).
- **A scenario that has failed for months** is either a defect the project has decided to live with
  (→ the declaration's known-and-accepted list, so runs stop re-reporting it) or a claim the product
  will never satisfy (→ the claim was wrong; change it).
- **Never soften a scenario to match the product.** If the claim was wrong, change it deliberately
  and say why in the index; if the product is wrong, the scenario stays and the failure is reported.

### Signs it has rotted (visible without reading the scenarios)

- **No failure band anywhere** — only happy paths were ever derived.
- **`Permission 0` on a flow whose product has roles.**
- **Nothing retired** — removals were never reconciled.
- **The coverage-gaps table is empty while operations have no scenarios.**
- **Most preconditions say "as *X*, up to step *n*"** — the scenarios can no longer be run
  independently.
- **Scenario titles name screens rather than outcomes.**

## What this skill does not decide

- **How the scenarios are executed.** Manually, or by whatever automation the project has — the
  specification is indifferent, and must stay that way to remain executable by both.
- **Whether a scenario passes.** That is the acceptance review's live pass, and its report is where
  the verdict lives.
- **How the environment is built, started or seeded.** This document only names the data it expects
  to find.
- **Whether the product may ship.** A specification states requirements; it does not weigh them.
