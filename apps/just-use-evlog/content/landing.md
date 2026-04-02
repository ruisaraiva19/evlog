---
title: Just fucking use evlog
description: Wide events and structured errors for TypeScript. One log per operation (request, job, or workflow), all the context, zero scavenger hunt.
ogTitle: Just fucking use evlog
ogDescription: Wide events and structured errors for TypeScript. One log per operation, zero scavenger hunt.
ogHeadline: Stop overthinking your logs
---

<p class="vitrine-eyebrow">Stop overthinking your logs</p>

# Just fucking use evlog.

You've been told to "add more logs" until your stdout looks like a twitch chat. You've opened Sentry at 3am and stared at a stack trace with zero context. You've told a junior "correlate by request id" while knowing half your handlers never set one. That isn't observability. It's hope with a JSON formatter.

**One log per operation. All the context. Zero scavenger hunt.** That's what evlog does. Not ten `INFO` lines that pretend to tell a story. Not "mystery meat" errors where the client sees `500` and the server sees `Error: undefined`. One structured event, with **why** it broke and **what to do next**.

---

## Your logs are a disaster.

Something breaks in prod. You open your log viewer and stare at a wall of events. Hundreds of lines, zero story. You scroll, you filter, you open three tabs trying to reconstruct what happened for *one* request or *one* job run. Half your output is noise ("handler started", "ok", "done"). The other half is missing **user**, **cart**, **flags**, or anything that tells you *what actually broke*.

```bash [Terminal]
$ node server.js
INFO  Starting handler
INFO  user loaded
INFO  db query ok
WARN  slow???
ERROR  Payment failed
ERROR  Error: undefined
INFO  done
```

Seven lines. Zero narrative. You end up in Slack asking "who touched checkout?" while mentally stitching fragments across log entries. **This is the debugging you've normalized.** Fine, but stop pretending scattered `console.log` is "good enough."

Now imagine the same checkout, with evlog:

```json [wide-event.json]
{
  "level": "error",
  "method": "POST",
  "path": "/api/checkout",
  "status": 402,
  "duration": 142,
  "requestId": "req_8x2kf9",
  "user": { "id": "usr_29x8k2", "plan": "pro" },
  "cart": { "items": 3, "total": 9999 },
  "error": {
    "message": "Payment failed",
    "why": "Card declined by issuer",
    "fix": "Try a different payment method"
  }
}
```

One event. The full story. User, cart, error, reason, fix. You open your dashboard, you click the row, you *know* what happened. No stitching, no guessing.

---

## How it works: accumulate, then emit.

You don't build that JSON by hand. You call `log.set()` as your code runs, adding context at each step: auth result, cart state, feature flags, downstream latency, records synced. Whatever matters for *this* operation. At the end, evlog emits **one** event with everything. The **level** reflects outcome. Errors carry **why**, **fix**, and optional **link**, so your frontend (and future you at 3am) stop reverse-engineering stack traces.

---

## What the fuck is evlog, technically?

TypeScript-first logger that works everywhere. In **Nuxt, Next.js, Nitro, Express, Fastify, Hono, Elysia, NestJS, TanStack Start**, framework hooks auto-create and auto-emit the logger at request boundaries. For scripts, jobs, and workflows, you create a logger, accumulate context, emit when done.

::code-group

```ts [Nuxt]
export default defineEventHandler(async (event) => {
  const log = useLogger(event)
  const user = await getUser(event)
  log.set({ user: { id: user.id, plan: user.plan } })

  const cart = await getCart(user.id)
  log.set({ cart: { items: cart.length, total: cart.total } })

  const charge = await processPayment(cart)
  log.set({ payment: { provider: 'stripe', status: charge.status } })

  return { ok: true }
})
```

```ts [Next.js]
export const POST = withEvlog(async (request) => {
  const log = useLogger()
  const { userId } = await request.json()
  log.set({ user: { id: userId } })

  const cart = await getCart(userId)
  log.set({ cart: { items: cart.length, total: cart.total } })

  const charge = await processPayment(cart)
  log.set({ payment: { provider: 'stripe', status: charge.status } })

  return Response.json({ ok: true })
})
```

```ts [Express]
app.post('/api/checkout', async (req, res) => {
  req.log.set({ user: { id: req.body.userId } })

  const cart = await getCart(req.body.userId)
  req.log.set({ cart: { items: cart.length, total: cart.total } })

  const charge = await processPayment(cart)
  req.log.set({ payment: { provider: 'stripe', status: charge.status } })

  res.json({ ok: true })
})
```

```ts [Hono]
app.post('/api/checkout', async (c) => {
  const log = c.get('log')
  const { userId } = await c.req.json()
  log.set({ user: { id: userId } })

  const cart = await getCart(userId)
  log.set({ cart: { items: cart.length, total: cart.total } })

  const charge = await processPayment(cart)
  log.set({ payment: { provider: 'stripe', status: charge.status } })

  return c.json({ ok: true })
})
```

```ts [Fastify]
app.post('/api/checkout', async (request) => {
  request.log.set({ user: { id: request.body.userId } })

  const cart = await getCart(request.body.userId)
  request.log.set({ cart: { items: cart.length, total: cart.total } })

  const charge = await processPayment(cart)
  request.log.set({ payment: { provider: 'stripe', status: charge.status } })

  return { ok: true }
})
```

```json [Result]
{
  "level": "info",
  "method": "POST",
  "path": "/api/checkout",
  "status": 200,
  "duration": 94,
  "requestId": "req_8x2kf9",
  "user": { "id": "usr_29x8k2", "plan": "pro" },
  "cart": { "items": 3, "total": 9999 },
  "payment": { "provider": "stripe", "status": "succeeded" }
}
```

::

Same code pattern, same output, every framework. Human-readable in dev, structured JSON in prod.

---

## Why it's fucking great

### Zero transitive dependencies

No peer deps, no polyfills, no bundler drama. Drains use platform `fetch`. The filesystem drain uses Node's `fs`. Nothing to audit, nothing that breaks on the next Node LTS.

### Emit is automatic (or manual, your call)

In framework mode, hooks call emit at request end. You add context with `log.set()`, the rest is wired. In standalone mode, you call `log.emit()` when the operation is done. Either way: no "remember to flush" surprise.

### Send logs anywhere, out of the box

**Axiom, OTLP** (Grafana, Datadog, Honeycomb…)**, Sentry, PostHog, Better Stack, HyperDX**: six built-in adapters, or two lines of `fetch()` if you roll your own. Sending is async, batched, out-of-band. Your users don't wait on your log pipeline.

### A filesystem drain your agents can read

Write NDJSON to disk locally. Your AI agents, your scripts, and your teammates can query structured context **without a Datadog subscription**. Wide events work for incidents. They also work for evals.

### AI routes stop being a black box

Wrap the model once with `createAILogger`. Token usage, tool calls, streaming metrics, finish reason: all land in the **same** wide event as the HTTP request.

```ts [server/api/chat.post.ts]
const log = useLogger(event)
const ai = createAILogger(log)

const result = streamText({
  model: ai.wrap('anthropic/claude-sonnet-4.6'),
  messages,
})
```

No callback conflicts. No separate pipeline for AI observability.

### Sampling that actually works

Drop 90% of `info` in prod, keep 100% of errors, and force-keep anything slower than 1s or matching `/api/checkout/**`. Head sampling controls volume. Tail sampling catches the events you'd regret losing. Two config blocks, no custom code.

### Errors that explain themselves

`createError({ why, fix, link })`. Parse with `parseError()` on the client. Your error toast can finally tell users *what went wrong* and *what to do about it*.

---

## "But wait…"

### "I already use pino."

pino gives you fast line-by-line JSON. evlog gives you that **plus** wide events, structured errors with `why`/`fix`/`link`, head + tail sampling, six drain adapters, AI SDK integration, and auto-instrumentation for eight frameworks. Zero transitive deps, lighter install, same job done better. pino was the standard. evlog is what comes next.

### "I already have Sentry / Datadog."

Great, they'll get better data. Right now your alert fires and you open a dashboard full of `INFO handler started` lines. With evlog, one wide event lands as a **single queryable row**: user, cart, duration, flags, error, fix. Filter by `status >= 400`, group by `user.plan`, done. Sentry adapter and OTLP adapter are two lines of config each.

### "Another dependency?"

One package, zero transitive deps. The alternative is another quarter of guessing. Your call.

### "We'll 'clean up logging' next sprint."

No you won't. Ship the pattern now or keep debugging the hard way forever.

---

## When you should actually use this

- You write TypeScript (APIs, jobs, scripts, workflows) and you're tired of **piecing together what happened from scattered logs**.
- You want **one artifact per operation** with business context and outcome together.
- You want errors your frontend, and agents, can **parse and act on**.
- You need sampling + drains without building a **second product** inside your repo.
- You're instrumenting AI calls and want usage and tool data in the **same** event as the rest of the operation.

## Enough excuses.

evlog isn't magic. It's **accumulate, emit once, drain safely, keep what matters**. Stop cargo-culting a dozen `console.log` calls per handler and calling it "good enough."

::landing-ctas
::
