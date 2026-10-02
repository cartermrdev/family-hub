# Host on Cloudflare Workers with D1 and a Household Durable Object

Family Hub runs entirely on Cloudflare Workers Paid (~$5/mo): the React SPA as static assets, a Hono API Worker, D1 (via Drizzle) as the single source of truth, one Household Durable Object that holds every screen's WebSocket and broadcasts invalidation signals, and a once-a-minute Cron Trigger for CalDAV polling, Chores falling due, and weather. We chose this over an always-on Node + SQLite server (e.g. Fly.io) because it fits the $5 budget fully managed, the Durable Object maps exactly onto "one Household, live updates to every screen", and cron replaces a job runner.

## Consequences

- No Node-only libraries on the server: tsdav (Workers-supported) for CalDAV, `@block65/webcrypto-web-push` instead of `web-push`.
- A spike from a deployed Worker (2026-10) confirmed that Workers' fetch can send CalDAV's PROPFIND/REPORT and can create, edit and delete Family Calendar events. An incremental sync costs ~60 ms CPU; a full resync of the 3,429-event calendar costs ~1.3 s, so it runs only on first connect, reconnect or an invalid sync token. That rules out Workers Free (10 ms CPU). The Fly.io fallback (one Node + SQLite container with the same API shape) isn't needed.
