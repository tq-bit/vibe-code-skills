---
name: implement-cap-jobs
description: Load when the user explicitly mentions the SAP Job Scheduler, a JobService, or asks to expose, migrate, or implement recurring background/maintenance jobs as externally callable OData actions in a SAP CAP project.
---

# Implement CAP Jobs (SAP Job Scheduler Pattern)

Recurring maintenance tasks in a CAP project must be implemented as unbound
OData actions of a dedicated job service, so an external scheduler (SAP Job
Scheduler) owns the cadence. Do NOT use `cds.spawn` or any internal timer for
scheduling — internal scheduling next to an external scheduler causes double
execution.

## Service definition

- Declare the job service in its own folder, e.g. `srv/jobs/JobsApi.cds`
- One unbound action per maintenance task, named after the task
  (`action archiveDocumentContents();`) — no parameters, no return value
  unless the caller needs a result
- Register it in the model entry point (`srv/index.cds`) via
  `using from './jobs/JobsApi';` — a service not reachable from the entry
  point is not part of the compiled model and gets no endpoint
- Keep the actual job logic in the existing handler/service modules; the job
  service only delegates to them

## Implementation

Sibling file next to the `.cds` file (`srv/jobs/JobsApi.ts`), with a
default-exported handler receiving the CAP `Service`:

- One `srv.on('<actionName>', ...)` per action; the action name is available
  as `req.event`
- A shared runner helper should: log the run start, execute the task, log
  the processed count when > 0, and catch unexpected errors with
  `req.reject(500, ...)` so the scheduler records a FAILED run
- Per-item failures (one document, one extraction) stay inside the delegated
  logic modules: log, skip, retry on the next scheduled run — never let them
  abort the whole action

## Transaction behavior (Postgres/production)

- The action handlers run inside the HTTP request's auto-managed transaction
- Bare `SELECT`/`UPDATE`/`DELETE` (via `cds.ql` without an explicit `cds.tx`)
  join that ambient transaction: one pooled DB connection is pinned and row
  locks persist for the ENTIRE action run, including remote calls
- When the task makes long remote calls (AI APIs, object stores), wrap each
  unit of work in its own short `cds.tx(async () => {...})` so no
  transaction/connection is held across them
- Reads never block (MVCC), but concurrent writes to the same rows wait for
  the action's commit — keep comments about transactions truthful

## Local development (no scheduler available)

Provide a dev-only sidecar script that replaces the scheduler locally:

- One `setInterval` per action sending `POST <service-path>/<actionName>`
- Intervals from environment variables (seconds) with sane defaults, read
  through the project's env utility
- Overlap guard: skip the tick when the previous request is still pending
  (a slow run must not pile up)
- Request failures are logged and retried on the next interval — the sidecar
  never dies
- Refuse to start when `NODE_ENV=production`
- Clear all timers on SIGINT/SIGTERM
- Run it via `node --env-file-if-exists=.env --import tsx <sidecar.ts>`
  (tsx alone does not load `.env`)

## Verification

Before declaring done:

1. Consult the CAP MCP server (`@cap-js/mcp-server` `search_docs` /
   `search_model`) when available — verify action/handler semantics against
   documented behavior
2. `cds compile <entry>` — the service and all actions must appear in the
   model
3. Typecheck (`tsc --noEmit` or the project's equivalent)
4. Dispatch each action: in-process (`cds.serve('all')`, then
   `srv.send({ event: '<actionName>' })`) or HTTP (`POST /odata/v4/job/<actionName>`,
   expect 204 for a void action, 500 means a genuinely failed run)
5. Note: `.ts` sibling implementations only bind when `CDS_TYPESCRIPT` is
   set — without it every action returns 501 "has no handler"

## Authorization

Job actions are privileged operations (they delete, archive, and mutate
bulk data). If the job service has no `@requires` restriction, flag it to
the user and ask before changing access control yourself.
