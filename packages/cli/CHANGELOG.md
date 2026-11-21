# @logic-sdk/cli

## 0.5.0

### Minor Changes

- e5806dc: Functions, listeners and events no longer take a `description`. Write one sentence of JSDoc above each export instead: editors show it on hover, and `logic-sdk list` and `show` print its first paragraph.

  What an AI tool reads now lives in meta, owned by the adapter. `logic-sdk/ai` exports `AIMeta` (`{ ai?: { tool?: boolean; description?: string } }`) for your meta shape: `init<{ meta: AIMeta & AppMeta }>()`. `toAITools(registry, caller)` takes the exposed functions marked `meta: { ai: { tool: true } }` by default (`filter` is now optional), and a new `describe` option picks descriptions from your own meta. `toAITool(caller, fn, { description })` overrides one tool. Order: the call's option, then `meta.ai.description`, else none.

  Migrate: move each `description: "…"` into a `/** … */` above the export; for AI tools, put the text in `meta: { ai: { description } }` and replace `filter: (f) => f.fn.meta?.agent === true` with `meta: { ai: { tool: true } }`.

  The `logic-sdk` package now ships its agent skill at `skills/logic-sdk/SKILL.md`.

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

### Patch Changes

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

- Updated dependencies [9fbf72b]
- Updated dependencies [b64b8aa]
- Updated dependencies [ce1fc29]
- Updated dependencies [2c1c7eb]
- Updated dependencies [e5806dc]
- Updated dependencies [4d63b2f]
- Updated dependencies [e5806dc]
- Updated dependencies [e5806dc]
- Updated dependencies [09466b5]
- Updated dependencies [3d50881]
  - logic-sdk@0.5.0

## 0.4.0

### Minor Changes

- 61ee14d: Functions are declared with `input`, `output` and `handler(ctx, input)`, the names app-logic libraries use (Convex, oRPC, TanStack Start), instead of the AI SDK's `inputSchema`, `outputSchema` and `execute(input, ctx)`. `ctx` comes first everywhere: handlers, listeners (`on(() => event, async (ctx, payload) => …)`) and `tx.stub(fn, async (ctx, input) => …)`. Events declare `payload` instead of `schema`.

  Ctx types come straight from your config: `type QueryCtx = QueryCtx<Config>`, with the scope taken from `Config`.

  A function that takes nothing leaves out `input`: its handler is `handler(ctx)` and callers write `run(fn)` instead of `run(fn, undefined)`. `toAITool` publishes such a function as a tool with an empty object schema.

  The CLI reads `handler`, finds `ctx` as its first parameter, and `show` prints `input none` for functions without input.

  Migrating: rename `inputSchema` to `input`, `outputSchema` to `output`, `execute` to `handler` and swap its two parameters; swap the parameters of listeners and stubs; rename `schema` to `payload` in `event({...})`; drop `inputSchema: z.undefined()` and the `undefined` in `run(fn, undefined)`.

### Patch Changes

- Updated dependencies [61ee14d]
  - logic-sdk@0.4.0

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
  - logic-sdk@0.3.0

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
  - logic-sdk@0.2.0
