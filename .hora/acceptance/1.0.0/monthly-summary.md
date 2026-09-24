# Acceptance — 1.0.0 — #monthly-summary

## Run 1
<!-- reach: scoped -->
<!-- scope: monthly-summary (rests on data-model, sign-in and expense-entry, all accepted) -->
<!-- live: no (skipped at the gate — and separately, it could not have run if requested; see the capability note) -->
<!-- reuse: none -->
<!-- not-accepted: none -->
<!-- version-criteria: not in scope (gate) -->
<!-- environment: not required at this reach. The MariaDB container was found DOWN and brought up; confirmed answering by query rather than by a start command's exit code. The application itself does not start here -->

### Verdict

**passed at static reach, with Gate 4 not run**

One finding, Minor — and it is **about an instrument rather than the product**. No product defect was
found. **That is a weaker statement than it looks**, and this record says so rather than letting the
absence read as strength: **the reach that found five defects on `#expense-entry` is the reach that
did not run here.**

### What ran

| Step | Delegate | Result |
|---|---|---|
| environment | — | not in scope at this reach; **checked anyway, and it was down** |
| unit (backend `__tests__`) | `hor-backend-testing`, `hoc-jest`, `hoc-test-execution` | **69 suites, 1693 tests** |
| unit (backend `_orders`) | same | **8 suites, 326 tests** |
| unit (frontend) | same | **50 suites, 959 tests** |
| lint (backend, frontend, app) | `hoc-workflows` | clean in all three |
| build (frontend) | — | succeeded |
| scenarios | `hof-e2e-test-specification` | **14 scenarios written, flow `MON`** — 9 capability, 3 provocation, 2 role. **None executed**, and **coverage could not be checked mechanically** (below) |
| review | `hof-acceptance-review` | **scoped, gates 0–3 pass, gate 4 not run; 1 finding** |
| version criteria | — | not in scope (gate) |
| UX | — | not in scope (gate) |
| security | — | not in scope (gate) — checkpoint 8 audited this feature's change set |

**Both acceptance skills had no digest — Q59's condition, on its fourth and final instance.** Both
were taken before the checkpoint ran, pinned `hora-skills-ort-furo 0.1.0`, each verified against the
installed package rather than assumed — one digester established it by **diffing the equipped copy
against the package's own**, which is stronger than reading a name.

**And a claim I wrote here first was false, caught before this record merged.** I wrote *"all 49
digest pins are now consistent"* on the strength of:

```
grep -h -o "hora-skills-ort-[a-z]* [0-9.]*" .hora/digests/*.md | sort | uniq -c
```

**That pattern matches from `hora-` onward, so it reads `<!-- @openreachtech/hora-skills-ort-furo
0.1.0 -->` and `<!-- hora-skills-ort-furo 0.1.0 -->` as the same string** — it could not see the
distinction I was asserting. Counted properly: **28 scoped, 21 unscoped.** Both name the package and
the version, so nothing is unpinned; the forms simply differ between older and newer digests.
**Recorded rather than churned at an acceptance gate.**

**That is Q58 inside this record**, which is the point worth keeping: the grep returned a plausible
tally rather than an error, and what caught it was a digester mentioning the two forms coexist —
**an outside observation, not my own check.**

```
backend    npm_config_script_shell=bash npm test -- --seeded --maxWorkers=3 tests/__tests__/   69 suites / 1693 tests
backend    npm_config_script_shell=bash npm test -- --seeded --maxWorkers=3 tests/_orders/      8 suites /  326 tests
backend    npm_config_script_shell=bash npm run lint                                            clean
frontend   npm run lint                                                                         clean
frontend   npm test --script-shell=bash                                                        50 suites /  959 tests
frontend   npm run build                                                                        succeeded
app        npm run lint / npm run lint:docs-pairs                                               clean / 8 pairs, 0 failures
```

**Every suite ran on `release/1.0.0` in its own repository**, not on a feature branch — release is
what the merge produced rather than what was proposed. **No step reused a recorded result.**

### Audit capability — what this run can and cannot prove

| | |
|---|---|
| mode | **scoped** — one feature, at its own gate |
| roles exercised live | **none** |
| browser driven | **no** |
| mechanical checks | 8 shipped scripts: **5 clean, 3 exited 2 — and one of those three is a false "not applicable"** |

### The live sweep — two facts, and they are not the same statement

**Skipped because of the kind of run.** Phase 4 is out of scope for a gate run **by default**. Nobody
requested it and this feature's acceptance was not deferred by a spec listing. So it was **not
attempted and did not fail**.

**Could not have run here if requested.** The backend does not start on this machine — **Q24** and
this tree's Windows binaries under WSL.

**Collapsing these would make a platform limitation look like a process choice, or the reverse.**

### The finding: a check that has never run, and reported itself inapplicable rather than blind

**Minor · Static.** `find-orphan-template-members.mjs` catches a template referencing
`context.<member>` that its class does not define — by its own docblock, the defect where *"the
template still compiles, lint still passes, and the unit tests still pass because they call the
context directly and never render"*.

It reports **`NOT APPLICABLE: no .vue file binds context to a Context class here.` That is false.**
Its pattern requires the **variable** to be named `context`; every screen here names it per screen
and exposes it under the key `context:`.

**So it has been blind to this project's entire idiom across three features.** `#expense-entry`'s
acceptance recorded three scripts exiting 2 as a limitation *"done by reading"* — **treating the exit
as a reach limit rather than asking whether the "not applicable" was true.**

**Redone by reading, adapted to the real idiom: 3 screens, 109 bound members, 0 orphans.** The gap is
real and nothing was hiding behind it.

**The wording is the more important half of the fix.** *"Not applicable"* reads as a reviewed
decision; *"no binding matched my pattern"* reads as what it is. **A check that was run and passed and
a check that was never run look identical in a record that only lists findings** — which is Q63's
sentence, and this is its second instance in two checkpoints.

**The other two exit-2 scripts were verified genuine**, not assumed: no component here declares a
visibility prop, and no context declares `static get EMIT_EVENT_NAME`.

### Two documents created that should have existed already

- **`ai/contexts/acceptance-context.md`** — Phase 1 says to create it rather than *"infer a core flow
  and then audit against the guess"*. **It did not exist**, so `#expense-entry`'s run audited without
  one; its Gate 1 passed on the other clause. Created here, filled from the codebase, **no `TBD`
  limited this review**.
- **`ai/specs/e2e/`** — **the directory did not exist**. `#expense-entry`'s 25 scenarios were
  *"derived in the run rather than read from a maintained document"* and **were never written down,
  so that derivation is gone**. This gate wrote its own flow's 14 and reserved `EXP` and `SIG` as
  namespaces. Writing those two is sweep-sized and not a gate's — but leaving the directory absent is
  the state that lost the first 25.

### The scenario skill's own coverage check does not exist here

`hof-e2e-test-specification` names `scripts/check-scenario-coverage.mjs` as what proves a flow's
coverage complete. **It exists nowhere** — not in this monorepo, not in any installed
`@openreachtech/hora-skills-ort-*` package. `find` over the whole tree returns nothing.

**So the 14 scenarios' coverage claim is hand-derived and cannot be verified by running anything**,
and the skill's own Gate 4 has no runnable check in this project. The derivation was made from the
API surface directly — which the skill sanctions when the capability inventory is absent, provided
the document **says it was derived that way**. The index does.

**Stated because "coverage complete" against a hand-scanned surface is the strongest false confidence
this kind of document can produce**, and a reader deserves to know which kind they have.

### Q62 is closed, said because a record could otherwise read otherwise

The stub-filter window opened at checkpoint 4 and **shut at checkpoint 6**, when the actual resolver
landed. The guard went green **on its own**, and git confirms the test file was never edited. It is
green in this run.

### Notes carried rather than fixed

- **Q66** — with two reads in flight the first answer clears the wait for both, so the month screen
  can briefly say a month is empty while it is still loading. Reported rather than patched: both
  cheap fixes are wrong and a correct one needs a request token nothing specifies.
- **Q64** — every `ph:` icon is fetched from `api.iconify.design` at runtime.
- **Q65** — the stub was never reachable, so checkpoint 16's *"this is where the stub is left behind"*
  was not claimed.
- **`find-unreachable-screens`' false positive on `/expenses` is now masked** rather than fixed — the
  new screen's back-link mentions the path, so the alias the script cannot see no longer matters.
  **If that link is ever removed, the false positive returns.**

### What this run could prove, stated plainly

That the product **builds, lints and passes 2978 unit tests across three repositories**; that **every
one of the 10 operations the API exposes is reached from a screen**; that **every screen is
navigable**, which it was not before checkpoint 15; that no template binds a member its context
lacks; that no markup sink or vocabulary leak exists; and that §12's criteria are backed where they
have a frontend surface, **with the two that do not named rather than claimed**.

**What it could not prove is everything a browser would have shown.**
