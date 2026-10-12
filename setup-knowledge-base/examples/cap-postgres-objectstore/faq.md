# FAQ

**How do I write an attachment?**
Send base64 in the `content` property (JSON) — directly on `TicketAttachments` or as a deep insert in `attachments` on a ticket. The handlers store it in the object store keyed by the attachment ID.

**How do I read an attachment?**
Read the attachment (or `$expand=attachments`) and select `content` — the handlers restore the base64 string from the object store.

**Why are my TypeScript handlers not running?**
`.ts` service impls only bind when `CDS_TYPESCRIPT` is set. `npm start` sets `CDS_TYPESCRIPT=tsx` and registers the tsx loader via `NODE_OPTIONS='--import tsx/cjs'`. Without them the service still serves generic CRUD — handlers are silently inactive.

**Why did CDS_TYPESCRIPT alone not work?**
`"type": "module"` in package.json makes Node load `.ts` as ESM, which rejects extensionless relative imports (`ERR_MODULE_NOT_FOUND`). The tsx CJS loader resolves them. The package is CJS (no `type` field), matching the loader.

**Where is attachment content stored?**
In the object store: AWS S3 bucket when `STORE_*` is configured (or the BTP objectstore binding in production), otherwise the local development driver below `.object-store/`.

**What happens when the object store fails?**
Writes and deletes reject with 502 — the row is kept so the client can retry. Failing content restores leave `content` null and log a warning.

**How do I run against PostgreSQL?**
Set `NODE_ENV=production` with a provisioned instance's credentials in the environment — `cds.requires['[production]'].db` switches to `@cap-js/postgres`. The `mta.yaml` provisions the instance for a Cloud Foundry deployment; nothing in this repo deploys anything on its own.

**Does the UI need a CSRF token?**
Yes in production — the approuter route for `/odata/v4/*` has `csrfProtection: true`, and the app fetches the token via `x-csrf-token: fetch`. Against plain CAP (local development) no token is returned and none is required; the empty token is accepted.

**Why does the UI fail with "failed to load com/example/tickets/Component.js: script load error"?**
The page was opened from a location that cannot serve the component scripts — typically `webapp/index.html` via file:// or a server without the UI5 middlewares. Open the app through a served origin: `http://localhost:4004/com.example.tickets/index.html` (CAP + cds-plugin-ui5) or `http://localhost:8080` (`ui5 serve` inside `app/ticket-upload`).

**How does the app appear in SAP Work Zone?**
The UI is deployed into the HTML5 Application Repository with the `tickets-create` FLP inbound and destination content (`sap.cloud.service: com.example.tickets.service`); after deployment add it to a Work Zone site from the content manager.

**Why is the PostgreSQL plan free?**
The template targets a zero-cost default: the `postgresql-db` free plan has fixed resources (`.deploy/pg-options-free.json` is empty because the plan accepts no sizing parameters) and a 120-day inactivity lifecycle. Switch `tickets-attachments-postgres` to `service-plan: standard` for productive use.

**Why does `mbt build` break my local `npm start`?**
Two `npm ci` side effects: the before-all replaces the root `node_modules`, and the UI module's `npm ci` inside `app/ticket-upload` (a workspace of the root) prunes the root dependencies to the UI-only tree — afterwards `cds-serve` is gone. Recovery: `npm ci` in the project root. Details: [deployment.md](deployment.md) and [backend/recent-changes.md](backend/recent-changes.md).

**Why does nothing happen when I press OK in a MessageBox confirmation?**
`sap.m.MessageBox.confirm` returns `undefined` — it does not return a Promise, so `await MessageBox.confirm(...)` resolves to `undefined` and any action check fails silently. Use the `onClose` callback. See [frontend/rules.md](frontend/rules.md).
