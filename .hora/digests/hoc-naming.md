# hoc-naming
<!-- hora-skills-ort-core 0.2.0 -->
<!-- source: .claude/skills/hoc-naming/ -->

**Read the source above whenever this leaves a question open.**

## Class names

- UpperCamelCase, **singular noun**. `class User {}` — never `class Users {}` (a plural class name leaves no way to name an array of its instances).
- **Avoid single-word class names wherever possible** — use a compound of two or more words.
- Abstract classes are prefixed `Base~`; an abstract class extended for the application adds `App`: `BaseSample` → `BaseAppSample`. `Base~` / `App~` are called "super prefixes."
- A derived class **removes the super prefix, keeps the suffix, and replaces the prefix with a distinguishing one**: `class AlphaSample extends BaseSample {}`, `class BetaSample extends BaseAppSample {}`.

## Non-ASCII

- **Do not use non-ASCII characters** in variable names, class names, or member names.

## Variables & properties

- Property names are, in principle, **nouns**. Acceptable if clear.
- A variable or property holding an **array is always the plural form of the noun denoting its element** (`#payments` correct, `#paymentList` incorrect). Never express plurality with a `List` suffix.

  ```javascript
  // NG
  const user = users.filter(it => it.enabled)
  const paymentList = payments.filter(it => it.completed)

  // OK
  const enabledUsers = users.filter(it => it.enabled)
  const completedPayments = payments.filter(it => it.completed)
  ```

## Datetime suffixes: `At` for an instant, `On` for a date

| value | suffix | example |
| :-- | :-- | :-- |
| carries a **time of day** | `At` | `modifiedAt`, `trashedAt`, `expiredAt` |
| meaning stops at the **calendar date** | `On` | `billedOn`, `dueOn` |
| **range** — keeps the suffix, appends `From` / `To` | `AtFrom`/`AtTo`, `OnFrom`/`OnTo` | `modifiedAtFrom` / `modifiedAtTo`, `billedOnFrom` / `billedOnTo` |

- `From` / `To` mark **the two ends of a range** only. A single value meaning "in effect from this moment" is not a range end — name it for the instant it holds (`effectiveAt`), never `effectiveFrom`.
- Never drop the suffix: `modifiedFrom`, `billed` → NG.

## Method names

- A method name is a **predicate with the receiver as its subject**, not a complete sentence: `#isValidStatus()` correct ("object is valid status"), `#isStatusValid()` incorrect.
- Start with the **base (present tense) form of a verb** (`findUsers()`, `deleteUsers()`); an auxiliary verb is also allowed (`#canFindEntity()`, `#shouldHaveFulfilledInput()`).
- A **third-person singular verb or auxiliary verb prefix means the return value is boolean** (`is...()` / `has...()` / `can...()` / `should...()`, `containsInvalidTag()`).
- Starting with a particle/preposition is allowed but avoid it alone: `fromOpenedAt()` correct, `from()` incorrect. Exception: inflator methods use `.from()` / `.of()` / `.by()` as canonical vocabulary.
- A **single-word instance method name is prohibited** (`#load()` → `#loadCardNumbers()`). A single-word **static** method name is exceptionally allowed (`CardNumbersLoader.load()`).
- An entry-point method is a **transitive verb with an object**: `ChunkBuilder#buildChunks()`, not `#build()`.

### Verb choice among abstract verbs

| verb | usage |
| :-- | :-- |
| `create~` | instantiate an instance from a class |
| `generate~` | generate a primitive value |
| `build~` | generate a temporary object |
| `make~` | **not used** |
| `find~` | retrieve an entity from the DB — anything wrapping `Model.findOne()` / `Model.findAll()` uses `find`, not `get` / `fetch` |
| `fetch~` | access an external API to retrieve data |
| `extract~` | extract data from a variable/property |
| `save~` | wraps processing that persists to the DB |
| `send~` | access an external API to update external data |
| `set~` | property update (rarely used — setters are prohibited) |

For other verbs, choose a word faithful to the method's responsibility.

## Accessors

- Getter names are in principle **nouns**; a boolean-method name (third-person-singular verb / auxiliary verb) is also permitted.
- Setters are prohibited, so no setter naming rule exists.

## Higher-order function callback parameters

- Use `it` for each item; the first-layer callback is basically `(it, index, array) => ...`.
- In a nested higher-order call, name the **inner** item by the meaning of its value (`teams.flatMap(it => it.members.map(user => user.name))`).
- Name a `reduce()` / `reduceRight()` accumulator by the meaning of what it accumulates (`total`).

## Abbreviations

Any abbreviation **not on this whitelist is prohibited**. Only "generally recognized widely, not just in programming" abbreviations qualify. The whitelist is shared between JavaScript and CSS custom property names.

| abbreviation | original word | note |
| :-- | :-- | :-- |
| admin | administrator | general knowledge |
| app | application | general knowledge |
| config | configuration | general knowledge |
| enum | enumerate | - |
| env | environment | general knowledge |
| id | identity | general knowledge |
| init | initialize | - |
| int | integer | general knowledge |
| max | maximum | general knowledge |
| min | minimum | general knowledge |
| nav | navigation | general knowledge |
| sin | sine | - |
| cos | cosine | - |
| tan | tangent | - |
| Ctor | constructor | `constructor` is a JavaScript reserved word |
| noop | no operation | for description |
| faq | frequently asked questions | - |
| func | function | reluctantly allowed since `function` is reserved. Prefer a name stating the functionality. `fn` / `f` are **not** allowed |

- Long-standing programming conventions are also allowed: `char`, `exec`, `eval`, etc.
- **Prohibited:** `acc`, `arr`, `avg`, `auth`, `btn`, `cate`, `cfg`, `cnt`, `cond`, `ctx`, `e`, `err`, `ev`, `ex`, `fmt`, `msg`, `no`, `num`, `prod`, `tx`, `tz`, and so on.
- An abbreviation used inside an external module is **not** a reason to use it yourself (do not imitate Sequelize's `msg` or Express's `req` / `res`).

## Prohibited words (as prefix or suffix)

`info` (`information`) · `data` · `helper` · `manager` · `item` · `list` · `util` (`utils`) · `type` (**as a suffix**).

- `type` → use `Category` instead (`granteeCategory` / `GranteeCategory`, `eventCategory`). **The only exception is a name borrowed verbatim from an external vocabulary**: `mimeType` (MIME standard), `contentType` (HTTP `Content-Type` header), and a constant mirrored from an external package keeps that package's word (an external `COLUMN_TYPE` stays `columnTypeName`).

### Redundant compound words

A word that adds no new meaning is redundant; do not compound with it.

| redundant | good example |
| :-- | :-- |
| UserInfo | userDetail / userPayment |
| SaveData | saveUser / saveMessages |
| FileUtil | FileNameCollector |

## American spelling

Always the American form.

| American | British |
| :--: | :--: |
| color | colour |
| center | centre |
| behavior | behaviour |
| meter | metre |
| finalize | finalise |
| organize | organise |
| analyze | analyse |
| license | licence |
| dialog | dialogue |
| canceled | cancelled |

## Contrasting terms — fixed pairs

| Positive | Negative | meaning |
| :-- | :-- | :-- |
| add | remove | add / remove |
| append | extract | append to list / remove from list |
| setup | teardown | set up / tear down |
| initialize | terminalize | initial processing / termination processing |
| benefit | drawback | benefit / drawback |
| pros | cons | pros / cons |
