# hof-acceptance-review
<!-- hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-acceptance-review/ -->

**Read the source above whenever this leaves a question open.**

## Scope

Post-implementation acceptance review of an application: whether every operation the backend exposes is
reachable from the UI, whether CRUD per entity is complete, whether affordances do something, whether
failures and waits are told truthfully, and whether baseline usability is present. Runs **after**
implementation, on a tree that already passes its own lint and unit tests — "those are a different
convention's job, and passing them is not evidence that a screen works."

Two principles govern the whole review:

- **Decide what each check can prove before running it.** Every finding carries a verification label;
  anything unproven goes in "couldn't verify" rather than passing quietly.
- **Green is not evidence.** Unit tests never render a template; a 200 on a client-rendered app returns a
  shell; a build succeeds with runtime type errors intact.

## Modes

| Mode | When | Inventory |
| --- | --- | --- |
| Scoped (default) | Straight after implementing a feature | The screens and operations the diff touches, plus the paths that reach them |
| Full | Before a milestone, or on request | Every screen, entity and operation |

Mode changes **the inventory**, not the list of phases. Say in the report which mode ran, "so a scoped pass
is never read as a clean bill of health for the whole app."

## Prerequisite: the live environment

Phase 4 drives the real application against the real services behind it. Before Phase 0, confirm that:

- the application runs locally **together with every service it talks to**, not just the UI;
- the accounts for each role can sign in;
- there is reviewable data present, or a command that puts it there.

**Building that environment is outside this skill.** If it cannot be brought up, the review **does not
quietly continue as a static-only pass**: say so in the capability note, record Phase 4 as not run, mark
Gate 4 accordingly, and leave every finding that depended on running the app under "couldn't verify".

> "A static sweep is a legitimate deliverable — a static sweep presented as an acceptance review is not."

## Phases and gates

| Phase | Does | Gate — fails when |
| --- | --- | --- |
| 0 Capability | Record UI framework, API style (schema-first / spec-first / route files), test runner, whether a browser can be driven; **check** (do not assume) whether the local environment is up; which mechanical checks apply. Output an *audit capability* note that opens the report | **Gate 0** — the capability note exists. "Without it there is no way to tell a check that passed from a check that never ran" |
| 1 Declaration | Read `<project-root>/ai/contexts/acceptance-context.md`. If missing, **create it before continuing** from the schema, filling everything discoverable and marking the rest `TBD`. "Never infer a core flow and then audit against the guess" | **Gate 1** — a run command and per-role credentials are present, **or** Phase 4 is recorded as not run |
| 2 Inventory | Build the capability matrix, taking the operation list from **the API surface**, not from the UI and not from memory | **Gate 2** — every operation classified as reachable, unreachable, or deliberately excluded **with a reason**. "An exclusion without a reason is indistinguishable from an oversight" |
| 3 Static sweep | Run the scripts that apply; for anything that cannot run here, do the check **by reading the code** and label the finding Inferred | **Gate 3** — no unexplained finding, and no stale exclusion |
| 4 Live sweep | Mount every screen in scope in a real browser, walk core flows, provoke failures, record API responses | **Gate 4** — every screen in scope mounted, every core flow completed, console free of uncaught errors throughout. Where a specification exists, its coverage is reported — a scenario left unrun is named, not omitted |
| 5 Report | Write `<project-root>/docs/reports/acceptance-<date>-<scope>.md` | — |

**A failed gate is reported as a failed gate. Do not summarise a run with a failed gate as a pass.**
Gate 4 records `pass / fail / not run`.

## Phase 3 — the static sweep

Every script: **the project root is its only argument**, and all share one contract.

```
node <script>.mjs [project-root]     # argument defaults to '.'
```

| Exit | Means |
| --- | --- |
| `0` | clean **within the reach stated in the script's own header** |
| `1` | findings |
| `2` | could not analyse this project — **not a pass** |

> "Exit code 2 is not a pass — record it under 'couldn't verify', then do the check by reading the code and
> label the finding Inferred." Treating `2` as clean "is the same mistake as citing a 200 for a blank screen."

**Each script's own header states what it can and cannot see. Read it before quoting the result.**

Scripts live in `.claude/skills/hof-acceptance-review/scripts/`.

| Script | What it catches | Exits 2 when | Reach / limits |
| --- | --- | --- | --- |
| `find-unused-operations.mjs` | An operation with a client built for it that nothing imports — **and a stale exclusion** (one that no longer describes anything) | no `*Launcher.js` under `app/graphql/client` or `app/restfulapi` | Operations = `*Launcher.js` files; consumers = `app`, `pages`, `components`, `composables`, `layouts`, `middleware`, `plugins`, `stores` (the client dirs themselves skipped). Reads the exclusions from the declaration's "Deliberately absent from the UI" table, matching each row's first cell as the launcher name and its second as the reason |
| `find-unreachable-screens.mjs` | A screen nothing navigates to, reachable only by typing its address | no `.vue` file under `pages/` | A path counts as referenced when its **literal** appears anywhere outside the screen's own directory. Dynamic segments (`[id]`, `[[id]]`, `[...all]`, `:id`) are stripped; `/` is skipped. Skips `node_modules`, `tests`, `test`, `.nuxt`, `.output`, `.git`, `dist`, `coverage`, `ai`. **The entry point and the sign-in screen will appear here** unless something else mentions them — record those as accepted in the declaration rather than teaching the script which paths are special |
| `find-orphan-template-members.mjs` | A template reading a `context.<member>` the paired Context class does not define | no `.vue` file binds `context` to a Context class | Walks the `extends` chain **including into `node_modules`**, so members inherited from the framework base are not reported |
| `find-unguarded-slots.mjs` | A container rendering slot content while closed, breaking its callers on mount | no component declares a visibility prop | Visibility props matched by name: `is`/`has`/`should` + `Shown`/`Showing`/`Open`/`Opened`/`Visible`/`Expanded`/`Active`. The failure belongs to the container, not the caller, which is why it is checked here |
| `find-emit-mismatches.mjs` | Three mistakes: an event declared and never emitted; an event emitted as a bare string rather than through the declaration; an event declared on the context but **missing from the component's own `emits` registration** (Vue then treats a caller's listener as a native DOM listener — which happens to work for `click` on a `<button>` root and silently does nothing for any event a DOM element does not raise) | no context declares its events in one place (`static get EMIT_EVENT_NAME ()`) | Searches `components`, `layouts`, `pages`; contexts are `*Context.js` |
| `find-missing-screen-states.mjs` | A list that cannot say "there is nothing"; a fetch that cannot say "I could not find out" | no `.vue` file under `pages/` | Two rules: a `resolve*State()` resolver must resolve to **all four** of loading / empty / error / ready; a screen that fetches must be able to say empty and error. Fetching detected by `Launcher`/`GraphqlClient`/`RestfulApiClient`/`Fetcher`; listing by `v-for` (only a list can be empty — a form has no empty state). Empty matched by `empty`/`EMPTY`/`hasNo[A-Z]`/`isEmpty`/`EmptyState`, error by `errorMessage`/`hasError`/`ERROR`/`ErrorState` — a project whose shared component is named otherwise makes this **under-report, not over-report** |
| `find-leaked-vocabulary.mjs` | An internal key printed as text; rendering `somethingName` where `somethingDisplayName` also exists | no enumeration-shaped constants and no templates found | **Values are only reported in text position** — the same string used as a prop or compared against is the code doing its job. Enumerations collected from `constants`, `app` |
| `find-unsafe-html.mjs` | A string handed to the browser as markup: `v-html`, `.innerHTML =`, `.outerHTML =`, `.insertAdjacentHTML(`, `dangerouslySetInnerHTML` | no `.vue`/`.js`/`.ts` file under the searched dirs | **Every occurrence is reported. "There is no comment that makes one safe"** — a genuinely required occurrence is recorded as an accepted decision in the project declaration, not as a comment in the file. Commentary is blanked before matching, so prose about the rule is not reported |

Two Phase-3 checks have **no script**:

- **Dead affordances** — a control that renders, is pressed, and does nothing. Where the project already has
  an equivalent check wired into its own scripts, run that rather than duplicating it.
- **Fixture fidelity** — tests passing against a response shape the API never sends. **Cannot be done
  statically**: it needs the recordings Phase 4 produces.

### Known limits observed in this project (not skill text — verify before quoting)

- `find-unreachable-screens.mjs` reports a page-level Nuxt `alias` as a **false positive**: the navigable
  path is the alias, so the file-derived route path is mentioned nowhere.
- Three static scripts exited `2` at `#expense-entry`'s acceptance, and those checks had to be done by
  reading the code (findings labelled Inferred, per the contract above).

## Phase 4 — live sweep (per screen in scope)

1. **Mount it and require the console to be silent. An uncaught exception is a failure, not a warning.**
2. **Walk each core flow to completion** — create, read back, update, delete, and undo the delete. "A flow
   that ends anywhere but its declared success condition is a finding."
3. **Provoke the failure branches**: stop a dependency, submit invalid input, act as a role without
   permission. Judge what the screen says against the failure-honesty criteria below.
4. **Record the API responses** the flows produce — the input to the fixture-fidelity check, and "the only
   honest source for a test fixture's shape."

- Where the project maintains an end-to-end test specification, **that specification is what this phase
  executes**, and the report states coverage against it. Where there is none, derive the walk from the
  declaration's core flows and say in the report that the scope was **derived rather than specified**.
- **A scenario the specification lists and this pass never executed is itself a finding.** Its severity is
  low, but recording it is what keeps a specification from describing tests nobody runs.
- This phase **uses** the local environment; it never builds one. Restart any dependency stopped to provoke
  a failure before the next flow, and leave the environment as the operator handed it over.
- Drive the browser with whatever automation the project has; this skill needs only that each screen be
  mounted, each flow completed, console output captured, and failures evidenced by a screenshot.

## Capability matrix (Phase 2; report appendix)

Columns: **Operation** (the identifier in the API surface) · **Source** (schema, spec, or route file) ·
**Entity** · **Verb** (`create` / `read` / `list` / `update` / `delete` / `restore` / `execute`) ·
**Reached from** (the screen *and the path through the UI*) · **Affordance → handler** (what the user
presses, and what that call reaches) · **Verification** (the report label) · **Note** (or the exclusion's
reason).

Filling it in: enumerate operations mechanically (prefer the script; where it cannot run, read the client
directory or the schema and **mark the rows Inferred**); classify each as reached / unreached / excluded
with a reason from the declaration; for rows claiming `Verified`, name what was observed and where — **a
live pass step, not a reading of the code**; apply the completeness rules per entity and add a finding for
each rule broken.

### What "complete" means

| Rule | Because |
| --- | --- |
| What can be created can be read back | Otherwise the user cannot confirm their own work |
| What can be updated can be read back | An edit whose result is invisible cannot be checked |
| What can be deleted either has a restore, or a confirmation that names the consequence in counts | A destructive action with neither is a trap |
| A list is paginated and has an empty state | An unbounded list breaks at scale; a blank one reads as a fault |
| An `execute` reports accepted, finished, and failed | A screen that can only say "accepted" leaves a failed job looking exactly like one still running |
| Every verb reachable by a role is reachable *only* by that role | A boundary guarded on one side is not guarded |

Not every entity needs every verb; these rules are what make the judgment defensible rather than arbitrary.

### Reachability is a path, not a presence

All of these are unreachable even though the code is present and correct:

- The affordance renders but nothing listens to it.
- The screen exists but nothing navigates there.
- The screen requires a role, and no user can obtain that role through the UI.
- The affordance appears only inside a container that never opens.

**Record the path, not just the screen.** "Reachable from the editor" hides that the only way in is a button
behind a dialog that no longer opens.

## Failure honesty (judged in Phase 4)

**Break things on purpose and watch.** "A failure branch nobody has ever executed is a branch nobody has
ever seen render."

The four ways a screen lies:

| Lie | Shape | Where it hides |
| --- | --- | --- |
| **Silence** | An operation fails and the screen says nothing; the user presses again or walks away believing it worked | A handler returning early on a null answer without setting a message (a dropped connection and a user-pressed cancel arrive the same way); a message written to state that is rendered only inside a container closed during that flow; a response read at the wrong depth — an envelope not unwrapped yields `undefined` for the id, a guard skips the next step, and the operation half-completes |
| **An unending wait** | A spinner or "being prepared" badge with no terminal state | Ask of every wait: **what ends it?** If the answer is "the thing it is waiting for succeeds", the failure case has no exit. Where a request answers the same code for "queued" and "failed", the screen must get that distinction from somewhere else |
| **The wrong state** | A failure rendered as empty. "There is nothing here" and "I could not find out what is here" are different sentences, and only one tells the user to do something | Every place a screen picks between loading, empty, error and ready — especially where the data arrives by more than one route, such as a first load with hooks and a poll without them |
| **A misleading affordance** | The screen invites a gesture it cannot accept, or shows something it cannot render | A drop zone on a screen that listens for no drop; a control offered to a role whose requests will be refused; a document handed to an image element, which paints a broken icon and reads as "this file is damaged" about a file that is fine |

| Provocation | How | Watch for |
| --- | --- | --- |
| Network failure | Stop the API, or block the request | A visible message; no silent early return |
| Dependency down | Stop the database, queue, or storage the flow needs | The failure named, and other flows still usable |
| Invalid input | Exceed a limit, empty a required field, submit the wrong type | The message beside the field, and the input kept |
| Wrong role | Sign in as a role without the permission | Refusal with a reason, not a bounce |
| Slow work | Submit work that takes real time | Progress, a way to cancel, and a terminal state |
| Failed async work | Make the background job fail | The screen says failed, and stops waiting |
| Repeated action | Press submit twice; press undo twice | No duplicate, no crash on the second |

Judging each: **1.** Does anything appear? (Nothing is the worst outcome and the most common.)
**2.** Does it say what happened, in the user's terms? (Not the code; not "an error occurred".)
**3.** Does it say what to do? (Retry, choose another file, ask an administrator, continue another way.)
"A message that fails the third question is still a pass at severity below one that fails the first two.
Rank them, and say which."

## Baseline usability — the floor, not the ceiling

Each item's absence is **a finding, not a suggestion**. Craft above this floor (hierarchy, spacing,
contrast, motion, tone) belongs to the interface-audit convention — do not duplicate it here.
**A `Live` item that could not be exercised is not a pass. Say which, and why.**

| Item | Check | How |
| --- | --- | --- |
| A field whose values are finite and known is a choice, not free text | The value comes from a master table, an enumeration, or a set the code already lists | Static |
| A field whose values are the user's own vocabulary is **not** locked to a list | Suggesting existing values is right; refusing a new one is not | Static |
| A numeric field distinguishes empty from zero | Clearing a box to retype it must not save `0`; `Number('')` is `0` and passes a finiteness test | Static |
| Submitting is disabled while a submit is in flight | Otherwise a second press sends twice | Live |
| A destructive action is confirmed, and the confirmation names the consequence | "Delete?" is not a confirmation; "delete this and the 24 under it" is | Live |
| Leaving with unsaved work warns | Both the in-app route change and the browser's own close | Live |
| Every data region has loading, empty, error and ready | An empty region rendering nothing reads as a fault | Static + Live |
| Empty is worded, not blank | And says what to do next where there is something to do | Live |
| A list is paginated | And says where in the list the user is | Live |
| Internal keys never reach the screen | Status and category names come from display values, not the codes the API branches on | Static |
| Sign in exists | | Static |
| **Sign out exists** | Sessions held in local storage survive closing the browser; on a shared machine the next person is still signed in, and no account can be switched | Static + Live |
| A role's own area is reachable from where they land | | Live, per role |
| Being unable to enter somewhere says why | Bouncing a user back to a form with no message is indistinguishable from a broken button | Live, per role |
| Current position is shown | Breadcrumb, title, or selection | Live |
| A way back exists that is not the browser button | | Live |
| A deep link opens, reloads, and survives being pasted elsewhere | Three separate checks; each fails differently | Live |
| Navigation is present on every screen that needs it | | Live |
| It collapses | A menu that keeps its full width leaves the work squeezed beside it | Live |
| It does not overflow the viewport at a narrow width | | Live, narrow |
| A panel or drawer can be dismissed | And dismissing it returns the space | Live |
| A user without a permission is not offered the control | Hiding it is not the guard, but offering it is a defect on its own | Static + Live, per role |
| The API refuses it as well | Two sides, because either alone is one mistake from open | Live, per role |

Several live items need **more than one role** — a review that could only sign in as one records the rest as
unverified rather than passing them.

## False confidence — signals that are not proof

Read before treating any existing signal as a pass, **including one this skill's own scripts produced**.

| Signal | What it actually proves | What to do instead |
| --- | --- | --- |
| The unit tests pass | The classes behave as called. **No template was rendered** | Mount the screen in a browser and require the console to be silent |
| The route answers 200 | On a client-rendered app, that the shell was served; the app may throw during mount and render nothing behind that 200 | Same. **Never cite a status code as evidence a screen works** |
| The build succeeds | The code parses and bundles; runtime type errors survive it intact | Same |
| Lint passes | The code matches the conventions; a control wired to nothing satisfies every rule | The dead-affordance and orphan-member checks |
| A fixture-backed test passes | The code agrees with the fixture — which, if hand-written, may be a shape the API never sends, and the test then defends the bug | Record real responses in the live pass and compare shapes |
| A checker reports zero | Zero *within its reach*, which includes its exclusion list | Require exclusions to be justified, and fail on ones that no longer apply |
| The screenshot looks right | That state looked right; nothing about the states you did not photograph | Walk the flow to completion; provoke the failures |
| The feature was implemented | Both halves exist — not that they are joined, nor that a user can get to them | The capability matrix, with the path recorded |
| A previous review passed | The app as it was | Re-run; a scoped pass is not a full one |

> **"A check may only be cited for what it structurally observes."** Before running one, state what it can
> prove; after running it, claim no more than that. Anything else goes under "couldn't verify", **which is a
> normal and expected section of a healthy report — not an admission of laziness.**

**Fixture fidelity:** during the live pass record the API responses each flow produces; then, for every
hand-written fixture, ask whether a recorded response has that shape. A fixture with no counterpart is
either untested territory or a fiction the tests are defending. Where an envelope is involved, check that
the code unwraps at the same depth the server wraps at — reading one level too shallow yields `undefined`
rather than an error, which is why it survives both the tests and the type annotations.

## Phase 5 — report

One file per run at `<project-root>/docs/reports/acceptance-<date>-<scope>.md`, written in the language the
reader is using.

### Severity (verbatim)

| Level | Meaning |
| --- | --- |
| Blocker | A user cannot complete a core flow, or completes it and loses work. Ship-stopping |
| Major | A capability is unreachable, or a failure is silent. The product is usable but a documented requirement is not met |
| Minor | Friction, an unclear message, a missing empty state where the region is rarely empty |
| Note | Worth knowing; no action required now |

### Verification labels — every finding carries exactly one (verbatim)

| Label | Means |
| --- | --- |
| Verified | Observed running. Say where — which step, which screen, which provocation |
| Static | Read from code with certainty, no execution involved |
| Inferred | Professional judgment, or a mechanical check that could not run here and was done by reading |
| Unverifiable | Cannot be settled with what this environment provides. Belongs in "couldn't verify", not in the findings |

### Skeleton

```markdown
# Acceptance Review — <app> — <date>

## What this review could prove
Mode: scoped | full — and what the scope was.
Stack: what was detected.
Live pass: ran | not run, and why.
Scenario source: a maintained specification (which one, which version) | derived from the declaration's core
flows. If specified: how many scenarios ran, and which did not.
Roles exercised: which, and which not.
Checks that could not run here: which, and what was read instead.

## Gates
| Gate | Result |
|---|---|
| 0 Capability stated | pass / fail |
| 1 Declaration present | pass / fail |
| 2 Every operation classified | pass / fail |
| 3 Static sweep clean | pass / fail |
| 4 Every screen mounted, flows completed, console silent | pass / fail / not run |

## Findings
Ranked, most severe first. Each one:

### <n>. <one sentence saying what is wrong>
- **Severity** / **Verification**
- **Where**: file and line, or screen and step
- **What a user experiences**: in their words, not the code's
- **Why it happens**: one or two sentences
- **What would fix it**: the direction, not the patch

## Couldn't verify
What was not settled, and what it would take. This section being non-empty is normal.

## Capability matrix
The full table as an appendix.
```

### Writing the findings

- **Lead with the user's experience.** "Uploading appears to do nothing" before "the envelope is not
  unwrapped." The second sentence is for whoever fixes it; the first is what makes them believe it matters.
- **One finding per defect**, even when several share a cause. A shared cause goes in each entry's "why".
- **Do not propose a patch.** State the direction and stop; the person who owns the code chooses.
- **Cite the project's identifiers** where it has them, so a finding can be traced to the requirement it
  breaks. Where it has none, cite files and lines.
- **Name what you did not check** next to what you did, "so an absence of findings in an area is never read
  as a pass in that area."

### After the report

Anything **accepted rather than fixed** goes into the declaration's "Known and accepted", so the next run
does not report it again. Anything **excluded on purpose** goes into the exclusion table **with its reason**
— the checks read that table, and an entry that stops matching is reported rather than trusted.

## The project declaration

`<project-root>/ai/contexts/acceptance-context.md`. Sections: **Purpose** · **Roles** (may reach / must not
reach) · **Credentials for review** (per role — an account, or the command that seeds one) · **Running the
app** (start command + port, environment, dependencies *and how to tell whether each is up*, seed command) ·
**Core flows** (Flow / Starts at / Succeeds when) · **Entities and expected operations** (Create / Read /
Update / Delete / Restore / Notes) · **Deliberately absent from the UI** (Operation / Reason) · **Out of
scope for this review** · **Known and accepted**.

Fill everything discoverable from the codebase; mark the rest `TBD`, and **say in the report which `TBD`
entries limited the review**.

- **"A row without a reason is not an exclusion — it is an oversight that has been written down."** A reason
  is mandatory on every exclusion; the unreached-operation check reads that table, and reports an exclusion
  that no longer matches anything.
- **Credentials are per role.** A review that can only sign in as one role cannot check a permission
  boundary, and must record that as unverified rather than passing it.
- **The dependency list must say how to tell whether each one is up.** "The API is running" is not
  checkable; "answers on port 8001" is.
- If the project keeps requirement or screen identifiers, name that scheme under Purpose and cite the
  identifiers in findings. Where there is no such scheme, cite files and lines instead.

full text: `references/project-declaration.md` — the markdown schema block, verbatim, for creating the file

## What this skill does not decide

- **Whether the code is well written** — the coding conventions' business.
- **Whether a commit may be made** — lint and unit tests before a commit belong to the development-workflow
  convention; this review sits later, at the point of calling a feature done.
- **What the UI should look like** — this review states that a capability is missing, unreachable or
  dishonest, and leaves the design of the fix alone.
- Element-level **accessibility, contrast, token discipline and visual craft** belong to the interface-audit
  convention: hand those findings over, and keep this report on capability, reachability, and truthfulness.
