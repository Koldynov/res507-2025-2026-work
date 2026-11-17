# Lab 30 – CI pipeline notes

Baseline green run (commit a453ac8): https://github.com/Koldynov/res507-2025-2026-work/actions/runs/38216329588
Pull request run (PR #1): https://github.com/Koldynov/res507-2025-2026-work/actions/runs/38218981460

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

## Extension: Lane C – fast checks separated from container validation
- `fast-checks` job: npm ci + JavaScript syntax checks.
- `container` job: Docker build + /health smoke test, with `needs: fast-checks`.
- Each job has its own checkout and Node setup, because jobs run on separate
  runners and share no workspace, dependencies or images.

Evidence:
- Split pipeline green (commit f1dfc72): https://github.com/Koldynov/res507-2025-2026-work/actions/runs/38218592016
- Syntax error (commit 0b73402): fast-checks failed with a SyntaxError in
  app.js, container job was skipped: https://github.com/Koldynov/res507-2025-2026-work/actions/runs/38218776078
- Fix (commit ac21614), both jobs green on push: https://github.com/Koldynov/res507-2025-2026-work/actions/runs/38218975333
- Same fix, pull_request run: https://github.com/Koldynov/res507-2025-2026-work/actions/runs/38218981460

Why the container job must not report success when fast-checks fails:
a green container result would suggest the image was validated, while the
code does not even parse. "Skipped" shows honestly that validation did not
run, and it avoids wasting time building a known-broken image.
