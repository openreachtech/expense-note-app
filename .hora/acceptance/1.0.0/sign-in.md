# Acceptance — 1.0.0 — #sign-in

## Run 1
<!-- reach: scoped -->
<!-- scope: sign-in (rests on data-model, accepted) -->
<!-- live: no (skipped at the gate — and see the capability note: it could not have run if requested) -->
<!-- reuse: none -->
<!-- not-accepted: none -->
<!-- version-criteria: not in scope (gate) -->
<!-- environment: not required at this reach and not brought up for the application. The MariaDB container WAS brought up at checkpoint 17 and torn down after; the application itself cannot start here (Q24) -->

### Audit capability — what this run can and cannot prove

**Gate 0 of `hof-acceptance-review` is a capability note, and without it a check that passed cannot
be told from a check that never ran.** This one bounds every claim below.

| | |
|---|---|
| UI framework | Nuxt 3, `ssr: false`, `@openreachtech/furo-nuxt` + `furo-vue`, OOP page/context pairing |
| API style | schema-first GraphQL, against a pinned contract in `.hora/contracts/1.0.0/` |
| test runner | Jest, both repositories |
| **can a browser be driven here?** | **No.** And not because none is installed: **no renchan server starts on this machine at all** |

**Q24 is the root cause and it is a one-line defect, not an environment quirk.**
`DeepBulkClassLoader.loadClasses()` passes a raw absolute path to `await import()`, which on Windows
is `D:\…` — protocol `d:` — and the ESM loader refuses it. The same line appears in
`@openreachtech/renchan` 2.9.3 and `@openreachtech/renchan-sequelize` 2.2.2, and it sits on every
class-loading path a server has at startup: models, **GraphQL resolvers**, post-workers, REST routes.

**So the live sweep was skipped twice over** — a gate run skips it by default, *and* it could not have
run had it been requested. Recorded here rather than left to be inferred from a verdict.

**What follows from that, stated plainly: nothing in this product has ever been exercised through a
running server.** Every gate of every feature has been met by unit tests and in-process integration.
"Verified" in this record never means a request crossed a socket.

### Verdict

**passed over 2 of 4 features; 0 not accepted — on unit and in-process evidence only, with the live
half never exercised.**

Two checkpoints inside the feature are recorded as **reached in part** rather than passed, and this
run does not round them up: **14** (the clients match the pinned contract, but "works against the
stub" was unmeetable) and **16** (the paths are driven by real capsules from real response envelopes,
but "shows real data from the actual API" was unmeetable). Both for the same root cause as above.

**§10's two use cases remain reached-as-far-as-the-feature-goes, and this run does not un-correct
that.** The sign-out control lives on §11.2's screen, which `#expense-entry` owns, and "every screen
after that knows who they are" refers to screens this feature does not build. Q41. They close at
`#expense-entry`'s gate or at the whole-version sweep.

### What ran

| Step | Delegate | Result |
|---|---|---|
| environment | — | not required at this reach; the application cannot start here regardless (Q24) |
| unit (backend) | `hor-backend-testing`, `hoc-jest`, `hoc-test-execution` | **720** passed in 41 `__tests__` suites, **195** passed in 8 `_orders` suites. Lint clean |
| unit (frontend-staff) | `hoc-jest`, `hoc-test-execution` | **337** passed in 24 suites. Lint clean |
| unit (hora repo) | — | no test suite; `lint` and `lint:docs-pairs` both clean |
| scenarios | `hof-e2e-test-specification` | **matched, and nothing was authored.** It derives scenarios from the API surface and states what must be true, not how to click it — but the harness it presumes cannot exist here (capability note), so a scenario list would have been a list nothing could ever run. Recorded as owed at the whole-version sweep rather than written to be unused |
| review | `hof-acceptance-review` | **matched, scoped mode.** Phases 0–3 and 5 ran; **phase 4 (live sweep) did not** — see below |
| version criteria | — | not in scope (gate) — §14's three criteria span all four features and only the sweep judges them |
| UX | — | not in scope (gate) |
| security | — | not in scope (gate); checkpoint 8 audited this feature's change set, scoped, and found six defects in already-green code |

### Phase 2 — the capability matrix

**Gate 2 requires every operation classified, and an exclusion without a reason to be treated as an
oversight.** Four operations, taken from the API surface rather than from the UI:

| Operation | Classification | Reason |
|---|---|---|
| `signIn` | **reachable** | the sign-in screen, on submit (§10.2) |
| `signedInStaffMember` | **reachable** | the sign-in screen, on opening (§10.2) |
| `renewAccessToken` | **reachable** | the client layer, on the opening path — no screen calls it, by §10.2's own statement |
| `signOut` | **deliberately excluded** | **§11.2 puts its control on the expense-entry screen**, which `#expense-entry` owns. §10.2's screen is explicitly "for a member of staff who is **not** signed in", so this feature has no screen a signed-in person ever sees. Q41 |

### Phase 3 — the static sweep

| Check | Exit | Result |
|---|---|---|
| `find-unused-operations` | 1 | **1 finding — `SignOutMutationGraphqlLauncher` built and imported by no screen.** Real, and it is the exclusion above. The mechanical check independently found what Q41 predicted |
| `find-unreachable-screens` | 0 | clean |
| `find-missing-screen-states` | 0 | clean — the four states built at checkpoint 15 |
| `find-leaked-vocabulary` | 0 | clean |
| `find-unsafe-html` | 0 | clean |
| `find-orphan-template-members` | **2** | **could not analyse — done by reading instead, see below** |
| `find-unguarded-slots` | 2 | not applicable — no component here declares a visibility prop. True: the one new component has no props |
| `find-emit-mismatches` | 2 | not applicable — no context declares its events in one place |

**Exit 2 is not a pass, and one of the three deserves its own note.**

`find-orphan-template-members` checks the one join nothing else does: a template reading a member its
context does not define. **A renamed member leaves the template compiling, lint clean and the unit
tests passing** — because the tests call the context directly and never render the template — and the
screen then throws at runtime.

It reported "no `.vue` file binds `context` to a Context class here". It does bind one. The script
looks for `const context = SomethingContext.create(`, and this page writes
`const signInPageContext = SignInPageContext.create({…})` — **because `javascript-style.md` forbids
chaining an instance method onto a factory call, so the instance must be named.** The rule-compliant
shape is the one the check cannot see, and its "not applicable" reads exactly like a clean result.

**Done by reading instead, and labelled Inferred:** all **14** distinct `context.<member>` references
in the template resolve against `SignInPageContext` or its base. None unresolved.

### Phase 4 — live sweep

**Did not run, and is recorded here rather than omitted**, because the skill says a review that
silently skips the live pass is the failure mode it exists to prevent. Root cause: Q24, above.

Everything a live sweep would have established is therefore **unestablished**: that a member of staff
can type an address and a password and arrive somewhere; that a refusal renders where a person looks;
that the pending state ends; that the redirect lands. Each is covered by a unit test against a real
capsule, and none has been seen.

### Findings

**None requiring work.** The one static finding is the deliberate exclusion above, classified with
its reason as Gate 2 requires.

Everything else this feature found was found *before* this run — by the checkpoints themselves — and
is already fixed and recorded. That is the shape worth noting: **the acceptance gate found nothing
because the gates before it did.** The list of what they found, and why no test could have caught
most of it, is in `.hora/tasks/1.0.0/sign-in.md`.

### The three lists owed at this gate

They live in `.hora/tasks/1.0.0/sign-in.md` rather than being copied here, so there is one authority
for each. Headline figures:

1. **What was found in code that was already committed, lint-clean and passing** — **24 items**, of
   which the great majority could not have been caught by any test, across **five distinct
   mechanisms**. Four are "the check and the failure never meet"; the fifth is **Q46, where the check
   met the failure and took its side** — a boilerplate test asserting the behaviour §6 forbids, so
   that a correct implementation is what turned it red.
2. **Where an equipped skill and an always-on rule disagreed** — **18 arbitrations**, in **four
   categories**: settled by Q10 (18, cost = detection only), skill versus boilerplate with **no
   adjudicator at all** (Q42, cost = a person), skill stricter than rule (additive, free), and — new
   at the frontend gate — **rule and skill together implying a third form neither states**.
3. **Where the decisions live, when control flow does not hold them** — **eight** places, each with
   the alternative it rejected. In six of the eight the declaration is also what makes a §10 criterion
   structural rather than agreed.
