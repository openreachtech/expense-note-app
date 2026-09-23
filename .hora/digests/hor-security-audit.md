# hor-security-audit
<!-- hora-skills-ort-renchan 0.1.0 -->
<!-- source: .claude/skills/hor-security-audit/ -->

**Read the source above whenever this leaves a question open.**

## Scope

A **read-only checklist audit** of a Node (JS/TS) repository: "**check and list places that could
become vulnerabilities** — not to fix them." Work through every check, run its detection commands,
judge each against its finding criteria, and produce a single **findings report**.

- **Stack-agnostic within the Node ecosystem.** Frameworks / ORMs / loggers are named only as
  *examples*. Discover what the target project uses (`package.json`, the entry point, the config
  files) and adapt each check's patterns. "Do not assume any specific file layout — detect it."
- For a **diff-only review of pending changes**, use `/security-review` instead.

## Rules (read first)

1. **Read-only. Never edit, never fix.** Only inspect, run non-mutating commands, and report.
   Suggest a remediation per finding as text, but do not apply it.
2. **Never print secret values.** Show the **key name and a masked value** (`API_KEY=abcd…(masked)`,
   or `****`). Never paste full tokens / passwords into the report or transcript.
3. **Do not exfiltrate.** No network calls that send repo contents anywhere. A dependency-advisory
   check against the registry (e.g. `npm audit`, which sends only package metadata) is fine; sending
   files is not.
4. **Every check must appear in the report** — as one or more findings, `PASS`, or `N/A` (with a
   one-line reason). "A gap must never look like 'not applicable'."
5. **Detect, don't assume.** Confirm which tools the project uses and where the relevant code lives
   before running a check. If a check's subject does not exist in this project, mark it `N/A`.

## How to run

1. **Profile the project** (about a minute): read `package.json` (scripts, deps, `engines`, `type`),
   find the server entry point and the config / env files, identify the HTTP framework, the ORM / DB
   driver, the GraphQL server (if any), and the logger.
2. Go **category by category** through the checklist. Per check: run the detection commands (adapted
   to this project), decide **PASS / FINDING / N/A**, record findings in the format below.
3. **Prefer `git grep`** so the search covers tracked source only (no `node_modules`, no build output).
4. Print the **summary table** (every check → verdict), then the detailed findings, most severe first.

## Finding format

```
### [HIGH|MEDIUM|LOW|INFO] <short title>   (check: <category>/<id>)
- Location: <file:line, or scope e.g. "the ORM connection config, non-local environments">
- Evidence: <masked snippet or command output>
- Risk: <why this is exploitable / what it exposes>
- Recommendation: <what to change — DO NOT apply it>
```

Severity guide, verbatim: **HIGH** = exploitable now / secret exposed / unauthenticated sensitive
access. **MEDIUM** = weakens the security posture (no transport encryption, weak config, unrestricted
upload type). **LOW** = hardening gap. **INFO** = worth noting, not a vulnerability.

## Checklist (22 checks)

| # | Check | Detail file |
| --- | --- | --- |
| 1 | Injection (SQL / NoSQL / command) via raw/interpolated input | `references/code-safety.md` |
| 2 | `eval` / dynamic-exec code | `references/code-safety.md` |
| 3 | Malicious / obfuscated code or install hooks | `references/code-safety.md` |
| 4 | Exposed ports / bind address / datastore exposure | `references/network-and-logging.md` |
| 5 | Production logging via a redacting logger; no PII | `references/network-and-logging.md` |
| 6 | Auth enforced on every endpoint (routes + GraphQL queries/mutations/**subscriptions**) | `references/auth-and-transport.md` |
| 7 | Public / guest allow-list is minimal & intentional (classified) | `references/auth-and-transport.md` |
| 8 | Datastore transport encryption (TLS / SSL) | `references/auth-and-transport.md` |
| 9 | Introspection / playground / debug endpoints disabled in prod | `references/auth-and-transport.md` |
| 10 | CORS scoped (not wildcard in production) | `references/auth-and-transport.md` |
| 11 | Rate limiting on public endpoints (and not bypassable) | `references/auth-and-transport.md` |
| 12 | Env files covered by `.gitignore` (all variants) | `references/secrets.md` |
| 13 | No secrets in committed env files; secret-free template present | `references/secrets.md` |
| 14 | No hardcoded secrets in code / config | `references/secrets.md` |
| 15 | No plaintext passwords / secrets in seed / fixture data | `references/secrets.md` |
| 16 | No vulnerable dependencies (advisory audit) | `references/dependencies.md` |
| 17 | Lockfile committed + version-pinning delay + no committed registry tokens | `references/dependencies.md` |
| 18 | Reproducible installs (clean / locked install, not mutating) | `references/dependencies.md` |
| 19 | File-upload validation (type + content + size) | `references/application-surface.md` |
| 20 | No unused / scaffold / boilerplate operations exposed | `references/application-surface.md` |
| 21 | GraphQL query depth / complexity limits | `references/application-surface.md` |
| 22 | Error responses don't leak internals in production | `references/application-surface.md` |

> Detection commands are condensed below to their search targets. **Full command lines per check: the
> reference file named in the table.**

## Code safety — checks 1–3

### 1. Injection (SQL / NoSQL / command)

Most ORMs / query builders parameterize by default. Risk appears where a **raw query is built with
string interpolation / concatenation from untrusted input** (request body, query params, headers,
path params, GraphQL variables, message payloads).

Search raw-query entry points for the project's library (`.query(`, `.raw(`, `$queryRawUnsafe`,
`$executeRawUnsafe`, `literal(`, `QueryTypes`, `createQueryBuilder`), the dangerous shape (a template
literal with `${` inside `.query(` / `.raw(`, or `WHERE|SELECT|INSERT|UPDATE|DELETE` followed by `+`
or `${`), and Mongo finders taking a raw `req.` / `request.` / `input` / `variables` / `body` object.

- **Distinguish the source of interpolated values.** Interpolating a **constant** (e.g. a table name
  from a file-local constant in a migration) is not injection. Interpolating anything tracing back to
  **request input** (`variables`, `input`, `req`, `args`, `body`, `params`, `headers`) is.
- **NoSQL:** a raw request object straight into a Mongo filter allows operator injection
  (`{ "$gt": "" }`) even without string building. Flag request objects used as query filters without
  validation / casting.

| Verdict | Criterion |
| --- | --- |
| **HIGH** | a raw query whose interpolated / concatenated value comes from user input, or a request object used directly as a NoSQL filter. Recommend parameterized queries (placeholders / bind params), the ORM's safe builder, or validating + casting the input first |
| **LOW/INFO** | raw query with only constant interpolation — note it, confirm the value cannot become input-derived |
| **PASS** | no raw queries, or raw queries use bind params / only constants; NoSQL filters are validated |

### 2. `eval` / dynamic-exec code

Search: `eval(`, `new Function(`, `vm.(run|compile|Script)`, `require('vm')` / `from 'vm'`;
`child_process`, `execSync`, `spawnSync`, `exec(`, `spawn(`, `execFile(`.

| Verdict | Criterion |
| --- | --- |
| **HIGH** | `eval(` / `new Function(` / `vm` run on any value that could be attacker-influenced; `child_process` exec/spawn with interpolated input (command injection — a shell string built from input is worse than an args array) |
| **LOW** | `child_process` with fully-static args (still worth noting) |
| **PASS** | none found, or all uses are on static values |

### 3. Malicious / obfuscated code & install hooks

Look for code that decodes-then-executes, phones home, or runs during install. Search: install-time
hooks (`"preinstall"`, `"install"`, `"postinstall"`, `"prepare"`, `"prepublish(Only)"`) in the root
**and any workspace** `package.json`; obfuscation (`Buffer.from(…base64`, `atob(`, `\xNN` hex escapes,
`fromCharCode`, `eval(` over `atob`/`Buffer`); outbound `curl … | sh` / `wget … | sh` and hard-coded
`https?://` hosts.

| Verdict | Criterion |
| --- | --- |
| **HIGH** | an `install` / `postinstall` / `prepare` script that downloads or executes remote code; base64 / hex-decoded strings passed to `eval` / `Function`; outbound calls sending repo / secret data to an unknown host |
| **INFO** | hard-coded external URLs — list them so a human can confirm each is expected (API endpoints, docs) and not exfiltration |
| **PASS** | no install hooks that fetch / execute; no decode-then-run; outbound hosts all expected |

Also sanity-check recent history (`git log -p -n 50 -- package.json`) for unexpected additions to
build / install scripts.

## Network exposure & logging — checks 4–5

### 4. Exposed ports / bind address / datastore exposure

Find every port reachable from outside and confirm each is intended — cover container definitions, the
process/service manager, and the server's bind address. The key risk is a **datastore or internal
service bound to a public interface (`0.0.0.0`)** instead of loopback / a private network. Search
`Dockerfile*` / `docker-compose*` / `compose*.y*ml` / `*.k8s.y*ml` (`EXPOSE`, `ports:`,
`0.0.0.0`), `listen(` / `.PORT` / `host:` / `hostname` / `127.0.0.1` in source, and the datastore
host/port keys in `.env*`.

| Verdict | Criterion |
| --- | --- |
| **HIGH** | a datastore (DB / Redis / Mongo / message queue) or an internal-only service published on `0.0.0.0` / a public interface, or a container `ports:` mapping binding a host port with no restriction. Datastores should bind to loopback / a private network only |
| **MEDIUM** | an app port exposed more broadly than needed, or an `EXPOSE` of a debug / admin port |
| **N/A** | no container / manager files (note it) — but still check the server bind address |

List every listening port + its bind scope so a human can confirm each is intentional.

### 5. Production logging via a redacting logger; no PII

Production logs should flow through the project's **structured / redacting logger** (whatever it uses),
not raw `console.*`. Wherever logging does **not** go through that logger, confirm **no PII is
written** — **email, password, tokens, phone, full name are forbidden**; opaque ids and non-personal
identifiers are OK. Identify the logger first (pino / winston / bunyan / a wrapper / `createLogger`),
then grep `console.(log|info|warn|error|debug)` excluding tests / `scripts/` / `bin/`, then PII terms
near a log call (`email|password|passwd|accessToken|token|secret|phone|fullName|firstName|lastName|
ssn|creditCard`).

| Verdict | Criterion |
| --- | --- |
| **HIGH** | any log call (logger or not) that writes **PII** (email, password, token, phone, name, and similar). Recommend logging a non-personal id instead |
| **MEDIUM** | raw `console.*` on a production code path (bypasses the logger's redaction / sinks). Recommend routing through the project logger |
| **PASS** | production logging goes through the redacting logger and no PII is logged; logging ids only is fine |

If the project's framework logs via an **injected logger, treat that as compliant** — but still scan
those calls for PII in their payloads.

## Auth & transport — checks 6–11

First identify: the HTTP framework (Express / Fastify / Koa / Nest / Hapi …), whether there is a
GraphQL server (Apollo / Yoga / Mercurius / a framework integration), and **how the project expresses
"this endpoint is authenticated"** (middleware, guard, decorator, a per-operation filter, an
allow-list of public operations). Then adapt the checks.

### 6. Auth enforced on every endpoint

Goal: **every route and every GraphQL operation is authenticated by default**, and the set of
operations reachable **without** auth is small, explicit, and intentional. Identify which enforcement
model the project uses:

| Model | What to verify |
| --- | --- |
| **Default-deny** — a global guard / filter authenticates everything; a named allow-list enumerates the public operations | the guard is actually wired; audit the allow-list (check 7) |
| **Per-endpoint opt-in** — each route/resolver attaches its own auth middleware / guard | none are missing it — "the failure mode here is a forgotten guard" |

**HTTP routes.** Grep how auth is expressed (`authenticate|requireAuth|isAuthenticated|ensureAuth|
@UseGuards|AuthGuard|passport|verifyToken|visa`), then enumerate the routes and cross-check each
has auth.

| Verdict | Criterion |
| --- | --- |
| **HIGH** | a route exposing sensitive data / actions with no auth middleware / guard |
| **MEDIUM** | a route whose auth status is ambiguous (no clear guard, not an intentional public route) |

Public routes (health check, an integration webhook, a public form submit) must be **intentional** and
otherwise protected (signature verification, IP allow-list, token, captcha).

**GraphQL — queries, mutations, and subscriptions.** "Subscriptions are frequently forgotten: a
project may authenticate queries/mutations over HTTP yet leave the WebSocket subscription transport
(`onConnect` / connection init) unauthenticated. Check all three operation types." Search the
enforcement point (`authorize|isAuthorized|isAllowed|checkPermission|hasPermission|Unauthenticated|
Unauthorized|requireAuth|context: …auth`), the subscription transport (`onConnect|connectionParams|
subscriptions:|useServer|SubscriptionServer`), and the operation inventory (`type Query|Mutation|
Subscription`, `@Query(`/`@Mutation(`/`@Subscription(`, `static get schema`).

| Verdict | Criterion |
| --- | --- |
| **HIGH** | the auth check is missing entirely (everything becomes public), **or** the subscription transport authenticates on connect differently from (or weaker than) queries/mutations |
| **HIGH** | a **state-changing mutation** reachable unauthenticated that is not an intentional public operation |
| **MEDIUM** | an operation whose coverage is ambiguous (neither clearly guarded nor an intentional public entry) |

### 7. Public / guest allow-list is minimal & intentional (classified)

If the project has an explicit list of unauthenticated operations / routes, enumerate it and
**classify every entry**, because severity depends on what the operation does:

| Allow-list entry | Verdict |
| --- | --- |
| **Read-only, non-sensitive** (health check, public content query, public form submit) | likely intentional; confirm and mark **INFO/PASS** |
| **State-changing** (any mutation that writes data) | **HIGH** unless there is a compensating control (signature, captcha, strict rate limit, idempotency) — "a public write is an abuse / spam / resource-exhaustion vector" |
| **Sensitive read** (PII, internal data) exposed publicly | **HIGH** |

Report the **full allow-list** so a human can confirm each entry matches product intent. Flag any
entry that looks like leftover scaffolding rather than a deliberate public endpoint (see check 20).

### 8. Datastore transport encryption (TLS / SSL)

SQL via most ORMs / drivers → a TLS/SSL option on the connection config; Mongo → `tls=true` /
`mongodb+srv`; Redis → a `rediss://` URL / `tls` options. Find the connection config, then grep
`ssl|tls|rejectUnauthorized|sslmode|rediss:|mongodb+srv` within it.

| Verdict | Criterion |
| --- | --- |
| **MEDIUM** | a **remote / non-local** environment (staging / production) connecting to a datastore with **no TLS/SSL** → traffic is unencrypted. Recommend enabling TLS and, where the driver supports it, verifying the server certificate — `rejectUnauthorized: true` rather than `false`, which accepts any cert (**a weaker MEDIUM**) |
| **N/A** | a **local test / dev DB** on loopback (a sqlite file, a localhost container) does not need TLS — mark N/A **for that environment**, "but do not let a local exception hide a missing setting on the production connection" |
| **PASS** | all non-local datastore connections use TLS with certificate verification |

### 9. Introspection / playground / debug endpoints disabled in prod

Search `introspection|playground|graphiql|ApolloServerPluginLandingPage|debug: true|NODE_ENV`.
**Determine the actual production value, not just that the flag is mentioned.**

| Situation | Verdict |
| --- | --- |
| `introspection: true` unconditionally | **MEDIUM** |
| gated on `NODE_ENV !== 'production'` (or a config flag that is off in prod) | **PASS** — note how it is gated |
| a landing page / GraphiQL served in production | **MEDIUM** |
| introspection or an interactive playground / debug console enabled in production | **MEDIUM** — recommend disabling in prod (keep it in development only) |
| no GraphQL server | **N/A** |

### 10. CORS scoped (not wildcard in production)

| Verdict | Criterion |
| --- | --- |
| **HIGH** | `origin: '*'` (or reflecting any `Origin`) **together with** `credentials: true` — "an invalid-but-dangerous combination that browsers partly block yet often leaks in practice; at minimum it signals no origin control" |
| **MEDIUM** | wildcard origin in production for anything beyond truly public, credential-less content |
| **PASS** | origins are an explicit allow-list / env-driven and not `*` in production |

**Concrete secure pattern to recommend** (adapted to the app's real domains): allow only the app's own
base domain **and its subdomains**, plus localhost for the test / dev environment, **driven from an
env var rather than hard-coded** — e.g. an `origin` function that accepts a request origin when it
equals or is a subdomain of the configured base domain, or is a localhost origin, and rejects
everything else. Requests with **no `Origin` header** (server-to-server, curl) can be allowed since
CORS is a browser control. **Do not reflect arbitrary origins.**

### 11. Rate limiting on public endpoints (and not bypassable)

Search for the limiter, and for the common bypass (`x-forwarded-for`, `req.ip`, `trust proxy`).
Confirm a limiter is (a) actually **applied** to the public / expensive endpoints
(**unauthenticated forms, login, upload, AI/LLM calls**), not merely imported, and (b) keyed on a
**trustworthy client identifier**.

| Verdict | Criterion |
| --- | --- |
| **MEDIUM** | no rate limiting on a public, abusable, or expensive endpoint |
| **MEDIUM** | rate limiting keyed on a client-controlled header (`X-Forwarded-For`) without a correctly configured `trust proxy` setting — an attacker rotates the header to bypass it. Recommend keying on the real peer address / a correctly trusted proxy chain |
| **PASS** | limiter applied to the relevant endpoints and keyed on a trustworthy identifier |

## Secrets — checks 12–15

**Mask every secret value in the report.**

### 12. Env files covered by `.gitignore` (all variants)

A `.gitignore` with a bare `.env` line matches **only** a file named exactly `.env` — it does **not**
match `.env.development`, `.env.staging`, `.env.production`, `.env.local`, etc. Check with
`ls -a | grep -E '^\.env'`, `git check-ignore -v .env .env.*`, and
`git ls-files | grep -E '(^|/)\.env'`.

- **Adding a pattern to `.gitignore` does not untrack an already-committed file** — `.gitignore` only
  affects untracked files. A tracked env file must be removed from the index with
  `git rm --cached <file>` (and the secret rotated). **Verify with `git ls-files`, not just by reading
  `.gitignore`.**
- **FINDING (HIGH if the tracked file holds real secrets, else MEDIUM):** any `.env*` file with
  secrets is **tracked** or **not ignored**. Recommend a broad ignore (e.g. `.env*` with a
  `!.env.example` negation), `git rm --cached` for anything already tracked, and — if a secret was
  ever committed — rotating it and purging history.
- **PASS:** every secret-bearing env file is ignored; only a secret-free template is tracked.

### 13. No secrets in committed env files; secret-free template present

Committed env files (a template, or any tracked per-env file) must not carry real secret **values**.
Inspect key **names** and whether their values are populated — **do not echo the values**. For each
tracked env file, list secret-bearing key names matching
`PASSWORD|SECRET|TOKEN|API_?KEY|PRIVATE|CREDENTIAL|_KEY|DSN|CONNECTION`, then check only whether a
value is present and mask it, e.g.
`grep -E '^SOME_API_KEY=' <file> | sed -E 's/=.{0,4}.*/=****(masked)/'`.

- **FINDING (HIGH):** a real secret value sits in a committed env file. Recommend replacing it with a
  placeholder, moving the real value to an untracked file / secret manager, and rotating.
- **Best practice to recommend:** keep exactly one committed, **secret-free** template (e.g.
  `.env.example`) listing every required key with placeholder values, and source **all** real
  credentials for non-local environments from env / a secret manager — never commit them, and do not
  leave per-env files (`.env.staging`, `.env.production`) tracked.
- **PASS:** values are empty / obvious placeholders (`your-key-here`, `changeme`, `xxxx`), and a
  secret-free template exists.

### 14. No hardcoded secrets in code / config

Search inline credential assignments
(`password|passwd|secret|api_?key|apikey|access_?token|client_?secret|private_?key` assigned a string
literal) across `*.js *.ts *.cjs *.json *.yml *.yaml`, excluding matches on `process.env` / `env.` /
`null` / `placeholder|example|changeme|xxxx`; plus known key shapes anywhere in tracked files:
`AIza[0-9A-Za-z_-]{20,}`, `sk-[A-Za-z0-9]{20,}`, `ghp_[A-Za-z0-9]{20,}`, `AKIA[0-9A-Z]{16}`,
`-----BEGIN [A-Z ]*PRIVATE KEY-----`.

- **Pay special attention to connection / datastore config files** — per-environment blocks commonly
  carry hardcoded `username` / `password`. "Hardcoded credentials in a committed config file are a
  finding **even if they look like placeholders**, because they normalize the pattern"; production
  credentials should come from env / a secret manager.
- **FINDING (HIGH real / MEDIUM placeholder):** a literal credential in tracked code / config.
  Recommend sourcing from env / a secret manager; rotate if real.
- **PASS:** credentials only ever come from `process.env` / a config facade.

### 15. No plaintext passwords / secrets in seed / fixture data

| Data kind | Requirement |
| --- | --- |
| **Production / master seed data** (data that ships to real environments) | **no** credentials at all — "production accounts are provisioned out-of-band, not seeded" |
| **Development / test fixtures** | may create login accounts, but any password must be **hashed** (through the app's real hashing path), never stored as plaintext that reaches the datastore |

Locate seed / fixture dirs (names vary: `seed*`, `fixture*`, `factories`), grep them for
`password|passwd|secret|api_?key|token|raw_password`, and confirm hashing
(`hash|bcrypt|argon2|scrypt|pbkdf2|password_hash`).

- **FINDING (HIGH):** a plaintext `password` / secret literal in production / master seed data.
  Recommend removing it (provision production credentials out-of-band).
- **FINDING (MEDIUM):** a dev / test fixture stores a plaintext password that reaches the datastore
  unhashed.
- **PASS:** production seeds hold only non-secret reference data; dev fixtures hash any password.

## Dependencies & install hygiene — checks 16–18

Detect the package manager from the committed lockfile: `package-lock.json` → npm, `pnpm-lock.yaml` →
pnpm, `yarn.lock` → Yarn, `bun.lockb` → Bun. Use that manager's commands.

### 16. No vulnerable dependencies (advisory audit)

`npm audit --omit=dev` (fallback `npm audit`) / `pnpm audit --prod` / `yarn npm audit --environment
production` (Berry) / `yarn audit` (classic) — registry metadata only, no repo upload.

- **FINDING (severity = the advisory's):** report each **high / critical** advisory — package,
  installed version, advisory title, and the fixed version. **Summarize moderate / low as counts.**
- **Remediation guidance to include per advisory:**
  - a non-breaking fix (patch / minor within the current range) → recommend the manager's safe
    upgrade, e.g. `npm audit fix` (**no `--force`**), which respects semver ranges.
  - the only fix is a **major** bump or requires `--force` → do **not** recommend applying it blindly;
    call it out as a breaking change needing a compatibility review and testing.
  - the vulnerable package is **transitive** → recommend upgrading the parent, or a pin / override
    (`overrides` in npm, `resolutions` in Yarn, `pnpm.overrides`) as a stopgap.
- **PASS:** no high / critical advisories.
- If the audit cannot reach the registry (offline / private-registry auth), say so and mark the check
  **BLOCKED** rather than PASS.

### 17. Lockfile committed + version-pinning delay + no committed registry tokens

| Item | Criterion |
| --- | --- |
| **Lockfile committed** | required for reproducible / clean installs (check 18). Missing lockfile → **MEDIUM** |
| **Version-pinning delay** | a cooldown before installing a just-published version weakens a class of supply-chain attacks (a malicious release pulled before it is caught). npm: `min-release-age`; pnpm: `minimumReleaseAge`. **Recommend ≥ 7 days** where the manager supports it. Missing / `< 7` → **LOW** |
| **No committed registry credentials** | a private-registry token (`_authToken`, `_auth`, `npmAuthToken`, an inline `NPM_TOKEN=`) committed in `.npmrc` / `.yarnrc.yml` → **HIGH**; such tokens must come from env / CI, not the repo |
| **PASS** | lockfile tracked, cooldown set (where supported), no committed tokens |

### 18. Reproducible installs (clean / locked install, not mutating)

CI, container builds, and production installs should use the manager's **clean, lockfile-exact**
install — which fails if the lockfile is out of sync and does not mutate it.

| Manager | Reproducible install | Avoid in CI/build |
| --- | --- | --- |
| npm | `npm ci` | `npm install` / `npm i` |
| pnpm | `pnpm install --frozen-lockfile` | `pnpm install` (writable) |
| Yarn Berry | `yarn install --immutable` | writable `yarn install` |
| Yarn classic | `yarn install --frozen-lockfile` | writable `yarn install` |

Search where installs happen: `.github/**`, `*.sh`, `package.json`, `Dockerfile*`,
`docker-compose*`, `docs/**`.

- **FINDING (LOW/MEDIUM):** a mutating install used in CI, a container build, or a deploy / setup
  script where the clean/locked variant is appropriate. Local first-time-setup docs using a plain
  install are usually acceptable — note but don't over-flag.
- **PASS:** CI / build / deploy paths use the clean / locked install, and the lockfile is committed
  (check 17).

## Application surface — checks 19–22

### 19. File-upload validation (type + content + size)

Any endpoint that accepts an uploaded file must validate it in **three** ways; checking only one
(commonly just size, or just the client-declared MIME type) is insufficient:

1. **Declared type allow-list** — accept only the MIME type(s) / extension(s) the feature needs.
2. **Content / magic-byte check** — verify the bytes actually match the claimed type. The client-sent
   MIME type and filename extension are attacker-controlled and trivially spoofed; a real check reads
   the file's magic number (e.g. `%PDF-` for PDF, `\x89PNG` for PNG, `PK\x03\x04` for zip/office).
3. **Size limit** — cap the accepted size to bound memory / disk / downstream cost.

| Verdict | Criterion |
| --- | --- |
| **MEDIUM** | an upload endpoint that validates size only, or trusts the client-declared MIME type without a magic-byte / content check (spoofable). Recommend a type allow-list **plus** a content check **plus** a size cap |
| **MEDIUM/HIGH** | no validation at all on an upload that is stored, parsed, or forwarded (parsing an unexpected type can be an exploitation or DoS vector) |
| **PASS** | uploads enforce type allow-list + content/magic check + size limit |
| **N/A** | no uploads |

A robust content check reads the leading bytes and compares against the expected magic number for the
allow-listed type(s), and rejects anything that does not match **even if the declared MIME type looks
correct**.

### 20. No unused / scaffold / boilerplate operations exposed

Generated scaffolding, boilerplate CRUD and leftover example endpoints are **attack surface** even
when the product does not use them — "they are often less reviewed, may be unauthenticated, and can
mutate data. Every reachable route / resolver / subscription should correspond to a real product
feature." Enumerate all operations, then grep scaffolding smells
(`example|sample|scaffold|boilerplate|todo|foobar|test-?only|demo|playground`).

| Verdict | Criterion |
| --- | --- |
| **MEDIUM** | an exposed operation that is unused / scaffold **and state-changing or sensitive** — especially if unauthenticated (ties into checks 6–7). Recommend removing it to shrink the attack surface |
| **LOW** | an unused read-only operation left exposed |
| **PASS** | every exposed operation maps to a real, intended feature |

Cross-reference each operation with whether the frontend / product actually calls it; operations with
no caller, or clearly generated example CRUD, are candidates for removal. **Report the operation
inventory so a human can confirm the mapping — do not delete anything (this skill is read-only).**

### 21. GraphQL query depth / complexity limits

Without a depth / complexity / cost limit, a client can send a deeply nested or highly branching query
(e.g. cyclic relations) that forces enormous resolution work — a denial-of-service vector.

| Verdict | Criterion |
| --- | --- |
| **MEDIUM** | a GraphQL server with **no** depth or complexity limit and a schema that contains nested / cyclic relations. Recommend a depth limit and/or a query-cost / complexity rule as a validation rule |
| **PASS** | a depth and/or complexity limit is configured |
| **N/A** | no GraphQL server, or a trivial flat schema with no nesting (note the reasoning) |

### 22. Error responses don't leak internals in production

Error responses returned to clients must not include **stack traces, raw DB / driver errors, internal
file paths, or SQL**. "Detail belongs in server logs (via the redacting logger, check 5), not the HTTP
/ GraphQL response." Search `stack` / `formatError` / `maskError` / `debug: true` / `NODE_ENV` /
handlers returning a raw error. **Determine what the client actually receives in production** — a
GraphQL server exposing `error.extensions.exception.stacktrace`, or an HTTP handler returning
`err.stack` / the raw error message, is a leak; a production error formatter that maps to a safe
message / code while logging detail server-side is correct.

| Verdict | Criterion |
| --- | --- |
| **MEDIUM** | stack traces / raw internal errors returned to clients in production. Recommend a production error formatter that returns a generic message + a correlation id, and logs the detail server-side |
| **PASS** | production responses carry only safe, sanitized errors; detail is logged, not returned |

## Additional checks worth proposing

Offer these when relevant, reported as findings / PASS / N/A like the rest:

- **Secrets in git history** (not just the current tree) — scan `git log -p` for secret patterns and
  known key shapes; **a secret that was ever committed must be rotated even if later removed**.
- **Datastore auth in production** — DB / cache / queue require a password and are not on a public
  interface.
- **Cookie / session flags** (`httpOnly` / `secure` / `sameSite`) if cookies are used.
- **Runtime pinning** (`engines` set; no end-of-life Node version).
- **Mass-assignment** — endpoints spreading raw request input straight into a DB write.

## Output

End with, in this order:

1. the **summary table** (every check → verdict),
2. **findings most-severe-first**,
3. the **count by severity**,
4. a **one-line reminder that nothing was changed**.

If asked, offer to hand specific findings to a fix workflow — **but this skill itself never modifies
code.** Write the report **in the language the reader is using**, "as the documentation convention
requires of any document generated for a reader."
