# Stages — 1.0.0

0. [x] Assets and sources          new project, nothing implemented (`_assets.md`)
1. [x] Use cases and actors        one actor, three features, every block drafted and approved
2. [x] The horizon                 three lists, five seams named, build order matches `depends`
3. [x] Non-functional requirements four numbers, and the security rows stage 6 added
4. [x] Data, API and execution     re-entered: seven tables, eight operations, no background job
5. [x] Screens and interaction     three screens, every use case walked, both directions reachable
6. [ ] Security                    re-entered: renewAccessToken's caller, section 7's
                                   Authentication row, and the reuse-after-sign-out criterion
7. [ ] Whole-document review       must run again: section 9 and 10.1 changed

## Decided in conversation

- one actor only, a member of staff managing their own expenses
- accounts are issued by an operator outside the product — no sign-up, no admin screen, and
  therefore no second actor in the table
- `Question language: English`
- **the session mechanism** (stage 4, re-entered after Q1). The two-token cookie flow the row
  already ships the orchestration for — a short-lived access token plus a persisted refresh
  token in an httpOnly cookie. Chosen because "stops working the moment its holder signs out"
  needs revocation, which rules out anything stateless, and because the equipped convention is
  what the checkpoints read anyway
- **the credential is a five-table cluster**, not columns on the entity. Following the
  convention fully rather than adding the token tables alone: the alternative left
  `verifiesPassword` with no table to sit on and made the sign-in path bespoke auth code
- **fifteen minutes for the access token, as a decision rather than a reading.** A grep found
  no such constant anywhere in the row — `.env.development`'s comment asserting one is
  upstream's and wrong — so the figure comes from the equipped convention and is written as a
  choice somebody made. Left unstated, checkpoint 3 would have picked one
- **`sequelize/models/` ships empty**, so all seven of section 9's tables are checkpoint 3's to
  write. The row ships the session orchestration and the cookie handling, and none of the
  storage

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
