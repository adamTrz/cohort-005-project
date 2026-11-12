# Ways to send notifications from a single-SQLite app

Research for issue #9 (map: #2 "Notifications for Cadence"). This note gives facts and trade-offs only. The decision belongs to a later grilling ticket (#10).

## Context

- Cadence runs on React Router with `better-sqlite3` through Drizzle (`app/db/index.ts`). There is no background job runner. The issue cites ADR 0006 for this, but no `docs/adr/` directory exists in the repo yet.
- Notifications will be **in-app** (a row in SQLite) plus **email** (a call to an HTTP provider).
- React Router's middleware docs describe no built-in background, `waitUntil` or after-response facility. Code after `await next()` still runs before the response returns. ([React Router middleware](https://reactrouter.com/how-to/middleware))

## Constraints that come from SQLite

- **The DB and the email send can't share one transaction.** `better-sqlite3` transaction functions "do not work with async functions": the transaction has already committed by the first `await`. The docs also call holding a transaction open across event-loop ticks "a very bad idea". ([better-sqlite3 API](https://github.com/WiseLibs/better-sqlite3/blob/master/docs/api.md)) So an atomic "write and send" is impossible. Any design either sends after commit (and can lose the send on a crash) or records an intent and sends it later.
- **One writer, one host.** In WAL mode, "there can only be one writer at a time". "All processes using a database must be on the same host computer; WAL does not work over a network filesystem." ([SQLite WAL](https://sqlite.org/wal.html)) A worker that touches the DB has to run on the same machine as the app, either in the app process or as a sibling process.
- **Hosting makes the same-host rule concrete.** On Fly.io, "a volume can be attached to only one Machine". ([Fly volumes](https://docs.fly.io/volumes/overview)) On Render, "Cron jobs can't provision or access a persistent disk". ([Render cron jobs](https://render.com/docs/cronjobs)) A platform-managed cron container therefore cannot open Cadence's SQLite file. It can only call an HTTP endpoint on the app.

## Option 1: send inline in the request

Write the in-app row, commit, then `await` the email API call in the same action.

- **Retries:** none beyond whatever retry loop you write inside the request. A crash or deploy between commit and send loses the email silently.
- **Idempotency:** the provider can dedupe retried calls. Resend accepts an `Idempotency-Key` header (1 to 256 chars) and keeps it for **24 hours**. A duplicate returns the original response. The same key with a different payload returns `409`. ([Resend idempotency](https://resend.com/docs/dashboard/emails/idempotency-keys)) Postmark's Email API and SendGrid's Mail Send API document no idempotency key. ([Postmark](https://postmarkapp.com/developer/api/email-api), [SendGrid](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send))
- **Latency:** the provider round-trip is added to the user's request, and provider downtime fails or slows the action.
- **Local dev:** trivial. Resend has test addresses (`delivered@resend.dev`, `bounced@resend.dev`, `complained@resend.dev`) that simulate outcomes without hurting reputation. They still count against quota. ([Resend test emails](https://resend.com/docs/dashboard/emails/send-test-emails))
- **Hosting:** nothing new.

## Option 2: transactional outbox drained by a worker or cron

In the same SQLite transaction that writes the domain change and the in-app notification, insert an `outbox` row. A separate relay reads pending rows, sends them, and marks them done.

- **Guarantee:** "Messages are guaranteed to be sent if and only if the database transaction commits." ([microservices.io: Transactional outbox](https://microservices.io/patterns/data/transactional-outbox.html)) This works naturally here because the in-app row and the outbox row live in one SQLite file, and the transaction is synchronous.
- **Retries:** you build them. Typically an `attempts` count, a `next_attempt_at` column with backoff, and a dead or failed state.
- **Idempotency:** delivery is at-least-once. The relay can crash after sending and before marking the row, then send again. The pattern says consumers must be idempotent. ([same source](https://microservices.io/patterns/data/transactional-outbox.html)) For email, pass the outbox row id as Resend's `Idempotency-Key`, which covers duplicates within its 24-hour window.
- **Ordering:** the pattern can keep production order. Per-user notification ordering rarely matters.
- **Where the relay can run, given the same-host rule:**
  - **In the app process.** A timer or loop started at server boot. No extra infrastructure, but it stops whenever the app stops. With more than one app process, rows must be claimed with an atomic `UPDATE ... WHERE status='pending'`.
  - **A sibling process or system cron on the same machine/volume.** Possible on a VM. On Fly, each Machine has its own volume, so the process would have to run on the same Machine.
  - **External cron hitting an authenticated HTTP endpoint** (Render cron, GitHub Actions, a hosted cron service). The drain runs inside the app process, and the external scheduler only provides the tick.
- **Local dev:** fully local, with no accounts needed. It's testable with the existing vitest setup by running the drain function directly.
- **Hosting:** none beyond the app, unless you choose the external tick.

## Option 3: hosted queue or workflow service

### Inngest

- **Execution model:** with `serve()`, Inngest invokes functions through an HTTP endpoint mounted in your app (for example `/api/inngest`), so the code **runs in the app process and can use the local SQLite file**. `connect()` is an alternative that uses an outbound WebSocket. There is a Remix adapter (`inngest/remix`). React Router isn't named. ([Serving functions](https://www.inngest.com/docs/learn/serving-inngest-functions))
- **Retries:** "four retries after the first attempt" by default, configurable from 0 to 20 in TS SDK v4, with backoff and jitter. Each `step.run()` has its own retry counter, and completed step results are memoized. ([Retries](https://www.inngest.com/docs/features/inngest-functions/error-retries/retries), [Error handling](https://www.inngest.com/docs/guides/error-handling))
- **Idempotency:** an event `id` deduplicates for 24 hours. A function-level `idempotency` CEL key allows one run per key per 24 hours. Memoization does not guarantee external side effects happen exactly once, so Inngest recommends idempotency keys for messages. ([Idempotency](https://www.inngest.com/docs/guides/handling-idempotency))
- **Atomicity gap:** `inngest.send()` is an HTTP call, so it cannot join the SQLite transaction. Sending the event after commit has the same crash window as Option 1, just moved to the enqueue step. Closing that gap still needs an outbox (or accepting the risk).
- **Local dev:** `npx inngest-cli@latest dev` runs a Dev Server UI/API on `localhost:8288`. Set `INNGEST_DEV=1` in the app. No signing key is needed locally. ([Dev Server](https://www.inngest.com/docs/dev-server))
- **Hosting:** Inngest Cloud, or self-host with `inngest start`. Self-hosting is a single binary with SQLite persistence (`./.inngest/main.db`) and an in-process Redis by default, "does not support scaling beyond a single node" without Postgres, and has no guaranteed support. ([Self-hosting](https://www.inngest.com/docs/self-hosting)) The app must be reachable from Inngest over HTTP, or use `connect()`.

### Trigger.dev

- **Execution model:** deployed tasks are packaged as a Docker image and run on Trigger.dev's infrastructure "in a secure, isolated environment", **not in your app server**. In `trigger dev`, tasks run locally on your machine. ([How it works](https://trigger.dev/docs/how-it-works)) A deployed task therefore **cannot reach Cadence's SQLite file** (see the same-host rule). It would need data passed in the payload, or a callback to the app's API.
- **Retries:** configurable in `trigger.config`. The documented example is `maxAttempts: 3`, `factor: 2`, `minTimeoutInMs: 1000`, `maxTimeoutInMs: 10000`, `randomize: true`, and dev retries are off by default. ([Config file](https://trigger.dev/docs/config/config-file), [Errors & retrying](https://trigger.dev/docs/errors-retrying))
- **Idempotency:** `idempotencyKey` on trigger returns the existing run. The default TTL is **30 days**, and failed runs clear their key. It dedupes triggers, not side effects inside `run()`. ([Idempotency](https://trigger.dev/docs/idempotency))
- **Hosting:** Trigger.dev Cloud, or self-host as a set of containers (webapp with Postgres and Redis, plus a worker) via Docker Compose or Kubernetes. ([Self-hosting](https://trigger.dev/docs/open-source-self-hosting))

### Others (not researched in depth)

Upstash QStash, AWS SQS + Lambda and Cloudflare Queues share Trigger.dev's shape: a hosted broker, plus a consumer that either runs elsewhere (and can't see the SQLite file) or calls back into the app over HTTP. Like Inngest, they have the same enqueue-after-commit gap.

## Option 4: provider-side scheduling

The email provider holds the message and sends it at a future time.

- **Resend:** `scheduled_at` (ISO 8601 or natural language) up to **30 days** ahead. It can be rescheduled (`PATCH /emails/{id}`) or cancelled (`POST /emails/{id}/cancel`), but not over SMTP. A scheduled email can fail if the API key is revoked or the account is under review. ([Resend schedule](https://resend.com/docs/dashboard/emails/schedule-email))
- **SendGrid:** `send_at` Unix timestamp, "cannot be scheduled more than 72 hours in advance". It can be cancelled or paused through a `batch_id`. ([SendGrid Mail Send](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send))
- **Postmark:** the Email API documents no scheduling field. ([Postmark](https://postmarkapp.com/developer/api/email-api))
- **What it solves:** delayed and digest-style timing for **email only**, with retries of the actual delivery handled by the provider. **What it doesn't:** the schedule call is still an inline HTTP request with the same commit-then-call gap, and it does nothing for in-app notifications. Because the in-app row is written directly in the DB, an in-app notification can appear before its email arrives. The 72-hour and 30-day limits rule out long-range reminders without a local scheduler.

## Comparison

| | Inline | Outbox + in-app/cron relay | Inngest | Trigger.dev | Provider scheduling |
|---|---|---|---|---|---|
| Lost on crash after commit | Yes | No | Yes, unless fed from an outbox | Yes, unless fed from an outbox | Yes (schedule call is inline) |
| Retries | DIY in-request | DIY (attempts/backoff columns) | Built in: 4 retries default, per step | Built in, configurable | Provider handles delivery |
| Idempotency | Resend key, 24h | Outbox row id as Resend key | Event id / CEL key, 24h | Trigger key, 30d TTL | Resend key, 24h |
| Can touch the SQLite file | Yes | Yes (same host) | Yes (runs via app endpoint) | No when deployed | n/a |
| Local dev | Trivial | Trivial, no accounts | Dev Server on :8288 | `trigger dev` CLI plus account | Needs provider account |
| Extra hosting | None | None, or an external tick | Cloud account, or self-host binary | Cloud account, or Docker/K8s stack | None |
| Covers in-app too | Yes | Yes (same txn) | Yes | Only via callback | No |

## Open points for the grilling ticket

- Is losing an email on a crash or deploy acceptable for Cadence's notification types? If not, some form of outbox is needed whatever runs the delivery.
- Does Cadence need delayed or digest sends, and over what horizon (the 72h and 30d provider limits)?
- Which email provider? Only Resend among those checked has both an idempotency key and scheduling.
- Where will Cadence be hosted? That decides whether an in-process relay, a same-host cron, or an external HTTP tick is possible.
