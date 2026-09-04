# Stages — 1.0.0

0. [x] Assets and sources          new project, nothing implemented (`_assets.md`)
1. [x] Use cases and actors        one actor, three features, every block drafted and approved
2. [x] The horizon                 three lists, five seams named, build order matches `depends`
3. [x] Non-functional requirements four numbers, and the security rows stage 6 added
4. [x] Data, API and execution     re-entered twice: nine tables, ten operations, no background job
5. [x] Screens and interaction     three screens, every use case walked, both directions reachable
6. [x] Security                    re-entered: renewAccessToken's caller stated, section 7's
                                   Authentication row rewritten, the reuse refusal and a
                                   rate-limiting row added
7. [x] Whole-document review       ran four times. Findings 1-8, every one fixed by the stage
                                   that owned it. Mechanical pass clean on the last run

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

## Stage 7 — review

Ran four times, because each fix could contradict something a later stage had written.

| # | Finding | Sent back to | Result |
|---|---|---|---|
| 1 | §4's reset seam named the member of staff's own row; the hash had moved | 2 | fixed |
| 2 | §10's criterion said "neither operation" of four | 7 | fixed (wording) |
| 3 | rate limiting was built this time and no scope list said so | 2 | fixed |
| 4 | §6 defined none of session / access token / refresh token | 1 | fixed |
| 5 | §6 said the access token goes on "every request"; three operations carry none | 1 | fixed |
| 6 | §7's Security level described a one-token design | 3 | fixed |
| 7 | §4's reset seam said "that one row"; §9.9 makes it two | 2 | fixed |
| 8 | §9's oldest criterion forbade a shared address "in no circumstance"; §9.8 permits it | 7 | fixed (wording) |

**The pattern across 1, 5, 6, 7 and 8, and worth carrying into later versions:** every one was a
sentence that was true when written and was falsified by a change somewhere else, and nothing
pointed back at it. Four of the five phrased something absolutely — "every request", "that one
row", "in no circumstance", "a session cookie". §7's Authentication row was corrected three
separate times as a second credential appeared, and now closes with "an operation added later
states which of the two it uses, or it is not finished" so the next author is forced to decide
rather than inherit a blanket.

**What no amount of re-reading the document caught:** findings 5 and 6 came from a question
about whether two rows were correct against the equipped convention, and the nine-table
credential cluster came from reading two running implementations. Stage 7 checks the spec
against itself; it cannot check it against the conventions the checkpoints build from.
