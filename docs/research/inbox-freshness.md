# Research: keeping the in-app inbox fresh

Ticket: #11 (map: #2 "Notifications for Cadence"). Researched 2026-11-12.

**Question:** What options does React Router (framework mode) with SQLite have for keeping a notification bell and unread count up to date? This covers revalidation on navigation, polling with `useRevalidator` or fetchers, Server-Sent Events, and WebSockets, with the cost, complexity and hosting constraints of each.

This file lists facts and trade-offs. It does not make the decision, which belongs to the map's "In-app freshness decision".

## Cadence's current setup (from the repo)

- The ticket says React Router 7, but `package.json` pins **`react-router` / `@react-router/*` 8.3.0**. The newest release is 8.5.1 ([releases](https://github.com/remix-run/react-router/releases)). The docs cited below are the current ("latest") docs.
- `react-router.config.ts` sets `ssr: true` (framework mode with SSR).
- The production server is `react-router-serve ./build/server/index.js`. The repo has no custom server file.
- The database is a single `data.db` opened lazily through better-sqlite3 in WAL mode (`app/db/index.ts`).
- Map constraint: one SQLite file and no background job runner.

## 1. Revalidation on navigation (no extra work)

- In framework mode with SSR, "route loaders are automatically revalidated after all navigations and form submissions (this is different from Data Mode)." ([Route Module: shouldRevalidate](https://reactrouter.com/start/framework/route-module))
- Framework mode with SSR is opt-out: "active loaders are revalidated on navigations and successful submissions (`Link`, `Form`, `fetcher.submit`)". Submissions that fail with 4xx/5xx do not revalidate by default. ([Optimize Revalidation](https://reactrouter.com/how-to/optimize-revalidation))
- If the unread count lives in the `root` loader (or a layout loader), it refreshes on every client navigation and after every mutation, such as "mark as read". `shouldRevalidate` opts out only "**this route's** loader. It does not skip parent or sibling loaders" ([Route Module](https://reactrouter.com/start/framework/route-module)). Other routes opting out will not stop the root count from refreshing.
- Gap: a user who stays on one page (watching a lesson, for example) never sees new notifications until they navigate or submit something.
- **Cost:** one extra `COUNT` query per navigation. **Complexity:** almost none. **Hosting:** none.

## 2. Polling (`useRevalidator` or a fetcher)

- `useRevalidator` exists to "Revalidate the data on the page for reasons outside of normal data mutations like `Window` focus or polling on an interval." It returns `{ revalidate(): Promise<void>, state }`. The docs warn not to use it for normal CRUD, because actions already revalidate. ([useRevalidator](https://reactrouter.com/api/hooks/useRevalidator))
  - `revalidate()` re-runs **all** active loaders on the page, not just the bell's. On a heavy page, every poll repeats all that work.
- A fetcher alternative is `fetcher.load("/api/notifications/unread")` on an interval, pointed at a small route loader. It hits only that loader. "`fetcher.load` only revalidates by default after action submissions and explicit `useRevalidator` calls" ([Optimize Revalidation](https://reactrouter.com/how-to/optimize-revalidation)), so it also refreshes after "mark as read". A `key` option shares one fetcher's state across the app ([useFetcher](https://reactrouter.com/api/hooks/useFetcher)).
  - A fetcher's target needs a loader. A resource route (loader with no default export) works, and the docs recommend returning `data()` from it when fetchers consume it ([Resource Routes](https://reactrouter.com/how-to/resource-routes)).
  - "A plain `fetch()` to a resource route does not go through the router, so it does not revalidate loaders." ([Optimize Revalidation](https://reactrouter.com/how-to/optimize-revalidation))
- Neither API has a built-in interval. You write `setInterval` in an effect yourself; the docs give no polling example.
- Polling in background tabs: the Page Visibility API is meant for exactly this ("An application showing a dashboard of information doesn't want to poll the server for updates when the page isn't visible"). Browsers already throttle `setTimeout` in background tabs (budget-based in Chrome and Firefox). ([MDN: Page Visibility API](https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API)) Usual pattern: pause on `hidden`, refresh at once on `visible`, and optionally refresh on window focus, as in the `useRevalidator` docs example.
- SQLite cheap-check option: `PRAGMA data_version` changes "if changes were committed to the database by any other connection in the interim" but is "unchanged for commits made on the same database connection" ([SQLite pragma docs](https://www.sqlite.org/pragma.html#pragma_data_version)). Cadence runs one process with one shared connection, so this pragma will **not** detect the app's own writes. It cannot replace a per-user query. An indexed `COUNT(*) WHERE recipient = ? AND read_at IS NULL` is the simple check.
- **Cost:** one short HTTP request plus one indexed SQLite read per open tab per interval. With better-sqlite3 (synchronous, in-process) this is cheap at Cadence's scale. Latency equals the interval. **Complexity:** low (one hook plus one route). **Hosting:** none; works with `react-router-serve` as it is, behind any proxy or serverless host.

## 3. Server-Sent Events (SSE)

- Browser side: `new EventSource(url)`. The server answers with `Content-Type: text/event-stream` and UTF-8 messages separated by blank lines. Fields are `event`, `data`, `id` and `retry`, and lines starting with `:` are comments, used as keep-alives. The browser reconnects automatically. ([MDN: Using SSE](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events))
- Limit: over **HTTP/1.1, browsers allow only 6 open connections per browser and domain**, and this is "especially painful when opening multiple tabs" (Chrome and Firefox: "Won't fix"). Over HTTP/2 the limit is negotiated (100 by default). ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)) Each open Cadence tab holds one SSE connection, and those count against the same pool as normal page requests.
- In React Router, a resource route loader can return any `Response` ([Resource Routes](https://reactrouter.com/how-to/resource-routes)). A `ReadableStream` body with `text/event-stream` should therefore work, **but the docs do not cover streaming SSE from a loader**. Verify it in a spike (compression middleware buffering is one known risk: `react-router-serve` enables `compression` by default ([serve docs](https://reactrouter.com/api/other-api/serve))).
- What triggers a push: SQLite has no LISTEN/NOTIFY, and `data_version` does not see same-connection writes (see above). In a single Node process, the action that inserts a notification would have to emit on an in-memory event emitter that open SSE streams subscribe to. This only works while Cadence runs as **one process**. More than one instance needs an external broker. The fallback is each stream polling SQLite on the server, which brings back polling's cost and adds held connections.
- Once a push arrives, the client would still call `revalidate()` or `fetcher.load()` to fetch the new count and list, unless the event carries the data itself.
- **Cost:** one long-lived connection and stream per open tab, plus keep-alive traffic. Latency is near-instant. **Complexity:** medium (stream lifecycle and cleanup on abort, an in-process pub/sub, reconnection and `Last-Event-ID` handling, proxy timeouts and buffering). **Hosting:** needs a long-running Node server (works with `react-router-serve`, untested). Serverless or edge hosts with request time limits are a poor fit. HTTP/2 at the proxy avoids the 6-connection limit.

## 4. WebSockets

- A two-way session "without having to poll the server for a reply". The protocol is RFC 6455. An open WebSocket may keep the page out of the bfcache, so close it when the user leaves. Tabs with open WebSockets "go unthrottled" in the background. `WebSocket` has no backpressure. ([MDN: WebSockets API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API), [MDN: Page Visibility API](https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API))
- React Router has no WebSocket API. Loaders and actions handle HTTP requests, not upgrade requests. `react-router-serve` is "production-ready but basic", built on Express, and "By design, we do not provide options to customize" it. To customize, "migrate to the `@react-router/express` adapter" ([serve docs](https://reactrouter.com/api/other-api/serve)). A WebSocket server (for example the `ws` library listed on MDN) therefore needs Cadence to **own a custom Express/Node server** and change the `start` script, plus matching dev-server wiring.
- The push trigger has the same problem as SSE: no database notifications from SQLite, an in-process emitter in a single process, or a broker for more than one.
- The inbox only needs server-to-client pushes. The extra client-to-server channel buys nothing that a normal action/fetcher submission does not already give.
- **Cost:** one held socket per tab, with low per-message overhead. **Complexity:** highest (custom server, upgrade handling, authenticating the socket from the session cookie, heartbeat and reconnection in the client, dev/prod parity). **Hosting:** long-running server with WebSocket upgrade support through every proxy. No serverless.

## Comparison

| Option | Freshness while idle on a page | New infra | Complexity | Works with current `react-router-serve` | Multi-instance safe |
| --- | --- | --- | --- | --- | --- |
| Navigation revalidation (root loader) | None (until navigate or submit) | None | Very low | Yes | Yes |
| Polling (fetcher or `useRevalidator`) + visibility/focus | Interval (e.g. 30–60 s) | None | Low | Yes | Yes |
| SSE | Near-instant | In-process pub/sub; long-lived connections | Medium | Probably (unverified; watch compression) | No (single process unless a broker is added) |
| WebSockets | Near-instant | Custom Express server + `ws` + pub/sub | High | No | No (single process unless a broker is added) |

## Notes for the decision

- The options stack: navigation revalidation comes free with a count in the root loader. Polling and pushes only cover the "user idles on one page" gap.
- Under the "no background job runner" constraint, notifications are written inside the request that caused the event. So "push" only means telling *other* users' open tabs, and an in-process emitter only works while Cadence runs as one process.
- Open questions a spike would settle: does an SSE `ReadableStream` from a resource route loader flush through `react-router-serve` and its `compression` middleware? What polling interval is acceptable for the product?
