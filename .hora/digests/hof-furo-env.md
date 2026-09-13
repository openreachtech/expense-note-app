# hof-furo-env
<!-- @openreachtech/hora-skills-ort-furo 0.1.0 -->
<!-- source: .claude/skills/hof-furo-env/ -->

**Read the source above whenever this leaves a question open.**

Use when adding/changing an environment variable, wiring an endpoint/key, or debugging a missing runtime value.

Value flow: `.furo-env.<env>` file → `NuxtFuroEnvLoader` → `app/globals/furo-env.js` → `nuxt.config.js` `runtimeConfig` → `useRuntimeConfig().public.<VAR>`

## Env files

All at the **repo root**.

| File | Purpose | Git |
| --- | --- | --- |
| `.furo-env.example` | **Source of truth for the key list**; placeholder/safe defaults. New variables are added here first. | tracked |
| `.furo-env.test` | test env; also sets `TEST_MESSAGE` to prove the correct file loads under test | tracked |
| `.furo-env.development` | development; developer-local copy of `.example` with real values | **gitignored** |
| `.furo-env` | production (no suffix); local copy of `.example` with real values | **gitignored** |

- **Initialize local files by copying the example — never author them from scratch:**
  ```
  cp .furo-env.example .furo-env.development
  cp .furo-env.example .furo-env
  ```
- Selection by `process.env.NODE_ENV` inside `NuxtFuroEnvLoader` (from `@openreachtech/furo-nuxt`): `production` → `.furo-env`; otherwise → `.furo-env.<nodeEnv>`; missing `NODE_ENV` defaults to `development`.
- Parsed with `dotenv` — `KEY = value` syntax, `#` comments allowed. Variable names are **SCREAMING_SNAKE_CASE**.
- `nuxt.config.js` watches `.furo-env.development` and restarts dev on change. **Never commit real secrets.**

## Known variables (`.furo-env.example`)

```
ENDPOINT_URL = http://localhost:3900/graphql-customer   # GraphQL HTTP endpoint
WEBSOCKET_URL = ws://localhost:3900/graphql-customer     # GraphQL WS endpoint

RENCHAN_RESTFUL_API_BASE_URL = http://localhost:8001     # REST base URL

ROBOT_PAYMENT_STORE_ID =
CHANNEL_TALK_PLUGIN_KEY =
```

## Wiring a GraphQL endpoint

The endpoint **path** lives in the variable value, not in code: `ENDPOINT_URL` (HTTP) and `WEBSOCKET_URL` (WS) carry the full URL including the `/graphql-<audience>` path. Set them in each `.furo-env.*` file; the consumer is `plugins/000.furo.js` (together with `RENCHAN_RESTFUL_API_BASE_URL`). No `nuxt.config.js` edit is needed for a public value.

## Loader & runtimeConfig

`app/globals/furo-env.js` — loads the parsed hash once:

```js
import {
  NuxtFuroEnvLoader,
} from '@openreachtech/furo-nuxt'

const furoEnv = NuxtFuroEnvLoader.create()
  .loadEnv()

export default furoEnv
```

`nuxt.config.js` spreads it into **both** server and client config:

```js
import furoEnv from './app/globals/furo-env'

// ...
runtimeConfig: {
  // on server
  ...furoEnv,

  // on client
  public: {
    ...furoEnv,
  },
},
```

**Keeping a variable private (server-only):** the `...furoEnv` spread copies **every** key into `public`, shipping it to the client bundle. Destructure the secret out and spread the rest into `public`:

```js
const {
  MY_SECRET, // kept out of `public`
  ...publicFuroEnv
} = furoEnv
```

Read such a secret via `useRuntimeConfig().MY_SECRET` (not `.public`), from server-side code only.

## Reading a value at runtime

Always read from `.public` (a Furo SPA — `ssr: false` — reads everything off `public`):

```js
const runtimeConfig = useRuntimeConfig()

const pluginKey = runtimeConfig.public.CHANNEL_TALK_PLUGIN_KEY
```

Read it in a plugin or component `setup` — never hard-code the value.

## Adding a variable — steps

1. Add it (SCREAMING_SNAKE_CASE) to `.furo-env.example` with an empty or safe default, then propagate to the local copies that need it (`.furo-env.development`, production `.furo-env`, `.furo-env.test`) with real values.
2. **No `runtimeConfig` edit for public values** — the `...furoEnv` spread picks it up. Exception: a private/secret value must be excluded from the `public` spread.
3. Add the field to the `RuntimeConfig`/`PublicRuntimeConfig` augmentation in `types/furo-nuxt.d.ts` so `useRuntimeConfig().public.<NAME>` is typed (see the `hof-nuxt` skill).
4. Read it via `useRuntimeConfig().public.<NAME>`.

## Missing or misspelled variable — not covered

**The skill does not state what happens when a variable is absent or its name is misspelled** — no failure mode, no validation step, no error message is documented anywhere in `SKILL.md` or `references/`. Do not assume it fails loudly. Verify against `NuxtFuroEnvLoader` in `@openreachtech/furo-nuxt` before relying on any behavior here.

## Conflicts with `D:\ORT\rules\` — the rules win

- **Env access path.** `rules/javascript-style.md` requires reading configuration through the globals barrel `app/globals/_.js` and **never** `process.env` directly. This skill routes env through a different global, `app/globals/furo-env.js`, and has application code read `useRuntimeConfig().public.<VAR>`; `process.env.NODE_ENV` is read only inside `NuxtFuroEnvLoader` (package-internal, not app code). The rule wins on any disagreement. Flagged, not resolved — confirm with the caller which accessor applies in this repo before writing app code that reads config.
- **Variable naming.** The skill mandates SCREAMING_SNAKE_CASE dotenv keys, and its documented keys use the abbreviations `API`, `URL`, `WS` (`RENCHAN_RESTFUL_API_BASE_URL`, `WEBSOCKET_URL`). `rules/naming.md` bans abbreviated names, permitting only universal/customary ones, and is written about class and member names — it does not say whether dotenv keys fall under it. Tension flagged, not resolved.
