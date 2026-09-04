# Expense note design document

## 1. Document information

| Item | Content |
|---|---|
| Product version | 1.0.0 |
| Document revision | 1 |
| Author | Open Reach Tech Inc. |
| Question language | English |

**Project name: `expense-note`**


## 2. Repository layout

| Repository | Origin | Role |
|---|---|---|
| `expense-note-backend` | renchan | the API, and it holds the DB |
| `expense-note-frontend-staff` | furo | the screens a member of staff uses |

### 2.1 Servers

| Server | protocol | consumer |
|---|---|---|
| `staff-graphql` | GraphQL | `frontend-staff` |

One server, one consumer. There is no REST server: nothing here is a file transfer, a redirect or a third party that cannot speak GraphQL, so the default stands.


## 3. Actors and roles

| Actor | Identified by | Roughly how many | Inside / outside |
|---|---|---|---|
| member of staff | an email address and password issued by an operator; there is no sign-up | 20, 50 foreseen | inside |

There is no second actor. Accounts are created by whoever operates the deployment, outside the product, so the operator never signs in and no screen is built for them.


## 4. Implementation scope

### Built this time (1.0.0)

- signing in and signing out
- recording an expense, correcting it, removing it, and reading back one's own entries
- a chosen month's entries with their total

### Out of scope for now (to be built later)

- approval by a manager → planned for 1.1.0. **Seam:** an expense carries a status from the first migration, holding one value now, and every read goes through it rather than assuming a recorded expense is a final one
- exporting a month for a claim form (CSV) → planned for 1.2.0. **Seam:** a month's entries and its total are produced by one operation, which an export calls rather than re-deriving
- a photograph of the receipt on an expense → needs object storage, which this version does not declare. **Seam:** nothing in the model assumes an expense has exactly one representation; an attachment is a child table added later, not a column added to `expenses`
- categories an operator can edit → **seam:** a category is already a row with an id, seeded, never an enum written into the code
- signing up, and resetting a password by mail → needs a mail sender, which this version does not declare. **Seam:** the password is already a one-way hash on the member of staff's own row, so a reset changes that row and touches nothing else

### Permanently out of scope

- more than one currency. Every amount is an integer number of yen, and no exchange rate, minor unit or currency column is designed for
- more than one company in one deployment. There is no tenant, and no row carries one
- a phone app, or any public API. There is exactly one consumer and it speaks GraphQL


## 5. Existing assets

Current implementation: none (new)
Treatment: build new — there is nothing to port and nothing to match
Authority: not applicable — no code exists for the spec to disagree with
Baseline: not applicable — nothing is inherited


## 6. Terminology and domain concepts

| Term | Description |
|---|---|
| expense | one payment a member of staff made and records: the date it was paid, the amount, a category and an optional memo |
| category | the fixed set an expense is filed under — transport, meals, supplies, other |
| month | the calendar month an expense's date falls in. The unit a list and a total are taken over |
| own entry | an expense recorded by the member of staff who is signed in. Nobody sees anybody else's |


## 7. Non-functional requirements

| Item | Requirement |
|---|---|
| Users at launch | 20 members of staff |
| Users foreseen | 50, within two years |
| The heaviest single operation | one member of staff's month — at most a few hundred rows read and summed. At the foreseen size it stays a single indexed read, and no total is stored |
| Response time | a month's entries and its total return in under one second at the foreseen size |
| Availability | office hours, best effort. No standby, and no uptime commitment |
| Data retention | an expense is kept for 7 years, for bookkeeping. Nothing expires automatically. An entry its owner removes is deleted outright rather than archived — what is retained is what remains |
| Security level | an ordinary internal business system. TLS in front, a session cookie, a one-way password hash |
| Authentication | every operation requires a session, and is refused before it reads anything without one — with two exceptions. `signIn` is how a session begins. `renewAccessToken` authenticates by the refresh-token cookie alone and is how a session continues. The access token is sent on a request header and lives fifteen minutes; the refresh token is an httpOnly cookie and never reaches a response body |
| Authorization | every operation that touches an expense is scoped to the signed-in member of staff. Another member of staff's expense is answered as not found, never as forbidden |
| Personal data | a member of staff's name and email address, and a memo, which may name a client. None of the three is ever written to a log line, and no operation returns a password hash |
| Rate limiting | `signIn`, 10 **failed** attempts per 15 minutes **per email address** — keyed on the address rather than the caller's IP, because the staff sit behind one office address and an IP-keyed limit would let one person's wrong password lock out everybody. A successful sign-in counts for nothing. `renewAccessToken`, 60 per hour per refresh-token series — roughly fifteen times normal use, which is one renewal per fifteen-minute access token. These are the two operations reachable without a session, and the only two limited at 1.0.0 |


## 8. Manual verification

| Middleware | Version | profile | Purpose |
|---|---|---|---|
| MariaDB | 10.5.12 | (default) | the primary data store |

**Redis is not declared, because this version runs no background job.** Every write finishes inside its own request, and nothing here leaves the process.


## 9. Data model
<!-- id: data-model -->
<!-- target: backend -->
<!-- depends: none -->

### 9.1 staff_members

| Column | Type | Constraint | Description |
|---|---|---|---|
| id | bigint | primary key | |
| name | varchar(191) | not null | what the signed-in member of staff is shown as |
| created_at / updated_at | datetime(3) | not null | |

The sign-in address and the password digest are **not** here — they sit in their own tables (§9.4, §9.5), one row each per member of staff. Kept apart, a query that reads somebody's name cannot carry a credential in its result set by accident. Serves §10's use cases, and every operation that names an owner.

### 9.2 expense_categories

| Column | Type | Constraint | Description |
|---|---|---|---|
| id | int | primary key | |
| name | varchar(191) | not null, unique | transport, meals, supplies, other |
| display_order | int | not null | the order the four are offered in |
| created_at / updated_at | datetime(3) | not null | |

Seeded, not editable in this version. A category is a row with an id from the first migration, so making it editable later adds a screen and changes no other table.

### 9.3 expenses

| Column | Type | Constraint | Description |
|---|---|---|---|
| id | bigint | primary key | |
| staff_member_id | bigint | not null, indexed | the owner. Every read is scoped by it |
| expense_category_id | int | not null, indexed | |
| spent_on | date | not null | the day the money was paid, not the day it was recorded |
| amount | int | not null | yen. An integer, because the yen has no minor unit |
| memo | varchar(191) | null | optional |
| status | varchar(32) | not null | `recorded` for every row this version writes. The seam approval is built on |
| created_at / updated_at | datetime(3) | not null | |

Indexed on `(staff_member_id, spent_on)` — the one read that matters is one member of staff's month, and it is the heaviest operation this version has.

### 9.4 staff_member_secrets

| Column | Type | Constraint | Description |
|---|---|---|---|
| id | bigint | primary key | |
| staff_member_id | bigint | not null, unique index | 1:1 with §9.1 |
| email | varchar(191) | not null, unique | the address the account was issued against. The sign-in identifier, and the only thing `signIn` looks a member of staff up by |
| created_at / updated_at | datetime(3) | not null | |

Serves both of §10's use cases.

### 9.5 staff_member_password_hashes

| Column | Type | Constraint | Description |
|---|---|---|---|
| id | bigint | primary key | |
| staff_member_id | bigint | not null, unique index | 1:1 with §9.1 |
| password_hash | varchar(191) | not null | a one-way hash. Never returned by any operation, never written to a log line (§7) |
| created_at / updated_at | datetime(3) | not null | |

A candidate password is verified against this row and nowhere else. Serves §10's first use case, and the password reset foreseen for later (§4) — which changes this row alone.

### 9.6 staff_member_access_tokens

| Column | Type | Constraint | Description |
|---|---|---|---|
| id | bigint | primary key | |
| staff_member_id | bigint | not null, indexed | 1:N — a member of staff may hold several |
| access_token | varchar(191) | not null, unique | the short-lived credential, sent on a request header. **Stored as itself, not as a digest** — see below |
| session_key | varchar(191) | not null, indexed | the series this token belongs to. Signing out revokes by it |
| generated_at | datetime(3) | not null | |
| expired_at | datetime(3) | not null | fifteen minutes after `generated_at`. **This project fixes the access token's life at fifteen minutes** — the figure the equipped convention is built around. Nothing in the row's code sets it, so leaving it unstated would have checkpoint 3 pick one |
| created_at / updated_at | datetime(3) | not null | |

**This table holds its token in the clear and §9.7 holds only a digest. The asymmetry is the design, not an oversight.** An access token is readable here because it lives fifteen minutes: the exposure a leaked dump buys is one lifetime, after which every row in it is useless. A refresh token lives fourteen days, so the same dump would be a set of working sessions — and §9.7 therefore stores nothing that can be presented. The lifetime is what earns the difference.

An expired access token is **deleted**, not flagged — there is no `revoked_at` here, because the lifetime is the revocation. Serves both of §10's use cases.

### 9.7 staff_member_refresh_tokens

| Column | Type | Constraint | Description |
|---|---|---|---|
| id | bigint | primary key | |
| staff_member_id | bigint | not null, indexed | 1:N |
| token_hash | varchar(191) | not null, unique | **the digest, never the token.** A dump of this table is not a set of usable sessions |
| session_key | varchar(191) | not null, indexed | the series a rotation keeps. Signing out revokes every row sharing it |
| used_at | datetime(3) | null | set when the token is spent on a rotation. Presented again after this, it is a reuse |
| revoked_at | datetime(3) | null | set on sign-out, and on a detected reuse |
| generated_at | datetime(3) | not null | |
| expired_at | datetime(3) | not null | `AUTH_REFRESH_TOKEN_TTL_DAYS` after `generated_at`, falling back to a 14-day default in code when unset. **A deployment's own value, not a fixed 14** |
| created_at / updated_at | datetime(3) | not null | |

This is the table that makes §10's second use case possible: the token is persisted, so signing out has something to revoke, and a reload has something to present. It reaches the browser as an httpOnly cookie and is never returned in a response body.

### Acceptance criteria
<!-- acceptance -->

- an expense row cannot be written without an owner, a category, a date and an amount
- two members of staff can hold the same email address in no circumstance
- the four categories exist after the seeder runs, in their display order
- an expense always names a member of staff who exists, and a category that exists
- a member of staff has exactly one sign-in address and exactly one password digest, and no second row of either can be written against them
- no refresh token is stored in a form that could be presented as one — only its digest
- two refresh tokens cannot share a digest
- a refresh token row records, separately, that it was spent and that it was revoked


## 10. Sign in
<!-- id: sign-in -->
<!-- target: backend, frontend-staff -->
<!-- depends: data-model -->

### 10.1 Operations
<!-- id: sign-in-operations -->
<!-- target: backend -->

| schema | input | result | kind | caller |
|---|---|---|---|---|
| `signIn` | `SignInInput(email, password)` | `SignInResult(staffMemberId, accessToken)` | mutation | anyone. The one operation reachable without a session |
| `signOut` | `SignOutInput()` | `SignOutResult(signedOut)` | mutation | the signed-in member of staff |
| `signedInStaffMember` | `SignedInStaffMemberInput()` | `SignedInStaffMemberResult(staffMemberId, name, email)` | query | the signed-in member of staff, about themselves only |
| `renewAccessToken` | `RenewAccessTokenInput()` | `RenewAccessTokenResult(accessToken)` | mutation | anyone presenting a refresh-token cookie. **Reachable without a session, by design** — it is what re-establishes one |

`signedInStaffMember` exists because a session is a cookie: after a reload the screens have to ask whether they still have one, and who is holding it.

`signIn` returns the short-lived access token in its result and puts the refresh token in an httpOnly cookie; the refresh token never appears in a response body. `renewAccessToken` reads that cookie, spends the presented refresh token, issues the next pair in the same series, and returns the fresh access token — so a member of staff whose access token has expired stays signed in without signing in again. Presenting a refresh token that was already spent is a reuse, and the whole series is revoked.

### 10.2 Screen
<!-- id: sign-in-screen -->
<!-- target: frontend-staff -->

For: a member of staff who is not signed in. It is the one screen reachable without a session, and every other screen sends somebody here when theirs has gone.

| Calls | Kind | When |
|---|---|---|
| `signIn` | mutation | on submitting the address and password |
| `signedInStaffMember` | query | on opening, to send an already-signed-in person on rather than asking twice |

### Use cases
<!-- usecases -->

- a member of staff opens the app on a Monday morning, signs in with the address and password they were issued, and is signed in — every screen after that knows who they are
- a member of staff who has finished on a shared machine signs out, and the next person to open the app is asked to sign in rather than landing in somebody else's account

### Acceptance criteria
<!-- acceptance -->

- an address with no account and a correct address with the wrong password are refused identically, and neither refusal says which of the two it was
- a session survives a page reload, and stops working the moment its holder signs out
- neither operation this feature adds returns a password or a password hash, and neither writes one into a log line
- `signedInStaffMember` called without a session is refused, and returns nobody
- `renewAccessToken` presented with a refresh-token cookie that is expired, revoked, or already spent is refused identically in all three cases, and the refusal reveals which of the three it was in none of them
- a refresh token presented after it was already spent revokes every token in its series — so a stolen cookie stops working, and so does the session it was stolen from
- `renewAccessToken` called with no refresh-token cookie at all is refused, and returns nobody
- an eleventh failed sign-in for the same email address inside fifteen minutes is refused for being too frequent, while a first attempt for a different address in the same window is not; ten successful sign-ins in that window are refused nothing


## 11. Expense entry
<!-- id: expense-entry -->
<!-- target: backend, frontend-staff -->
<!-- depends: sign-in -->

### 11.1 Operations
<!-- id: expense-entry-operations -->
<!-- target: backend -->

| schema | input | result | kind | caller |
|---|---|---|---|---|
| `expenses` | `ExpensesInput(pagination)` | `ExpensesResult(expenses, pagination)` | query | the signed-in member of staff, own entries only |
| `expenseCategories` | `ExpenseCategoriesInput()` | `ExpenseCategoriesResult(expenseCategories)` | query | any signed-in member of staff |
| `recordExpense` | `RecordExpenseInput(spentOn, amount, expenseCategoryId, memo)` | `RecordExpenseResult(expenseId)` | mutation | any signed-in member of staff. The row is written against them |
| `correctExpense` | `CorrectExpenseInput(expenseId, spentOn, amount, expenseCategoryId, memo)` | `CorrectExpenseResult(expenseId)` | mutation | the owner of that expense, nobody else |
| `removeExpense` | `RemoveExpenseInput(expenseId)` | `RemoveExpenseResult(expenseId)` | mutation | the owner of that expense, nobody else |

Each mutation returns the identifier of what it wrote and nothing more; the screen re-reads through `expenses`.

### 11.2 Screen
<!-- id: expense-entry-screen -->
<!-- target: frontend-staff -->

For: a signed-in member of staff, looking at their own entries, most recent first.

| Calls | Kind | When |
|---|---|---|
| `expenses` | query | on opening, and after every write |
| `expenseCategories` | query | on opening, to fill the category field |
| `recordExpense` | mutation | on submitting the form |
| `correctExpense` | mutation | on submitting an entry that was opened for correction |
| `removeExpense` | mutation | on confirming a removal |
| `signOut` | mutation | on signing out |

### Use cases
<!-- usecases -->

- a member of staff who paid a 1,200 yen train fare on the way to a client records it that evening — the date, the amount, the category and a short memo — and sees it in their entries
- a member of staff who typed 12,000 instead of 1,200 opens that entry, corrects the amount, and what they see from then on is the corrected one
- a member of staff who recorded the same lunch twice removes the duplicate, and their entries no longer show it

### Acceptance criteria
<!-- acceptance -->

- an expense with no amount, an amount of zero, or a negative amount is refused, and nothing is recorded
- an expense dated after today is refused
- the memo is optional: an expense recorded without one is accepted, and reads back with an empty memo rather than failing
- correcting an entry changes it in place — the number of entries a member of staff has does not change
- a removed entry is gone from every later read, and removing it a second time changes nothing
- reading, correcting or removing an expense belonging to somebody else is answered as not found, and the answer says nothing about whether it exists
- every operation this feature adds is refused without a session, before it reads anything


## 12. Monthly summary
<!-- id: monthly-summary -->
<!-- target: backend, frontend-staff -->
<!-- depends: expense-entry -->

### 12.1 Operations
<!-- id: monthly-summary-operations -->
<!-- target: backend -->

| schema | input | result | kind | caller |
|---|---|---|---|---|
| `monthlyExpenses` | `MonthlyExpensesInput(year, month)` | `MonthlyExpensesResult(expenses, totalAmount)` | query | the signed-in member of staff, own entries only |

One operation returns both the month's entries and its total, so that the two can never disagree, and so that the export foreseen for 1.2.0 calls this rather than summing again.

### 12.2 Screen
<!-- id: monthly-summary-screen -->
<!-- target: frontend-staff -->

For: a signed-in member of staff, reading one month at a time. The month it opens on is the current one.

| Calls | Kind | When |
|---|---|---|
| `monthlyExpenses` | query | on opening, and on moving to another month |

### Use cases
<!-- usecases -->

- a member of staff filing a claim at the end of the month picks that month, reads the total, and writes it on the claim form
- a member of staff checking last month against their card statement switches to the previous month and reads its entries without leaving the screen

### Acceptance criteria
<!-- acceptance -->

- the total shown equals the sum of the amounts of the entries shown, to the yen
- an expense dated the first or the last day of the chosen month is in that month; one dated the day before or the day after is not
- a month in which the member of staff recorded nothing shows a total of zero and says the month is empty, rather than an empty table
- an expense recorded, corrected or removed is reflected in the chosen month's total on the next read
- another member of staff's expense in the same month changes neither the entries shown nor the total
- the operation this feature adds is refused without a session, before it reads anything


## 13. Implementation plan

### Milestone 1 (MVP)

1. the data model, and the seeder that fills the four categories
2. sign in and sign out
3. recording, correcting and removing an expense, and reading back one's own entries
4. a chosen month's entries and their total

### Milestone 2

There is none. Everything this version builds is the MVP, and what follows it is a version of its own rather than a second milestone in this one.

### Fine to leave for later

- approval by a manager (1.1.0)
- exporting a month for a claim form (1.2.0)
- a photograph of the receipt
- categories an operator can edit
- signing up, and resetting a password by mail


## 14. Version acceptance criteria

### 1.0.0
<!-- id: version-acceptance-1-0-0 -->

- a member of staff signs in, records an expense dated today, and finds that expense counted in the current month's total
  spans: #sign-in, #expense-entry, #monthly-summary
- a member of staff who has signed out reaches neither their own entries nor any month's total until they sign in again
  spans: #sign-in, #expense-entry, #monthly-summary
- two members of staff signed in at the same time see neither each other's entries nor each other's totals, in any month
  spans: #sign-in, #expense-entry, #monthly-summary


## 15. Key file map

| Path | Role |
|---|---|
| `expense-note-backend/sequelize/migrations/` | the three tables, and the index on `(staff_member_id, spent_on)` |
| `expense-note-backend/sequelize/seeders/master/` | the four categories |
| `expense-note-backend/server/graphql/schemas/staff/` | the SDL of the seven operations |
| `expense-note-backend/server/graphql/resolvers/staff/actual/` | the resolvers, one file each |
| `expense-note-frontend-staff/app/pages/` | the three screens |

Placement is settled against the real tree `/hora-setup` reads; this table is what the spec knows before that.


## Sources

None. Nothing was handed over, and every requirement in this document was decided in the conversation that wrote it.


## Annex

None.
