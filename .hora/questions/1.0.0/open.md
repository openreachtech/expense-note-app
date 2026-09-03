# Open questions — 1.0.0

Append-only. Answered by editing `specs/`, never by editing an entry here.

## Q1. Nothing in the data model holds a session
<!-- spec: data-model, sign-in -->
<!-- blocking: yes -->
<!-- category: unmet-usecase -->

§9 declares three tables — `staff_members`, `expense_categories`, `expenses` — and none of
them holds a session. §10's `SignInResult(staffMemberId)` carries nothing the client can
authenticate with on the next request.

Two of §10's own acceptance criteria cannot be met by the design as written:

- "a session survives a page reload" — nothing is persisted for a reload to find
- "stops working the moment its holder signs out" — a stateless signed cookie could survive
  the reload, but could not be revoked on sign-out without something to revoke against

The reading was put up as a check and confirmed: it is a hole in the spec, not in the reading.

**Which session mechanism to use is a design decision, so it is not settled here.** It reaches
both the data model and the operation list, and it changes the pinned contract:

- a server-side session row keyed by an opaque cookie needs one table, and `signIn` /
  `signOut` / `signedInStaffMember` stay as the three operations §10 declares
- the two-token flow the equipped cookie-authentication convention describes needs an access
  token on `SignInResult`, a refresh-token cookie, its own token tables, and a **fourth
  operation** to renew — which would then need a kind and a caller stated in §10.1 like every
  other operation

**Routed to `/hora-spec`, stage 4 (data, API and execution).** `/hora-plan` does not write a
design decision into `specs/`.

- [ ] unresolved

## Q2. `amount` and `totalAmount` are typed `Int!` in the contract
<!-- spec: expense-entry-operations, monthly-summary-operations -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

The equipped GraphQL schema convention types money and decimal fields `String!`, emitted via
`BigNumber#toFixed(2)`. That rule exists to protect decimals, and this product has none: §9.3
declares `amount int`, and §4 permanently excludes a minor unit and any second currency.

- [x] resolved
      Decided in conversation: `Int!`. `String!` would send `"1200.00"` for a currency with no
      minor unit, and every screen would strip the decimals back off. The departure from the
      house rule is deliberate and recorded here so it is not read as an oversight.

## Q3. `spentOn` is typed `String!` as an ISO `YYYY-MM-DD` date
<!-- spec: expense-entry-operations, monthly-summary-operations -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

§9.3 declares `spent_on date` — "the day the money was paid, not the day it was recorded" —
so there is no time of day to carry. The convention's declared `DateTime` scalar would carry
a midnight nobody set.

- [x] resolved
      Decided in conversation: `String!`, ISO `YYYY-MM-DD`. §12's month-boundary criterion
      turns on which day an expense falls on, and a timezone shift applied to an invented
      midnight can move that day across a month boundary. A dedicated `Date` scalar was
      offered and not taken — it adds a scalar and a serializer both repositories must agree
      on, for a field that is already unambiguous as text.

## Q4. The expense row exposes `status` in 1.0.0
<!-- spec: expense-entry-operations -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

§4's approval seam states that an expense "carries a status from the first migration, holding
one value now, and every read goes through it". Whether the *contract* exposes that field to
the frontend in this version is a separate decision from whether the column exists.

- [x] resolved
      Decided in conversation: expose it now. 1.1.0's approval feature then adds no field to
      the row — it only adds values the field can hold. The frontend ignores the single value
      `recorded` for now.

## Q5. The rest of the contract was derived from the parenthesized contents
<!-- spec: sign-in-operations, expense-entry-operations, monthly-summary-operations -->
<!-- blocking: no -->
<!-- category: undefined-detail -->

§10.1, §11.1 and §12.1 name every input and result with their contents in parentheses
(`ExpensesInput(pagination)`), which is a shape to derive rather than an unknown one to
invent. Recorded here is what was derived and how.

Derived from the equipped GraphQL schema convention, and from the SDL already in the backend
row (`server/graphql/schemas/<audience>.graphql`, one file per audience, `healthCheck` only):

- `<Operation>Input` / `<Operation>Result` one-to-one per operation, never shared between two
- non-null `!` everywhere except a genuinely optional field — `memo` alone, per §9.3
- ids typed `Int!`, following the convention's own entity example, though the columns are
  `bigint`
- `pagination` is the convention's `PaginationInput!` / `Pagination!` pair — `limit`, `offset`
  and `totalRecords` non-null, `sort` nullable
- the expense row is one reused type, `Expense`, named after §6's own term. `ExpensesResult`
  and `MonthlyExpensesResult` both carry it, which is what keeps §12's "the total equals the
  sum of the entries shown" checkable against one shape
- `expenses` sorts most recent first, from §11.2's "their own entries, most recent first"
- `timestamps` are the convention's `DateTime!` scalar
- **the three operations the spec writes as `SomethingInput()` take no argument at all.** The
  convention gives an operation with no input no argument, and it prohibits a
  single-character key — so no empty input type is declared and no placeholder field is
  invented to make one legal
- **the `Sort` / `SortInput` pair is `key` / `direction`**, taken from the framework's own
  `RequestSort` in the backend row rather than invented. No operation in 1.0.0 lets the
  caller choose a sort, so the pair is declared for the convention's pagination shape and
  left unused

**Not derived, and deliberately left out: everything the session mechanism decides** (Q1).
The contract file marks that surface open rather than guessing at it.

- [x] resolved
      Recorded. Nothing above was invented; each line traces to the spec or to the convention.
