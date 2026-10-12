# Lab 30 – CI failure diagnosis

## Deliberate failure
- Failed run: https://github.com/Koldynov/res507-2025-2026-work/actions/runs/3821# Lab 30 – CI failure diagnosis

## Deliberate failure
- Failed run: https://github.com/Koldynov/res507-2025-2026-work/actions/runs/38217134379
- Breaking commit: 62017f8
- Failed step: "Run container smoke test"
- Cause: the health check requested `/healthz`, a route that does not exist.
  The app returned HTTP 404, so `curl --fail` failed on every attempt and the
  loop ended with "Health check failed".

## Recovery
- Fixing commit: 464d8d0
- Green run: https://github.com/Koldynov/res507-2025-2026-work/actions/runs/38217399107
- Why it fixes it: the check now requests `/health`, which exists and returns
  `{"ok": true}`, so the assertion passes. The check itself was not removed
  or weakened.

## What green CI does NOT prove
The smoke test runs the container without PostgreSQL, and `/health` does not
query the database. CI can be green while `/` fails to load quotes or
`/quotes` fails to insert (e.g. a typo in a SQL query).
- Breaking commit: 62017f8
- Failed step: "Run container smoke test"
- Cause: the health check requested `/healthz`, a route that does not exist.
  The app returned HTTP 404, so `curl --fail` failed on every attempt and the
  loop ended with "Health check failed".

## Recovery
- Fixing commit: 464d8d0
- Green run: https://github.com/Koldynov/res507-2025-2026-work/actions/runs/...
- Why it fixes it: the check now requests `/health`, which exists and returns
  `{"ok": true}`, so the assertion passes. The check itself was not removed
  or weakened.

## What green CI does NOT prove
The smoke test runs the container without PostgreSQL, and `/health` does not
query the database. CI can be green while `/` fails to load quotes or
`/quotes` fails to insert (e.g. a typo in a SQL query).
