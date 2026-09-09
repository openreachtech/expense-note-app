# Acceptance — 1.0.0 — #data-model

## Run 1
<!-- reach: scoped -->
<!-- scope: data-model -->
<!-- live: no (skipped at the gate) -->
<!-- reuse: none -->
<!-- not-accepted: none -->
<!-- version-criteria: not in scope (gate) -->
<!-- environment: not required — a gate run whose live review is skipped neither needs the stack nor brings it up. Docker Desktop was in fact down for this run -->

### Verdict

passed over 1 of 4 features; 0 not accepted

### What ran

| Step | Delegate | Result |
|---|---|---|
| environment | — | not required at this reach; not brought up |
| unit (backend) | `hor-backend-testing`, `hoc-jest`, `hoc-test-execution` | 199 passed in 20 suites, then 43 passed in 3 `_orders` suites. Run through the repository's own `test.sh`, exit 0 |
| unit (frontend-staff) | `hoc-jest`, `hoc-test-execution` | 27 passed in 8 suites, exit 0 |
| scenarios | `hof-e2e-test-specification` | **matched, nothing in scope** — it derives scenarios from the API surface, and §9 exposes none |
| review | `hof-acceptance-review` | **matched, nothing in scope** — every phase it runs asks whether an operation is reachable from the UI; this feature has no operation and no screen |
| version criteria | — | not in scope (gate) |
| UX | — | not in scope (gate) |
| security | — | not in scope (gate); checkpoint 8 audited this feature's change set twice, scoped |

### Findings

None.

### What this verdict does and does not claim

**It claims** that the nine tables, their models, their type declarations and the category
seeder are in place and that every suite in both repositories passes with them — 269 tests
across 31 suites, none reused, all executed in this run.

**It does not claim that a review examined this feature**, and the two rows above say so rather
than reading as a clean review. Both delegates were matched and both are the correct ones; §9
declares no operation and no screen, so neither had anything to look at. A row reading "passed"
there would have claimed a review found nothing wrong when no review could run at all.

**It does not rest on a driven browser.** A feature-gate run skips the live review by default,
so nothing here was verified through a UI. That is the gate's own reach, not a shortfall —
and it is why `live:` reads `no`.

### Which features this run did not drive, and when each last was

The version carries four features. This run drove one.

| Feature | Last driven live | Why not here |
|---|---|---|
| `#data-model` | **never** — this run's live review was skipped at the gate | — |
| `#sign-in` | **never** | not started; every checkpoint `[ ]` |
| `#expense-entry` | **never** | not started; every checkpoint `[ ]` |
| `#monthly-summary` | **never** | not started; every checkpoint `[ ]` |

**No feature in this version has ever been driven live**, read off `.hora/acceptance/` where no
block anywhere has `live: yes`. That is expected at the first feature gate of the first
version, and it is written down because "never" is the answer worth reading: the whole-version
sweep is the run that has to change it, and until then nothing about this product has been
verified through a browser.
