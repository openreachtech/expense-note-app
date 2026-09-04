# Glossary

Append-only, not split per version. It stops one concept acquiring two names. A contract pins
the type names on an API's surface; this pins the class, model, method and variable names that
never appear there.

Identifiers are English, and were checked against `@openreachtech/eslint-config`'s
`id-denylist` as it is actually written in the backend row
(`node_modules/@openreachtech/eslint-config/lib/configurations/core-rule-option-hash.js`).

| Term | Identifier | Kind | Used in | Notes |
|---|---|---|---|---|
| member of staff | `StaffMember` | entity | backend / frontend | table: `staff_members`. The spec's one actor |
| expense | `Expense` | entity | backend / frontend | table: `expenses`. Also the GraphQL row type both list operations carry |
| category | `ExpenseCategory` | entity | backend / frontend | table: `expense_categories`. Never bare `Category` — a single-word class name is avoided, and `cate` is on the denylist |
| month | `year` + `month` | value pair | backend / frontend | `MonthlyExpensesInput(year, month)`. Not one packed string, so neither field needs parsing |
| own entry | scoping by `StaffMemberId` | rule | backend | not a name of its own. Every read and every write is scoped to the signed-in member of staff |
| total | `totalAmount` | value | backend / frontend | yen, an integer. Never `total` alone — it does not say of what |
| status | `status` | field | backend | `recorded` in 1.0.0. The seam 1.1.0's approval is built on |
| session | scoping by `sessionKey` | value | backend | the series a token pair belongs to. Not a class: signing out revokes every row sharing the key |
| access token | `StaffMemberAccessToken` | entity | backend | table: `staff_member_access_tokens`. Fifteen minutes, sent on a request header |
| refresh token | `StaffMemberRefreshToken` | entity | backend | table: `staff_member_refresh_tokens`. Stored as `tokenHash` only, delivered as an httpOnly cookie |
| sign-in address | `StaffMemberSecret` | entity | backend | table: `staff_member_secrets`. Named for what the convention calls it, not for `email` — the model holds the identifier, whatever it later becomes |
| a superseded credential | `StaffMemberSecretsBk`, `StaffMemberPasswordHashesBk` | entity | backend | tables: `staff_member_secrets_bk`, `staff_member_password_hashes_bk`. **`Bk` is the framework's own suffix for a backup table** — not an abbreviation this project chose, and not one to expand |
| password digest | `StaffMemberPasswordHash` | entity | backend | table: `staff_member_password_hashes`. Never `password` alone: the column holds a hash, and naming it for the plaintext invites returning it |

## Names avoided, and why

Each left-hand name is one somebody reaches for naturally, and each fails against a rule that
is enforced rather than remembered.

| The naive name | Why it fails | What was used |
|---|---|---|
| `expenseList`, `categoryList` | `list` is on the `id-denylist`, and an array is named as the plural of its element | `expenses`, `expenseCategories` |
| `expenseData`, `staffData` | `data` is on the `id-denylist`, and it tells a reader nothing | `expense`, `staffMember` |
| `expenseInfo`, `staffInfo` | `info` is on the `id-denylist`, and `information` has the same singular and plural | `expense`, `staffMember` |
| `cate`, `cat` | `cate` is on the `id-denylist`; abbreviations are prohibited in any case | `expenseCategory` |
| `amt`, `val` | `val` is on the `id-denylist`; `amt` is an abbreviation | `amount` |
| `num`, `no`, `cnt` | all three are on the `id-denylist` | `totalRecords`, `totalAmount` |
| `ctx` | on the `id-denylist` | `context` |
| `err`, `e`, `ex` | `err`, `ex` are on the `id-denylist`; the error variable is written out | `error` |
| `getExpenses`, `getMonthlyTotal` | the `get~` prefix is prohibited — it says neither what nor from where | `findExpenses` (from the DB), `extractTotalAmount` (from rows) |
| `ExpenseManager`, `ExpenseHelper` | `manager` and `helper` are prohibited: both invite a god class | name the actual responsibility — `MonthlyExpenseCollector`, `ExpenseOwnershipInspector` |
| `ExpenseUtil` | `util` adds no information about the responsibility | as above |
| `load()`, `read()` on an instance | a single-word instance method wastes the reader's mental map | `loadMonthlyExpenses()`, `readOwnExpenses()` |
| `token`, `secret` as a bare property | neither says which of the four credential kinds it is, and two of them must never be logged | `accessToken`, `tokenHash`, `passwordHash` |
| `StaffMemberEmail` for the secrets table | names the column rather than the concept, so a later identifier change makes the class name a lie | `StaffMemberSecret`, as the convention calls it |
| `StaffMemberPasswordHashBackup` | `Bk` is the framework's suffix and the backup mixin resolves `<Model>Bk` by name; spelling it out breaks that lookup | `StaffMemberPasswordHashesBk` |
| `Category`, `Expense` as a *class* name in app code | a single-word class name is avoided wherever it can be | the model keeps `Expense`; app-side classes are compound (`ExpenseRecorder`) |

**Recording the reason is the point.** Without it somebody later restores the naive name, and
lint fails on a rule they then work around locally.
