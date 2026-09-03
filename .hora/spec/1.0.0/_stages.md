# Stages — 1.0.0

0. [x] Assets and sources          new project, nothing implemented (`_assets.md`)
1. [x] Use cases and actors        one actor, three features, every block drafted and approved
2. [x] The horizon                 three lists, five seams named, build order matches `depends`
3. [x] Non-functional requirements four numbers, and the security rows stage 6 added
4. [x] Data, API and execution     three tables, seven operations, no background job
5. [x] Screens and interaction     three screens, every use case walked, both directions reachable
6. [x] Security                    every operation and every screen states its caller and its refusal
7. [x] Whole-document review       mechanical checks clean; every use case satisfiable

## Decided in conversation

- one actor only, a member of staff managing their own expenses
- accounts are issued by an operator outside the product — no sign-up, no admin screen, and
  therefore no second actor in the table
- `Question language: English`

## Proposals not taken

None. Every proposal this run made was accepted.

## Assumptions recorded

None. Nothing was assumed in the absence of an answer.

## Stage 7's findings

- **Reading one's own entries belongs to `expense-entry`, not to `monthly-summary`.** Placed the
  other way round, `expense-entry`'s use cases would name a feature built after it, which is a
  `forward-reference` stop at `/hora-plan`. `monthly-summary` adds the month and the total on top
- **The document keeps one file.** Three features, seven operations and three screens fit in it,
  and splitting is what a document does once it grows
- No forward reference, no duplicate `id`, no operation without a kind or a caller, and every
  `Fine to leave for later` entry corresponds to an `Out of scope for now` entry
