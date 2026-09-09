# hor-sequelize-model
<!-- hora-skills-ort-renchan 0.1.0 -->
<!-- source: .claude/skills/hor-sequelize-model/ -->

**Read the source above whenever this leaves a question open.**

## Grand principle

A model is a **filled-in template, not free-form code**. Every model extends
`BaseAppRenchanModel` and has the same skeleton. The six extension points
(`createAttributes` / `createOptions` / `associate` / `defineScopes` / `defineSubqueries` /
`setupHooks`) are **always laid out in the same order, even when unused**, and any that have no
content keep `super.xxx?.()` plus `// noop`. **Do not "delete to omit".** No custom static
methods, no changed initialization order — extend via a Mixin or a framework extension point.

Comments inside model code are written in **English** (e.g. `// ForeignKey must start with upper case.`).

## File placement & class declaration

| Rule | Value |
| --- | --- |
| location | one class per file, directly under `sequelize/models/` |
| filename | the **(singular) PascalCase class name** (`CustomerOrder.js` / `OriginObjectCategory.js`) |
| base class | `extends BaseAppRenchanModel` (from `../baseModel/BaseAppRenchanModel.js`), `export default class` |
| never | extend Sequelize's `Model` directly, nor `RenchanModel` directly |
| exception | only tree (Fertile Forest) models extend `FertileForestModel` (exported from the package) |

Filename = class name = registered name, because `SequelizeActivator` in `sequelize/_.js` scans
`modelsPath` and registers by class name; drifting names make association resolution grab `undefined`.

**Imports are limited to**: `@openreachtech/renchan-sequelize` (`ModelAttributeFactory` / Mixins),
the app base `BaseAppRenchanModel` (relative path), and **domain constants** used in place of
`ENUM` (`app/domain/*`). No business logic, no external I/O.

```js
import {
  ModelAttributeFactory,
} from '@openreachtech/renchan-sequelize'

import BaseAppRenchanModel from '../baseModel/BaseAppRenchanModel.js'

/**
 * CustomerOrder model
 *
 * @class CustomerOrder
 * @extends {BaseAppRenchanModel}
 */
export default class CustomerOrder extends BaseAppRenchanModel {
  // createAttributes / createOptions / associate / defineScopes / defineSubqueries / setupHooks
}
```

## The six methods — fixed order, each with JSDoc

1. `createAttributes (DataTypes)` — **abstract on the base (throws); always implement it**
2. `createOptions (sequelizeClient)`
3. `associate ()`
4. `defineScopes (Op)`
5. `defineSubqueries ()`
6. `setupHooks ()`

A **JSDoc block** stating purpose, `@param`, `@returns` sits immediately above each. The
smallest complete skeleton:

```js
export default class CustomerOrder extends BaseAppRenchanModel {
  /**
   * Define model attributes
   *
   * @param {import('sequelize').DataTypes} DataTypes - Sequelize DataTypes
   * @returns {object} Model attributes
   */
  static createAttributes (DataTypes) {
    const factory = ModelAttributeFactory.create(DataTypes)

    return {
      ...factory.ID_BIGINT,
      // ... attributes
    }
  }

  /**
   * Define model options
   *
   * @param {import('sequelize').Sequelize} sequelizeClient - Sequelize instance
   * @returns {object} Model options
   */
  static createOptions (sequelizeClient) {
    return {
      ...super.createOptions(sequelizeClient),
    }
  }

  /**
   * Define model associations
   */
  static associate () {
    super.associate?.()

    this.belongsTo(this._.Customer)
  }

  /**
   * Define model scopes
   *
   * @param {import('sequelize').Op} Op - Sequelize operators
   */
  static defineScopes (Op) {
    super.defineScopes?.(Op)

    // noop
  }

  /**
   * Define subqueries
   */
  static defineSubqueries () {
    super.defineSubqueries?.()

    // noop
  }

  /**
   * Setup model hooks
   */
  static setupHooks () {
    super.setupHooks?.()

    // noop
  }
}
```

`associate` / `defineScopes` / `defineSubqueries` / `setupHooks` are **never deleted**; an unused
one is exactly two lines — `super.xxx?.()` (optional call) then `// noop`. Putting
`super.xxx?.()` **first** is mandatory: each method on `RenchanModel` calls the Mixin's same-named
handler via `mixinsApplier`, so an override that does not call `super` **disables the Mixin**
(Backup's `afterSave`, etc. silently stop running).

## createOptions

Spread `...super.createOptions(sequelizeClient)` and add **only** this model's own extras. The
base `RenchanModel` returns
`{ modelName, sequelize, syncOnAssociation: false, timestamps: true, underscored: true }`.
Never rewrite `timestamps` or `underscored` per model. Most models finish in the spread alone.

| Extra | When |
| --- | --- |
| `tableName: 'customer_orders_bk'` | only when the physical name breaks "physical = plural / model = singular" — typically a backup `*Bk` table (model `CustomerOrdersBk` would infer `customer_orders_bks`, clashing with the real `customer_orders_bk`). Do **not** write it when inference is already correct. |
| `paranoid: true, // for deleted_at column` | only for a table with `deleted_at` soft delete (migration used `...factory.TIMESTAMPS_WITH_DELETED_AT`). Adding it without the column makes queries fail. |

## createAttributes

- Build the factory: `const factory = ModelAttributeFactory.create(DataTypes)`.
- Spread the PK **at the top**: `...factory.ID_BIGINT` (or `...factory.ID_INTEGER` for an integer PK).
  **Never hand-write `id`.**
- Attribute keys are **camelCase** (`registeredAt` / `questionsJson`). Never write the snake_case
  physical name — `underscored: true` maps it automatically.
- **Declare `allowNull` on every attribute**; declare `defaultValue` on any column that has a
  default. Keep both in sync with the identically named migration column.
- Attributes correspond **one-to-one with the migration's columns**; always change both together.
- **Do not put `createdAt` / `updatedAt` / `deletedAt` in attributes.** Columns come from the
  migration's `...factory.TIMESTAMPS`, values from `timestamps: true`. Attributes hold **business
  attributes only** (`practicalAttributeNames` and `BackupMixinModel` both exclude those three).
- A business "saved-at" time gets its **own** dedicated `DATE(3)` column — never repurpose
  `createdAt`: `savedAt` / `postedAt` / `registeredAt` / `modifiedAt` / `generatedAt` / `effectiveAt`.
  `~At` = carries a time of day; `~On` = meaning stops at the calendar date (`billedOn`); a range is
  two attributes keeping the suffix (`modifiedAtFrom` / `modifiedAtTo`), so a single instant meaning
  "in effect from this moment" is `effectiveAt`, not `effectiveFrom`.
- **Prefer defining the TypeScript type (interface) in `type.d.ts`**, collected into the TS global
  namespace `model`. Do **not** use a JSDoc `@typedef` at the bottom of the model file.

| Type | Use |
| --- | --- |
| `BIGINT` | PK / FK-like id |
| `INTEGER` | small integer PK, quantity, version |
| `STRING(n)` | variable-length string (`191` is the default length; pick `8`/`16`/`32`/`64` by use) |
| `TEXT` | long text (body, message) |
| `DATE(3)` | millisecond-precision datetime (`registeredAt` / `savedAt`, etc.) |
| `BOOLEAN` | truth value (`isActive`, etc.) |
| `JSON` | structured data (`questionsJson` / `resultJson`) |
| `DECIMAL(p, s)` | money / rate (`dailyRate`, etc.) |
| `BLOB('long')` | binary (uploaded file body) |
| `ENUM(...)` | **avoid by default**. Use a domain constant + `STRING(n)` instead |

Datetimes default to `DATE(3)`. Do not default string lengths to "255 for now" — identifiers
`STRING(32)`, display names / emails `STRING(191)`; match the migration's physical length.

```js
static createAttributes (DataTypes) {
  const factory = ModelAttributeFactory.create(DataTypes)

  return {
    ...factory.ID_BIGINT,

    registeredAt: {
      type: DataTypes.DATE(3),
      allowNull: false,
    },
    category: {
      type: DataTypes.STRING(64),
      allowNull: false,
      defaultValue: 'default',
    },
    isActive: {
      type: DataTypes.BOOLEAN,
      allowNull: false,
      defaultValue: true,
    },
  }
}
```

### FK-like columns

The attribute key **starts with an uppercase letter** (`CustomerOrderId` / `TenantBrandId`), type
`BIGINT`, with **`// ForeignKey must start with upper case.` immediately above it**. lowerCamel
(`customerOrderId`) fails to wire the relation, since Sequelize resolves the FK by
"`<associated model name>` + `Id`". Never drop the comment.

```js
// ForeignKey must start with upper case.
CustomerOrderId: {
  type: DataTypes.BIGINT,
  allowNull: false,
},
// A nullable FK (optional relation) also declares allowNull explicitly.
// ForeignKey must start with upper case.
TenantBrandId: {
  type: DataTypes.BIGINT,
  allowNull: true,
},
```

You **may** add `unique: true` to state intent (a 1:1 relation FK, a natural key), but the actual
constraint is enforced by the **migration's named unique index** — `unique: true` on the model
alone creates nothing. When written on both sides, keep them consistent.

## associate()

Always reference the associated model via **`this._.<ModelName>`** (`this._` is
`sequelize.models`, resolved after all models are registered). A direct import of another model
causes a circular dependency. `super.associate?.()` comes first.

| Method | Meaning | Side holding the FK |
| --- | --- | --- |
| `this.belongsTo(this._.X)` | I reference one X (I hold the FK) | **me** |
| `this.hasOne(this._.X)` | X references me once (1:1) | the other (X) |
| `this.hasMany(this._.X)` | X references me many times (1:N) | the other (X) |
| `this.belongsToMany(this._.X, { through: this._.Y })` | many-to-many with X (through join table Y) | the join table (Y) |

`belongsTo` and `hasOne` / `hasMany` are two sides of the same coin: `belongsTo` on the child side
that holds the FK, `hasOne` / `hasMany` on the parent side (you may declare both sides).

```js
static associate () {
  super.associate?.()

  this.hasOne(this._.CustomerOrderTotalPrice)

  this.hasMany(this._.CustomerOrderProduct)

  this.belongsToMany(this._.BrandShipment, {
    through: this._.BrandShipmentOrder,
  })
}
```

**Pass association options only when the default inference is wrong** — `through` for
`belongsToMany`, and `foreignKey` when the FK attribute departs from the convention (a role-named
FK, or two FKs pointing at the same model). Restating an option inference already resolves is a
violation:

```js
// Good example (the FK follows the convention; no options)
this.belongsTo(this._.Customer)

// Good example (the child's FK is role-named, so inference fails; name it explicitly)
this.hasMany(this._.TabGroupDisplayTab, {
  foreignKey: 'DisplayTabOriginCategoryId',
})

// Bad example (restating the option the inference already resolves)
this.belongsTo(this._.Customer, {
  foreignKey: 'CustomerId',
})
```

Include attaches loaded rows under the **singular model name** for `belongsTo` / `hasOne`
(`order.CustomerOrderTotalPrice`) and the **pluralized model name** for `hasMany` /
`belongsToMany` (`order.CustomerOrderProducts`).

A many-to-many **join-table model** is an ordinary model with a `belongsTo` to **each** end, holds
each end's `<Model>Id` FK attributes, and adds `paranoid: true` when it soft-deletes.

**Do not add DB foreign-key constraints** (`references` / `onDelete` / `onUpdate`) — referential
integrity is enforced in the app layer. Associations are declarations for JOIN and include only.

## defineScopes / defineSubqueries / setupHooks

| Method | What it is for | API used |
| --- | --- | --- |
| `defineScopes(Op)` | add named scopes (reusable units of filtering / ordering) | `this.addScope(name, options)` |
| `defineSubqueries()` | register correlated subqueries by name so `this.subquery(name, ...)` can use them | `this.addSubquery({ name, generator })` |
| `setupHooks()` | add lifecycle hooks (before/after save and find) | `this.beforeFind` / `this.afterSave` / `this.addHook(...)` |

Routine behavior belongs in a **Mixin, not hand-written here** ("clone to a history table on every
save" → pass `BackupMixinModel`, never write it in `setupHooks`). Add a per-model scope / hook only
for a one-off requirement no Mixin can express. `defineSubqueries` is implemented only in models
needing a correlated subquery (read the physical column name via the attribute's `.field`);
otherwise noop.

## Mixins

Override **`static get Mixins ()` returning an array** of MixinModels imported from
`@openreachtech/renchan-sequelize`. The `Mixins` getter **and the abstract getters a Mixin
requires go after the six methods**, at the end of the class.

```js
import {
  BackupMixinModel,
  ModelAttributeFactory,
} from '@openreachtech/renchan-sequelize'

import BaseAppRenchanModel from '../baseModel/BaseAppRenchanModel.js'

export default class CustomerOrder extends BaseAppRenchanModel {
  // ... createAttributes through setupHooks (the six methods)

  /**
   * get: Mixin models to apply
   *
   * @returns {Array<Function>} Mixin models
   */
  static get Mixins () {
    return [
      BackupMixinModel,
    ]
  }

  /**
   * get: Backup model for BackupMixinModel
   *
   * @returns {typeof import('./CustomerOrdersBk')} Backup model declaration
   */
  static get BackupModel () {
    return this._.CustomerOrdersBk
  }
}
```

A Mixin's required abstract getter `throw`s on the base; not implementing it fails the moment that
Mixin's handler runs with `".get:XxxModel" must be inherited`. Return the related model with
**`this._.<ModelName>`** (avoids a circular import).

| Mixin | Purpose | Required abstract getter | Optional override (default) |
| --- | --- | --- | --- |
| `BackupMixinModel` | clone/append business attributes to another table on every save (history) | `BackupModel` | — |
| `LatestStatusMixinModel` | keep a status history; auto-include on find and get the latest | `StatusModel` / `StatusPhaseModel` | `orderAttributeOfStatusPhaseModel` (`'savedAt'`) |
| `SuiteVersionMixinModel` | fetch a versioned "suite" per version | `SuiteModel` | `versionKey` (`'startedAt'`) / `getSuiteSorter` |
| `PaginationMixinModel` | provide `findAllWithPagination()` and a `&pagination` scope | none | — |
| `ReferralMixinModel` | referral tree via invite codes (Fertile Forest) | `InviteCodeModel` / `ReferralNodeModel` | `inviteCodeAttributeOfInviteCodeModel` and others |
| `AttributesLinearizerMixinModel` | flatten an included nested structure into each node's `dataValues` | none | — |

`BaseMixinModel` is the base of all Mixins and is **never passed directly**.

### BackupMixinModel — the two-model pair

- In `afterSave` it `build`s → `save`s the body's business attributes (**all attributes except
  `id` / `createdAt` / `updatedAt` / `deletedAt`**) into `BackupModel`, appending a generation.
- **Pass the Mixin to the body table** (`CustomerOrder`); its `BackupModel` getter returns the
  **backup table** (`CustomerOrdersBk`).
- The backup table is an **ordinary model** (no Mixin) holding the body's business attributes plus
  a save time (`savedAt`), and it **declares `tableName`** because its pluralized inference clashes
  with the real `<original table name>_bk`.
- The body table's `setupHooks()` must still call `super.setupHooks?.()`, or Backup's `afterSave`
  is swallowed.

### Other mixins — how to pass

```js
// LatestStatusMixinModel — belongsToMany(StatusModel, { through: StatusPhaseModel }),
// auto-included in beforeFind; instance's latestStatus is the head of savedAt descending.
static get StatusModel () { return this._.OrderStatus }
static get StatusPhaseModel () { return this._.OrderStatusPhase }

// SuiteVersionMixinModel — hasMany(SuiteModel); findCurrentSuite() / findAllSuites() /
// createWithSuite(); pulls the latest version at or before versionKey.
static get SuiteModel () { return this._.PriceTableRow }
static get versionKey () { return 'effectiveAt' } // when overriding the default 'startedAt'

// ReferralMixinModel — createByInviteCode() / buildByInviteCode(); resolves the parent node in
// beforeCreate, sprouts a Fertile Forest node in afterCreate.
static get InviteCodeModel () { return this._.CustomerInviteCode }
static get ReferralNodeModel () { return this._.CustomerReferralNode } // extends FertileForestModel

// PaginationMixinModel / AttributesLinearizerMixinModel — just pass them; no getter needed.
```

full text: .claude/skills/hor-sequelize-model/references/mixins.md#how-to-pass-each-mixin
