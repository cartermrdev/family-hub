# Host on Cloudflare Workers with D1 and a Household Durable Object

Family Hub runs entirely on Cloudflare Workers Paid (~$5/mo): the React SPA as static assets, a Hono API Worker, D1 (via Drizzle) as the single source of truth, one Household Durable Object that holds every screen's WebSocket and broadcasts invalidation signals, and a once-a-minute Cron Trigger for CalDAV polling, Chores falling due, and weather. We chose this over an always-on Node + SQLite server (e.g. Fly.io) because it fits the $5 budget fully managed, the Durable Object maps exactly onto "one Household, live updates to every screen", and cron replaces a job runner.

## Consequences

- No Node-only libraries on the server: tsdav (Workers-supported) for CalDAV, `@block65/webcrypto-web-push` instead of `web-push`.
- Whether Workers' fetch can send CalDAV's PROPFIND/REPORT, and the CPU cost of a sync, is unverified until the CalDAV spike runs from a deployed Worker; if it fails, the fallback is a single Node + SQLite container on Fly.io with the same API shape.
