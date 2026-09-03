# 1.0.0

Order taken from spec §13 (Implementation plan, Milestone 1) and from each section's
`depends`. The two agree, so nothing was reordered.

## Features

1. [ ] #data-model         backend                                depends: none
2. [ ] #sign-in            backend, frontend-staff                depends: data-model
3. [ ] #expense-entry      backend, frontend-staff                depends: sign-in
4. [ ] #monthly-summary    backend, frontend-staff                depends: expense-entry

## Acceptance

- [ ] Sweep the whole version, once every feature above is done
      Version criteria: 3 (#version-acceptance-1-0-0), 0 resting on a not-accepted feature

## Not accepted

None. `Existing assets` declares `Baseline: not applicable` — nothing is inherited, so no
feature is listed rather than specified.

## Withdrawn

None. No section carries `kicked: yes`.

## Files more than one feature writes

Features are built one at a time, so these are a signal to re-read the real file before
writing, not a lock.

| The file | Which features | Why |
|---|---|---|
| the staff audience's SDL | #sign-in, #expense-entry, #monthly-summary | the backend row ships **one `.graphql` file per audience** (`.hora/tree/expense-note-backend.md`), while the equipped schema convention wants numbered per-domain files. Checkpoint 3 of #sign-in reconciles the two; whichever shape wins, the scalars / `Pagination` / `Sort` block is written once and the later two features append beside it |
| the staff audience's GraphQL type declarations under `types/` | #sign-in, #expense-entry, #monthly-summary | every operation's `Input` / `Result` is mirrored there, so all three append |
| `server/index.js`, plus the staff engine and context pair | #sign-in | `server/index.js` starts its servers **by hand — no scanning**, so the staff audience is added once. #sign-in is the first feature with operations, so it opens the server; the later two add resolver files only |

**Resolvers and frontend operation classes need no mark.** The backend registers a resolver by
**directory scanning** and Nuxt auto-registers a component, so neither has an aggregation file
for a feature to append to.

## Where the version's own criteria are checked

All three of §14's criteria span #sign-in, #expense-entry and #monthly-summary together, and
**no feature gate reads them** — the whole-version sweep is the only run that does. Three
criteria against four features is a low ratio, which is the shape the feature-at-a-time design
is meant to produce: each feature carries its own acceptance, and the sweep checks only what
genuinely crosses all of them.
