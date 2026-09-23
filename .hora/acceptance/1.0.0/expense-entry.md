# Acceptance — 1.0.0 — #expense-entry

## Run 1
<!-- reach: scoped -->
<!-- scope: expense-entry (rests on data-model and sign-in, both accepted) -->
<!-- live: no (skipped at the gate — and separately, it could not have run if requested; see the capability note) -->
<!-- reuse: none -->
<!-- not-accepted: none -->
<!-- version-criteria: not in scope (gate) -->
<!-- environment: not required at this reach. The MariaDB container was brought up at checkpoint 17 and is still running with the live database seeded; the application itself does not start here -->

### Verdict

**passed, after five defects were fixed and three records corrected**

The audit did not pass this feature as it stood. It produced five findings — two Major, three Minor
— and found three overstatements in this feature's own checkpoint records. Every finding is fixed
with a test that fails without it; every overstatement is corrected in place. **This block records
the run that found them, not a clean arrival.**

### What ran

| Step | Delegate | Result |
|---|---|---|
| environment | — | not in scope at this reach |
| unit (backend `__tests__`) | `hor-backend-testing`, `hoc-jest`, `hoc-test-execution` | **64 suites, 1510 tests** |
| unit (backend `_orders`) | same | **8 suites, 314 tests** |
| unit (frontend) | same | **43 suites, 700 tests** |
| lint (backend, frontend) | `hoc-workflows` | clean in both |
| build (frontend) | — | succeeded |
| scenarios | `hof-e2e-test-specification` | **25 scenarios, flow `EXP`; coverage complete against all five operations; none executed** |
| review | `hof-acceptance-review` | **scoped mode, gates 0–3; 5 findings** |
| version criteria | — | not in scope (gate) |
| UX | — | not in scope (gate) |
| security | — | not in scope (gate) — checkpoint 8 audited this feature's change set |

**The commands, because a figure without its invocation is not a claim anybody can check.** This is
the standing practice adopted after the backend gate, where a green suite turned out to have been
measured against an invocation CI does not use.

```
backend    npm test -- --seeded --maxWorkers=3 tests/__tests__/     64 suites / 1510 tests
backend    npm test -- --seeded --maxWorkers=3 tests/_orders/        8 suites /  314 tests
backend    npm run lint                                              clean
frontend   npm run lint                                              clean
frontend   npm test --script-shell=bash                             43 suites /  700 tests
frontend   npm run build                                             succeeded
```

The two backend commands are **exactly what CI runs**, including `--maxWorkers=3` — the parallel
form that was failing at the backend gate and that no local run had used before that fix. On Windows
both need `npm_config_script_shell=bash`; CI runs the identical script strings on Linux. The frontend
form differs from bare `npm test` only in the shell, for the same reason.

**The backend suites ran on `release/1.0.0`**, not on `feature/expense-entry`. The two are identical
in content — `git diff --stat` between them is empty — but release is what the merge produced rather
than what was proposed, so it is the truer tree to accept. The frontend ran on `feature/expense-entry`,
which has no gate merge yet.

**No step reused a recorded result.** Every figure above was executed in this run.

### Audit capability — what this run can and cannot prove

| | |
|---|---|
| mode | **scoped** — one feature, at its own gate |
| roles exercised live | **none.** The member of staff and the no-session caller were reasoned about from code and from the suites |
| browser driven | **no** |
| checks that could not run | three static scripts exited 2 and were done by reading; fixture fidelity in full needs live recordings |

### The live sweep — two facts, and they are not the same statement

**Skipped because of the kind of run.** `hof-acceptance-review`'s phase 4 is out of scope for a gate
run **by default**. Nobody requested it, and this feature's acceptance was not deferred by a spec
listing. So it was **not attempted and did not fail**.

**Could not have run here if requested.** Independently: the backend does not start on this machine,
for two measured reasons — **Q24** (renchan's loader hands a raw `D:\…` path to `import()`; filed
upstream in four repositories; does **not** reproduce on WSL) and **this tree's own `node_modules`**
holding Windows binaries, which kills it under WSL at `sqlite3 … invalid ELF header`.

**Collapsing these would make a platform limitation look like a process choice, or the reverse.**
Only the second is about this machine, and neither is a property of `#expense-entry`.

### §11's seven acceptance criteria — all met, none taken on a record's word

Each was tied to the code and the assertion that establishes it, not accepted because a checkpoint
said so.

| # | criterion | how it is held |
|---|---|---|
| 1 | no/zero/negative amount refused, **nothing recorded** | the validator throws before a transaction opens, so "nothing recorded" is structural; seven `_orders` blocks assert the refusal **and** re-read through the product's own path |
| 2 | dated after today refused | lexical ISO comparison against `context.now` rendered in `CALENDAR.TIMEZONE`, plus the screen declining the send — asserted `.not.toHaveBeenCalled()` |
| 3 | memo optional, reads back empty | both halves in one test, re-read through `expenses` rather than by inspecting the row |
| 4 | correcting changes in place, count unchanged | `entity.update()` on the row found by `(id, StaffMemberId)`; **no delete-and-reinsert path exists** |
| 5 | removed entry gone; a second removal changes nothing | asserted, with the already-removed case answered by the **same code** as the other two |
| 6 | somebody else's expense → not found, saying nothing | one read narrowing by `(id, StaffMemberId)`, so **there is no second code to leak**; asserted as an identity of codes, and the two sentences are byte-identical |
| 7 | refused without a session, before reading anything | `schemasToSkipFiltering` pinned whole; exercised through the **built schema** with a real session-less context, ten blocks, five operations |

### Findings

All five fixed in `0130f30`, each with a test that fails without it, each mutation-checked.

1. **Major — a removal on the last page hid every remaining entry.** The re-read reused the offset
   already on screen, which after the removal was past the end: zero rows against a non-zero total,
   which the table cannot tell from "there are none". **21 entries, remove the one on page 2, and the
   screen says "No entries yet" over 20 that are still there** — and at exactly 20 the paginator
   vanishes too, so there is no way back without reloading. Sent back to **checkpoint 16**.

   **The existing test asserted the request offset and never what the screen then showed, so it
   passed under both the broken and the fixed code.** That is a test arguing the behaviour is
   correct, on behalf of a defect — Q46's shape, in code this project shipped. Its expectations are
   now the offsets those fixtures' own totals actually have a page at, which **fails** under the old
   behaviour.

2. **Major — a failed category read left the form permanently unusable.** The message shared the slot
   every submit clears, so pressing *Record* erased the explanation and replaced it with *"Choose a
   category."* — an instruction nobody could follow — and the only retry on screen read the entries.
   Sent back to **checkpoint 15**, correctly: **the form's own source failing was a fifth condition
   the design never had**, so the gap was in the design rather than the wiring.

3. **Minor — a second Remove during an in-flight removal** reported not-found for an entry that had
   in fact been removed.

4. **Minor — nothing showed a removal was in flight**, because the busy flag was read only into a
   dialog that closes before the request goes out.

5. **Minor — fixtures named a category the API cannot send.** `'Travel'` has never existed; the
   master data is `transport`, `meals`, `supplies`, `other`. **Found in one file, fixed in three** —
   the same defect was in two capsule suites the finding did not name.

### Three overstatements in this feature's own records, and they are the more interesting finding

Corrected in place, not quietly.

- **"Eight furo-vue components serve the whole screen."** Eleven do. Eight was the count of component
  **skills matched**; three of those route to two components each.
- **"Every message goes through `extractResolvedErrorMessage()`."** One does not — the client-side
  future-date refusal reads the hash directly, deliberately, so a member of staff cannot tell which
  side noticed. **The code is right and the sentence was wrong.**
- **"One screen, four states."** They are the **entries'** four states. Finding 2 is the fifth
  condition that had none.

**All three are the same shape: a claim whose scope nobody stated, standing in for a wider one.**
That is Q58, checkpoint 9's memo near-miss, checkpoint 15's four-link table and the serial-figures
correction — one mechanism, found in code, in tests, in instruments and in prose.

### Notes carried rather than fixed

- `expenseCategories` is the one operation of this feature with no in-resolver session guard.
  Criterion 7 still holds — the engine's filter refuses before `resolve()` is entered, which is what
  the criterion names — but the other four carry a guard as defence in depth and this one does not.
  What would leak is the four-row public category master.
- The screen selects `status` and drops it. The approval seam is honoured on the contract and in the
  document; nothing on the frontend would notice a status other than `recorded` when 1.1.0 adds
  approval.
- `hof-e2e-test-specification` and `hof-acceptance-review` **have no digests**, and two of their
  Phase 1 prerequisites do not exist in the frontend repository. The scenario list above was derived
  in the run rather than read from a maintained document.
- `find-unreachable-screens` reports `/expenses` as address-bar-only. **False positive** — the script
  cannot see a Nuxt route alias. Recorded so the next run does not re-file it.
