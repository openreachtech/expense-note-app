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
| session | **undecided** | — | backend / frontend | **blocked on Q1.** The mechanism is not chosen, so nothing is named for it yet |

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
| `Category`, `Expense` as a *class* name in app code | a single-word class name is avoided wherever it can be | the model keeps `Expense`; app-side classes are compound (`ExpenseRecorder`) |

**Recording the reason is the point.** Without it somebody later restores the naive name, and
lint fails on a rule they then work around locally.
