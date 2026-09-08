# hor-cookie-authentication
<!-- hora-skills-ort-renchan 0.1.0 -->
<!-- source: .claude/skills/hor-cookie-authentication/ -->

**Read the source above whenever this leaves a question open.**

**Scope of this digest: the DATABASE half only** — the credential and token tables' migrations and models. The skill also covers `SessionClerk`, the `RefreshTokenExpressCookieClerk` HttpOnly-cookie clerk, the `signIn` / `signUp` / `signOut` / `renewAccessToken` resolvers, the engine's `schemasToSkipFiltering` auth-filter policy, the `AUTH_*` env facade and a dev-login seeder; those exist and are **out of this digest's scope** — read `SKILL.md` and the matching reference when you need them.

## Read this first — the equipped skill is incomplete on the secrets tables

The project's spec at `specs/1.0.0/spec.md` §9 is the authority for this cluster, **not** the equipped skill. Two known omissions in `0.1.0` (fixed upstream in `hora-skills-ort-renchan` PR #37 / `release/0.2.0`, not published; this project pins `^0.1.0`):

| Equipped `0.1.0` says | The authority (spec §9, Q8) says |
| --- | --- |
| `references/migrations.md` lists four tables and **omits `<actor>_secrets` and `<actor>_secrets_bk`** | nine tables, **including both secrets tables** |
| `references/token-models.md` names `<Actor>Secret` in the cluster but gives it no columns, no code, and does **not** say it takes the backup mixin | `<Actor>Secret` **takes the backup mixin**, like the password hash |

The rule the reference never wrote down: **both credential tables (`<Actor>Secret`, `<Actor>PasswordHash`) take a backup mixin; the token tables take none** — a token is deleted or revoked, never rewritten, so there is no prior value to keep, and `<actor>_access_tokens` / `<actor>_refresh_tokens` have no `_bk`.

**A checkpoint finding the skill and the spec in disagreement here must not "correct" the spec back.** Full text: `.hora/questions/1.0.0/open.md#Q8` — including the two places §9 is deliberately stricter than the running projects: `<actor>_secrets.<actor>_id` is **unique**, not plainly indexed, and `<actor>_secrets.email` is **unique** where the running projects index it not at all.

## The per-actor table cluster

An authenticated actor's credential and session live in a small cluster of tables, kept apart from the actor's profile. For an actor `<Actor>`:

| Model | Table | Role |
| --- | --- | --- |
| `<Actor>` | `<actor>s` | the entity; its profile fields are the app's own design (see `hor-database-design`) |
| `<Actor>Secret` | `<actor>_secrets` (+ `<actor>_secrets_bk`) | the sign-in identifier (`email`), kept apart from the profile |
| `<Actor>PasswordHash` | `<actor>_password_hashes` (+ `<actor>_password_hashes_bk`) | the password digest; verifies a candidate password |
| `<Actor>AccessToken` | `<actor>_access_tokens` | the short-lived token (15 min), sent on the `x-renchan-access-token` header |
| `<Actor>RefreshToken` | `<actor>_refresh_tokens` | the long-lived token; **stored only as a digest**, delivered as an `HttpOnly` cookie |

Each is a normal renchan model (`hor-sequelize-model`); one create-table migration per credential / token table, written with the standard migration shape (`MigrationAttributeFactory`, a `COLUMN_NAME` map, snake_case `field:`, `addIndex`) — `hor-sequelize-migration` for the skeleton.

Columns map one-to-one to the model attributes — camelCase attribute ↔ snake_case column via `underscored: true`. **Add or change a column in the migration and the model together.** The reference's column lists name auth-specific columns only, which is why `id` and `created_at` / `updated_at` never appear in them (the factory supplies them; `saved_at` sits *beside* `created_at` / `updated_at`, it does not replace them).

## Columns per table

- `<actor>_password_hashes` (+ `_bk`) — `<actor>_id`, `password_hash`, `saved_at`.
- `<actor>_access_tokens` — `<actor>_id`, `access_token`, `session_key`, `generated_at`, `expired_at`.
- `<actor>_refresh_tokens` — `<actor>_id`, `token_hash`, `session_key`, `used_at`, `revoked_at`, `generated_at`, `expired_at`.
- `<actor>_secrets` (+ `_bk`) — **absent from the equipped reference**; per spec §9.4 / §9.8: `<actor>_id`, `email`, `saved_at`.

The `_bk` backup table **mirrors the source columns** and adds nothing; it pairs with the credential model's backup mixin.

## Indexes

- **Unique index on the token digest** (`token_hash`) and on `access_token` — each is the lookup key, and the unique index makes a duplicate impossible.
- **Index `session_key`** on both token tables — rotation and revocation query by series.
- Foreign keys (`<actor>_id`) are plain `BIGINT` columns with an index and **no** DB-level `references` constraint — the renchan convention. Nothing here declares `references` / `onDelete` / `onUpdate`.
- Index kind by cardinality: a **1:1** satellite FK gets a **unique** index (`<actor>_secrets`, `<actor>_password_hashes`); a **1:N** child FK gets a **plain** index (both token tables, and both `_bk` tables — a history table is the one place an actor has many rows, so in `<actor>_secrets_bk` neither the FK nor `email` is unique).

## The rotation columns (refresh token)

`sessionKey` is the **series**; `usedAt` / `revokedAt` / `expiredAt` drive rotation and reuse detection. Only the **digest** is stored (`tokenHash`, unique) — a dump of the table is not a set of usable sessions.

```js
// <Actor>RefreshToken.createAttributes — the auth-specific columns
<Actor>Id: {
  type: DataTypes.BIGINT,
  allowNull: false,
},
tokenHash: {
  type: DataTypes.STRING(191),
  allowNull: false,
  unique: true, // the digest, never the token; the lookup key
},
sessionKey: {
  type: DataTypes.STRING(191),
  allowNull: false, // the series a rotation keeps
},
usedAt: {
  type: DataTypes.DATE,
  allowNull: true, // set when spent on rotation
},
revokedAt: {
  type: DataTypes.DATE,
  allowNull: true, // set on sign-out or reuse
},
generatedAt: {
  type: DataTypes.DATE,
  allowNull: false,
},
expiredAt: {
  type: DataTypes.DATE,
  allowNull: false,
},
```

`<Actor>AccessToken` mirrors this but has **no `usedAt` / `revokedAt`** (an access token is deleted, not flagged) and carries the `accessToken` value (**unique**) plus the same `sessionKey`.

Lifetimes: the refresh-token model reads the TTL from the env facade, with a fallback default; the access-token lifetime is **not** env-driven — a fixed short module constant (15 minutes) in the access-token model.

```js
static get ttlDays () {
  return Number(env.AUTH_REFRESH_TOKEN_TTL_DAYS)
    || DEFAULT_REFRESH_TOKEN_TTL_DAYS // 14
}
```

## Store the digest, hand back the plaintext

The token models never store the plain token. `buildWithGeneratedAttributes` takes the plain refresh token and stores only its digest through `hashToken`; the plaintext is returned by the clerk for the cookie.

```js
static buildWithGeneratedAttributes ({
  userId,
  sessionKey,
  refreshToken,
  generatedAt,
  expiredAt = this.createExpiredAt({ generatedAt }),
}) {
  return this.build({
    <Actor>Id: userId,
    sessionKey,
    tokenHash: this.hashToken({ token: refreshToken }), // only the digest is persisted
    generatedAt,
    expiredAt,
    usedAt: null,
    revokedAt: null,
  })
}
```

## State reads live on the model

Availability and reuse are model methods, so a resolver never re-implements the rules:

```js
isUsed () {
  return this.get('usedAt') !== null // presented again after this ⇒ reuse
}

isAvailable ({ pointsAt }) {
  return !this.isUsed()
    && !this.isRevoked()
    && !this.isExpired({ pointsAt })
}
```

## Audience-neutral foreign-key read

The shared `SessionClerk` holds the token model without knowing the actor, so expose the concrete foreign key through `extractUserId()`:

```js
extractUserId () {
  return this.get('<Actor>Id') // the Admin model returns AdminId
}
```

## Backup mixin — how `Mixins` / `BackupModel` are declared

Declared on the model as two static getters. `BackupModel` reaches the backup model through the model registry `this._`, whose property is the **plural `…Bk`** name:

```js
static get Mixins () {
  return [BackupMixinModel]
}

static get BackupModel () {
  return this._.<Actor>PasswordHashesBk
}
```

Both credential models carry this pair — `<Actor>PasswordHash` (shown above; full text: `references/token-models.md#password-hash`) and `<Actor>Secret` (**per spec §9, not stated by the equipped skill**; its `BackupModel` is `this._.<Actor>SecretsBk`). **Neither token model carries it.**

## Password verification

`<Actor>PasswordHash` stores `passwordHash` and verifies a candidate through the enciphered compare — **never a plain equality**:

```js
async verifiesPassword ({ password }) {
  const passwordHash = this.get('passwordHash')

  if (!passwordHash) {
    return false
  }

  return Encipher.create()
    .compare(password, passwordHash)
}
```

Anything seeded into `password_hash` must be produced by the **same enciphering `verifiesPassword` compares against** — a pre-hashed digest of a known dev password, never the plaintext, or the login never matches.
