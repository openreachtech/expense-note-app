# hor-execution-placement-pattern
<!-- hora-skills-ort-renchan 0.1.0 -->
<!-- source: .claude/skills/hor-execution-placement-pattern/ -->

**Read the source above whenever this leaves a question open.**

## Grand principle: write processing lives in only two places — "API" or "Worker"

"In a web system, **processing that changes state (writes)** only ever happens in one of two places.
Every requirement must resolve into this binary choice. The deciding axis is **how heavy / how
long-running the processing is**."

| Placement | Execution model | What it holds |
| --- | --- | --- |
| **API** | Responds **synchronously** to the request | Things that are **light and finish quickly** |
| **Worker** | Runs **asynchronously** in the background | Things that are **heavy and take time** |

- **"When the boundary is unclear, lean toward Worker."**
- **The rule of thumb for "heavy", verbatim:** "anything involving external I/O, AI calls, large
  record counts, or file generation counts as the heavy side."
- Read-only processing is out of scope: "As a rule it just returns synchronously via the API
  (GraphQL query / REST GET) and is not turned into a Worker."
- A third place is forbidden — it "scatters responsibility, monitoring, and retry paths so they can
  no longer be traced."

## Decision flow — in this order

1. **Is it a write?** If read-only, return it via an API query / GET and you're done.
2. **Is it light & short-lived?** → **API** (§1).
3. **Is it heavy / time-consuming / uncertain due to external dependencies?** → **Worker** (§2).
4. **What triggers the Worker?**
   - User actions, external integrations, etc. — **request-based** → enqueue from inside the API handler (§2.1).
   - A side effect unrelated to the main processing, run **after the response** → dispatch from a **post-worker** (§2.2).
   - Automatic, by time / interval → **schedule-based** (§2.3).

| Nature of the processing | Placement | Concretely |
| --- | --- | --- |
| Light, short-lived, needs a synchronous response | **API** | GraphQL resolver / REST renderer |
| Heavy, time-consuming, **request-based** | **Worker (API-triggered)** | renchan-job-bullmq (enqueue from resolver/renderer) |
| A **side effect** unrelated to main processing, run **after the response** | **Worker (post-worker-triggered)** | post-worker only dispatches (e.g. sending an email after signup) |
| Heavy, **automatic by interval/time** | **Worker (scheduled)** | renchan-job-bullmq (cron / interval scheduler) |

Step 4 is reached **only** for heavy work. Light work never becomes a post-worker or a schedule.

## 1. Put it in the API (light, short-lived)

Two families; choose by the caller named in the requirement.

| Family | Where | Use for |
| --- | --- | --- |
| **GraphQL** | `server/graphql/resolvers/<role>/actual/{mutations,queries}/` | "Ordinary reads/writes from your own frontend default to this." |
| **REST API (renderer)** | `server/restfulapi/renderers/v1/{post,get}/*Renderer.js` | "file uploads, endpoints hit by external systems, and non-GraphQL clients" |

Rule of thumb for what goes here: "simple CRUD against the DB, input validation, and short
aggregations/updates that complete within a single request."

```js
// Avoid: running even heavy processing synchronously inside the resolver → the response never returns and times out
const content = await generateContentWithAi({ accessToken }) // takes tens of seconds
return { content }
```

## 2. Put it in a Worker (heavy, time-consuming)

Workers are implemented with **`@openreachtech/renchan-job-bullmq`**. Place the Manifest / Worker /
Dispatcher under **`app/jobs/<kebab-name>/`**. How to write them: the `hor-renchan-job-bullmq` skill.

Three kinds by trigger. **"In all three, the shape is the same: 'the real work is in the Worker, the
trigger is elsewhere'" — don't put logic beyond dispatch in the trigger.**

### 2.1 Request-based — enqueue from inside the API handler

The API (resolver / renderer) **only enqueues and responds immediately**; the real work runs in a
Worker in the Daemon. Progress is returned via a subscription or similar.

```js
// Good: heavy processing is enqueued and the response returns immediately (the real work is in the Worker)
await context.share.jobDispatcherProvider.dispatchJob({
  DispatcherCtor: ContentGenerationPublicLpJobDispatcher,
  body: {
    accessToken,
  },
})
return { accepted: true }
```

### 2.2 post-worker — a side effect after the response

renchan's post-worker (`BaseGraphqlPostWorker`) fires as an **`onResolved` hook per GraphQL
operation**, after the main resolver has returned.

**"A post-worker holds only the dispatch to the Worker. Don't put complex logic here"** — otherwise
logic accumulates where "neither retry, progress notification, nor monitoring apply", and failures
get swallowed.

**The test that separates §2.1 from §2.2 — "is the user requesting it?":**

| The processing | Placement |
| --- | --- |
| **The user is requesting it** — waits for the result, or it's part of the response | enqueue inside the handler (§2.1) |
| A **side effect unrelated to the main processing**, run afterward without making the response wait | post-worker (§2.2) |

```js
// Good: a post-worker only dispatches (the real work of the side effect is on the Worker side)
/** @override */
async onResolved ({ variables, context, response: { output, error } }) {
  if (error) {
    return
  }

  await context.share.jobDispatcherProvider.dispatchJob({
    DispatcherCtor: SendWelcomeEmailJobDispatcher,
    body: {
      customerId: output.customer.id,
    },
  })
}
```

To use one at all: **"set the engine's `postWorkersPath` and place a post-worker that extends
`BaseGraphqlPostWorker`."** The skill records this repository's state itself — `postWorkersPath: null`,
not wired.

### 2.3 Schedule-based

Starts **automatically by time / interval** rather than from a user. Register a cron / interval
scheduler — `*CronJobScheduler.js` / `*IntervalJobScheduler.js` (`hor-renchan-job-bullmq` skill).
For periodic batches, cleanups, reminders, re-aggregations.

```js
// Avoid: substituting a periodic cleanup with "an admin presses a button and the resolver runs a long loop"
//   → forgotten presses, double execution, and timeouts occur. Put periodic processing in a scheduler.
```

## 3. External integrations — decide the two axes separately

Integrations ride on API or Worker like everything else. **"The receiving method follows the other
party's spec, and the placement (API or Worker) follows this skill's weight criterion."**

- **inbound (receiving calls from outside)**: "often a REST API (renderer)", but **not necessarily** —
  it can be received via GraphQL. Choose by the other system's constraints. If the processing after
  receiving is heavy, the renderer sticks to enqueuing and hands the real work to a Worker (§2.1).
- **outbound (calling an external API)**: call it via the rocket-client Launcher
  (`hor-external-api-client` skill). "External calls are slow and uncertain, so call them from a
  Worker if they would block the user's response."

## This repository (1.0.0): nothing runs outside the request path, by decision

`specs/1.0.0/spec.md` §8: **"Redis is not declared, because this version runs no background job.
Every write finishes inside its own request, and nothing here leaves the process."** So for every
requirement in this version the decision flow terminates at step 2 — **API** — and step 4 is not
reached. A requirement that looks heavy is a question for the main session, not a licence to add a
job facility.

**There is no job facility here.** Verified in the tree:

| Fact | State |
| --- | --- |
| `@openreachtech/renchan-job-bullmq` | **absent** — not in `expense-note-backend/package.json`, not installed |
| `app/jobs/` | **does not exist** (`app/` holds `constants`, `globals`, `session`, `tools`) |
| `server/graphql/post-workers/` | **does not exist** |
| `postWorkersPath` | **`null`** in all three engines — `Staff`, `Customer`, `Admin` |
| `jobDispatcherProvider` / `dispatchJob` | **no occurrence** anywhere in `server/` or `app/` |

**Do not read `package.json` or the compose file as evidence that the infrastructure is already
there.** `ioredis@^5.8.0` *is* a declared dependency and a `redis:7.4` service *does* sit in
`expense-note-backend/docker-compose.development.yml` behind an opt-in `profiles: [redis]` — but
nothing imports `ioredis`, the compose header says Redis is behind a profile because "this version
runs no background job, and the queue is the only thing that would need it", and the three engines
each carry the same commented-out `NOTE: Uncomment the following line to enable Redis PubSub` above
`redisOptions: null`. That is **boilerplate residue, not a job facility.**

**What a later version would actually have to wire:**

- **A post-worker** (needs no new package — `BaseGraphqlPostWorker` ships with the installed
  `@openreachtech/renchan`): set that engine's `postWorkersPath` to a real path, create
  `server/graphql/post-workers/`, and add a class extending `BaseGraphqlPostWorker` with
  `static get schema ()` matching the operation and an `onResolved` hook. While `postWorkersPath`
  is `null`, **a post-worker file would not run at all.**
- **A job / Worker**: add `@openreachtech/renchan-job-bullmq` as a dependency, create
  `app/jobs/<kebab-name>/` (Manifest / Worker / Dispatcher per `hor-renchan-job-bullmq`), stand up
  the Daemon, expose a job dispatcher on `Share` so `context.share.jobDispatcherProvider` exists,
  and bring Redis into the default compose profile — plus declare it in the spec's §8 middleware
  table, which currently lists MariaDB only.
- `server/restfulapi/renderers/v1/{get,post}/` **do exist but are empty**, so the REST-renderer half
  of §1 is available without new wiring.

## Finishing checklist

- [ ] Is this processing a **write**? (If a read, return it via query/GET and you're done.)
- [ ] Did you resolve it into the binary of **light → API / heavy → Worker**, leaning toward Worker when unsure?
- [ ] If placing it in the API, did you choose **GraphQL resolver** or **REST renderer** based on the caller?
- [ ] If placing it in a Worker, did you choose one of **request-based (enqueue)** / **post-worker** / **schedule-based** by trigger?
- [ ] For the request-based case, does **the API side only enqueue** (no real work written in the resolver/renderer)?
- [ ] If running a side effect unrelated to main processing after the response, did you write **only a dispatch in the post-worker** and keep the real work in the Worker?
- [ ] For external integrations, did you decide the **receiving method (other party's constraints)** and the **placement (weight criterion)** separately?
- [ ] In **this** version: did the decision land on **API**, as §8 of the spec requires — and if it did not, was that raised rather than implemented?
