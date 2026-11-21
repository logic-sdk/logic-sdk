# logic-sdk

## 0.5.0

### Minor Changes

- 9fbf72b: Listeners take `attempts` and `delay`, and tests use hooks.

  ```ts
  export const sendReceipt = listener({
    event: () => orders.placed,
    attempts: 5, // tries in total; default 1, like Rails and Laravel
    delay: 60,   // seconds after the event
    async handler(ctx, { orderId }) { ... },
  });
  ```

  - **`attempts`** (default 1) are enforced in `registry.jobs.run`: out of tries, a job is `failed`, and a job claimed after its last try never finished is `failed` without running again. `inProcess()` ignores `attempts` and `delay`, like Laravel's sync queue.
  - **`onError` gets `outcome`** (`"retry"`, `"failed"` or `"discarded"`) for a listener's failures.
  - **Adapters** get optional `check({ listeners, database })`, run at `createRegistry`, and `fail(job, error)`, called when a job a queue consumer ran fails for good. Jobs carry `runAt` when their listener has a `delay`. `run` takes a `fail` callback in its options, so an adapter keeps a failed job with its error.
  - **A job this code can't run** (its listener was renamed or removed, or an unknown kind) fails at once and is kept, like Rails' `ClassMissingError`: `registry.jobs.retry(id)` runs it once the code is back. A throw from `jobs.scope.load` counts as a try, like a throw from the listener.
  - **The outbox claims a commit's jobs after commit,** in one update. A process that dies in between leaves them unclaimed, so the sweep's claim is their first try, not a lost one.

  **Breaking, tests:** `t.transaction(async (tx) => ...)` is replaced by hooks, and `listeners: false` by `jobs: "manual"`:

  ```ts
  const t = createTestRegistry(registry, { services: { mailer } });
  beforeEach(() => t.begin());
  afterEach(() => t.rollback());

  test("...", async () => {
    await t.as(agent).run(bookings.confirm, { bookingId }); // was tx.as(...)
    expect(t.events).toHaveLength(1); // t.db, t.stub, t.registry too
  });
  ```

  - `jobs: "manual"` keeps jobs in `t.jobs.queued`; `t.jobs.drain()` runs them and `t.jobs.perform(listener, payload, { scope })` runs one.
  - A failing listener now makes `run`, `drain` and `perform` throw in tests, and `rollback` when it ran through `t.registry`, so a test can't pass over it.
  - Tests never use your app's jobs adapter, and listeners finish before a call returns, even when the registry sets `waitUntil`.

- b64b8aa: `logic-sdk/cloudflare`: jobs through Cloudflare Queues.

  ```ts
  jobs: {
    adapter: outbox(db, { via: cloudflareQueue(env.JOBS) });
  } // durable: a row in your transaction, sent after commit
  jobs: {
    adapter: cloudflareQueue(env.JOBS);
  } // sent after commit, lost if the Worker dies in between

  export default {
    async queue(batch, env, ctx) {
      await handleQueue(makeRegistry(env, ctx), batch);
    },
    async scheduled(_controller, env, ctx) {
      await makeRegistry(env, ctx).jobs.sweep();
    },
  } satisfies ExportedHandler<Env, Job>;
  ```

  - `outbox(db, { via })` hands a commit's jobs to a queue in one send instead of running them in the request. A failed send keeps the rows, is reported to `onError`, and the sweep sends them again. A job that fails for good in the consumer comes back as a `failed` row.
  - `handleQueue(registry, batch)` runs each message's job and acknowledges it, or retries it with a backoff while the listener has `attempts` left, one message at a time.
  - Sends stay under Cloudflare's limits (100 messages, 256,000 bytes per send, 128,000 per message); a bigger job fails with its size. A listener's `delay` goes out as `delaySeconds`; alone, over 24 hours is refused at `createRegistry`.
  - Adapters now get a `Runner` (`{ run, report }`) in `deliver` and `sweep`, so an adapter can report its own failures.

- 2c1c7eb: The delivery contract for durable events. Each committed event becomes one `Delivery` per listener (`{ kind: "delivery", id, listener, envelope }`, `id` being `<listener id>@<event id>`), so a failing listener retries alone. Dispatchers get `enqueue(tx, deliveries)` and `deliver(deliveries, run)`; `inProcess()` is unchanged for you.

  `registry.dispatch(envelope)` is now `registry.jobs.run(delivery, { attempt })`. It resolves `"done"`, or `"failed"` when retrying can't help (a domain error other than `conflict`, or a payload that no longer matches its schema), and throws when a retry might help. `onError` gets the delivery id and attempt.

  `emit` checks that the payload and the scope are plain JSON, so a `Date` fails in tests instead of after a queue. Envelopes carry `depth`, so the loop limit holds across hops. Listeners get `ctx.event` as `{ id, name, payload, occurredAt, attempt }`, without the stored scope: `ctx.scope` is the one they run as.

- e5806dc: Functions, listeners and events no longer take a `description`. Write one sentence of JSDoc above each export instead: editors show it on hover, and `logic-sdk list` and `show` print its first paragraph.

  What an AI tool reads now lives in meta, owned by the adapter. `logic-sdk/ai` exports `AIMeta` (`{ ai?: { tool?: boolean; description?: string } }`) for your meta shape: `init<{ meta: AIMeta & AppMeta }>()`. `toAITools(registry, caller)` takes the exposed functions marked `meta: { ai: { tool: true } }` by default (`filter` is now optional), and a new `describe` option picks descriptions from your own meta. `toAITool(caller, fn, { description })` overrides one tool. Order: the call's option, then `meta.ai.description`, else none.

  Migrate: move each `description: "…"` into a `/** … */` above the export; for AI tools, put the text in `meta: { ai: { description } }` and replace `filter: (f) => f.fn.meta?.agent === true` with `meta: { ai: { tool: true } }`.

  The `logic-sdk` package now ships its agent skill at `skills/logic-sdk/SKILL.md`.

- 4d63b2f: Background work is configured under `jobs`, and what you call on it lives on `registry.jobs`:

  ```ts
  const registry = createRegistry({
    modules,
    database: drizzleAdapter(db),
    jobs: {
      adapter: outbox(db), // default inProcess()
      scope: {
        store: (s) => ({ userId: s.userId }),
        load: (saved) => scopeFor(saved),
      },
    },
  });
  await registry.jobs.sweep(); // a cron or an interval
  await registry.jobs.retry(id); // a failed job back in line
  await registry.jobs.run(message.body, { attempt: message.attempts }); // a queue consumer
  ```

  - `events: dispatcher` (the option) is now `jobs: { adapter }`, and `registry.deliver` is now `registry.jobs.run`.
  - `jobs.scope` sets who a job runs as at rest: `store` picks what's saved (checked as plain JSON at emit), `load` rebuilds it when the job runs. `load` returning `null` discards the job: a new `"discarded"` outcome, reported to `onError`, deleted from the outbox and counted by `sweep`. Applied in every adapter, `inProcess()` and tests included.
  - Every definer takes an optional `id`, local to its module (`listener({ id: "send" })` in `notifications` is `notifications.send`), so you can rename an export without orphaning queued jobs. Duplicates throw at `createRegistry`. The CLI shows and links definitions by these ids.
  - Jobs carry `kind: "delivery"`, and their id is now `<listener id>@<event id>` (BullMQ rejects `:` in job ids).
  - `registry.jobs.sweep()` runs what the adapter has due; adapters that store nothing resolve zeros. `registry.jobs.retry(id)` only revives failed jobs and returns whether it did, and throws on `inProcess()`. The outbox's `retry` used to reset any row, so it could run a pending one twice at once.
  - The adapter contract (`Dispatcher`) gets optional `sweep(run, options)` and `retry(id)`.
  - `registry.functions`, `registry.events` and `registry.listeners` are now `registry.definitions.functions`, `.events` and `.listeners`.

- e5806dc: `on(() => event, handler)` is now `listener({ event: () => event, meta?, handler(ctx, payload) })`, an object like `query`, `mutation`, `action` and `event`. `meta` now reaches `wrap` for listeners too.

  Migrate:

  ```ts
  // before
  export const sendReceipt = on(() => placed, async (ctx, payload) => { … });
  // after
  export const sendReceipt = listener({
    event: () => placed,
    async handler(ctx, payload) { … },
  });
  ```

  Rename `on` to `listener` where you destructure `init<Config>()`.

- 09466b5: `outbox(db)` in `logic-sdk/drizzle`: durable events on Postgres. Each delivery is a row written in the mutation's transaction, run after commit by the same request, and deleted when its listener is done. A listener's `attempts` decide its tries, with backoff between them (`e^attempt` seconds); `delay`ed jobs wait as rows until due. A crash is a lease running out, after which `registry.jobs.sweep()` (from a cron or an interval) takes over; a last try that never finished is failed without running again. `createRegistry` throws if the outbox isn't given the same database as `database`. Rules saying no are failed at once; failed rows stay for you to query, and `registry.jobs.retry(id)` puts one back. Re-export `outboxTable` from your schema so drizzle-kit adds it to your migrations. `drizzle-orm` (1.0) is now an optional peer dependency.

### Patch Changes

- ce1fc29: Leaner runtime checks. `createRegistry` still refuses an `expose: true` function without `authorize` or `output`, naming it. The checks that only repeated type errors are gone: an async policy is now simply denied (a policy must return exactly `true`). Config and event payloads share one plain-data check, so their errors read the same: "config isn't plain data at app.launch: a Date".
- e5806dc: `query`, `mutation` and `action` have one typed signature instead of five overloads. Same rules, clearer mistakes: `expose: true` without `output` now says "Property 'output' is missing" instead of "No overload matches this call", and a wrong return type points at the handler. Type-checking cost is unchanged (within 10%).
- 3d50881: `waitUntil` docs: answering before listeners finish on a long-lived server is only safe with the outbox. With `inProcess()`, work still running at a restart is lost.
- Updated dependencies [e5806dc]
- Updated dependencies [4d63b2f]
- Updated dependencies [e5806dc]
  - @logic-sdk/cli@0.5.0

## 0.4.0

### Minor Changes

- 61ee14d: Functions are declared with `input`, `output` and `handler(ctx, input)`, the names app-logic libraries use (Convex, oRPC, TanStack Start), instead of the AI SDK's `inputSchema`, `outputSchema` and `execute(input, ctx)`. `ctx` comes first everywhere: handlers, listeners (`on(() => event, async (ctx, payload) => …)`) and `tx.stub(fn, async (ctx, input) => …)`. Events declare `payload` instead of `schema`.

  Ctx types come straight from your config: `type QueryCtx = QueryCtx<Config>`, with the scope taken from `Config`.

  A function that takes nothing leaves out `input`: its handler is `handler(ctx)` and callers write `run(fn)` instead of `run(fn, undefined)`. `toAITool` publishes such a function as a tool with an empty object schema.

  The CLI reads `handler`, finds `ctx` as its first parameter, and `show` prints `input none` for functions without input.

  Migrating: rename `inputSchema` to `input`, `outputSchema` to `output`, `execute` to `handler` and swap its two parameters; swap the parameters of listeners and stubs; rename `schema` to `payload` in `event({...})`; drop `inputSchema: z.undefined()` and the `undefined` in `run(fn, undefined)`.

### Patch Changes

- Updated dependencies [61ee14d]
  - @logic-sdk/cli@0.4.0

## 0.3.0

### Minor Changes

- fcaaf47: Config, and per-call options on `run`. Breaking:

  - New `ctx.config`: declare its shape in `init<Config>()` (`config: { orders: { maxItems: number } }`), fill it at the entry with `createRegistry({ config })`, and read it in queries, mutations, actions and listeners. Plain data only, copied and frozen. Tests override any part of it: `createTestRegistry(registry, { config: { orders: { maxItems: 1 } } })`.
  - `run(fn, input, { scope, signal })` holds what's specific to one call. To act as someone else: `ctx.run(fn, input, { scope: { actor: system } })`.
  - Removed: `ctx.with(...)` and `caller.with(...)` (use `run`'s third argument), and the second argument of `registry.as(scope, options)`: it takes only the scope.
  - `ctx.signal` exists only on actions. A signal is checked before the call you pass it to, never between steps.
  - Adapters pass their signal on: the fetch handler uses `request.signal`, oRPC its `signal`, AI tools their `abortSignal`. `callerOptions` in `toFetchHandler` is gone.
  - CLI: new `env-in-module` rule (module code reading `process.env`, `import.meta.env`, `Bun.env` or Cloudflare's `env`, directly or through app files it imports). `show` lists the config keys a function reads and which scope keys a run changes (`config` and `scopeChanges` in `--json`).
  - Repository moved to github.com/fprl/logic-sdk.

### Patch Changes

- Updated dependencies [fcaaf47]
  - @logic-sdk/cli@0.3.0

## 0.2.0

### Minor Changes

- 713918a: Authorization, errors and agent tools, reworked. Breaking:

  - `init<Config>()` is a single call. Policies are plain sync functions you import: `authorize: member`.
  - `expose: "staff"` becomes `authorize: staff` plus `expose: true`. `expose: true` requires `authorize` and `outputSchema`. Internal functions may have `authorize` too, and it runs even when your own code calls them.
  - Policies run before input validation and must return exactly `true`. A policy that returns a Promise is an error.
  - Record rules go in `execute`: load through a scoped helper, then `if (!canCancel(actor, order)) throw ctx.errors.deny()`.
  - Errors moved to `ctx.errors`: `throw ctx.errors.reject("not_pending")`.
  - `unauthenticated()` moved to `logic-sdk/fetch`.
  - Bad input on a nested `run` is a plain Error (a bug, 500), no longer a client 400.
  - `toAITools` requires a `filter` (the agent door is opt-in). New `toAITool(caller, fn)` for one function.
  - The fetch handler no longer lists functions on `GET {base}`.
  - Node 24 is supported (`engines` is now `>=24`).
  - CLI: shows `authorize` and `exposed`, lists `can*` checks, adds the `async-can` and `unused-can` rules, and closes boundary holes (path aliases, `#` imports, files outside modules, re-exports, dynamic imports).

### Patch Changes

- Updated dependencies [713918a]
  - @logic-sdk/cli@0.2.0
