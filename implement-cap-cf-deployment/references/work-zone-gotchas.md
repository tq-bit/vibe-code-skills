# Work Zone / Cloud Foundry deployment gotchas

Companion reference for `implement-cap-cf-deployment`. Distilled from the
proven deployment guide (CAP + UI5 + PostgreSQL + Object Store, Cloud
Foundry + SAP Build Work Zone). Verify these whenever a deployment fails
or a file deviates from its reference.

## HTML5 Runtime — two different things, never confuse them

| | html5-apps-repo instance (plan `app-runtime`) | destination instance with `HTML5Runtime_enabled` |
|---|---|---|
| MTA resource | `…-html5-runtime` (`service: html5-apps-repo`, `service-plan: app-runtime`) | `…-destination` resource with `config.HTML5Runtime_enabled: true` |
| Serves | the STANDALONE approuter (direct CF route) | the WORK ZONE managed approuter (app import + serving) |

- `HTML5Runtime_enabled: true` on the destination instance is **mandatory
  for the Work Zone import** — without it the HTML5 Apps content provider
  can neither import nor serve the app.
- The flag must be present from **creation** (resource `config` block); an
  existing instance created without it must be **deleted and recreated**
  (`cf delete-service <instance> -f` + redeploy) — a config update alone
  does not reliably apply `init_data`.
- The instance-level `srv-api` destination (`HTML5.ForwardAuthToken: true`)
  is required for the managed approuter: it resolves the app's `/odata` and
  `/ws` routes from the destination **instance**, NOT from the dynamic
  app-metadata destination registered by the app-deployer.
- Instance-level destinations need no `ServiceInstanceName`; content-module
  destinations always do.

## Work Zone flow

- `webapp/manifest.json` must carry `sap.cloud: { public: true, service:
  "<sap.cloud.service>" }` — the same value as the `sap.cloud.service` of
  the destination content entries. A mismatch breaks the content provider
  listing.
- The app's own `xs-app.json` is shipped inside the UI5 zip (e.g. via
  ui5-task-zipper `additionalFiles`); it routes `/odata/*` and `/ws/*` to
  destination `srv-api`, everything else is served by `html5-apps-repo-rt`.
- After the deploy: assign the role collections (`<app>-admin` /
  `<app>-user`) to users in the BTP cockpit, then add the app to a Work
  Zone site. Users must re-login — old JWTs lack the new scopes.

## XSUAA / xs-security

- `xsappname` rules: max 100 chars, NO dots (letters, digits, `_`, `-`, `/`),
  unique per subaccount; keyed on the space when the org name does not fit
  the limit. Two MTA variants must never run in different spaces of the
  same subaccount at the same time.
- `role-template-references` MUST be prefixed strings:
  `["$XSAPPNAME.admin"]` — bare names (`["admin"]`) are rejected by the
  broker, object forms (`[{ "name": "admin" }]`) crash the sibling resolver.
- Explicit `grant-types` list (authorization_code, client_credentials,
  refresh_token, user_token): a partial list silently disables the
  respective flow (approuter login / technical clients / Work Zone token
  exchange). No list = broker default (all).
- `redirect-uris` + the `requires: app-api` dependency belong on the xsuaa
  resource ONLY when a standalone approuter module exists (its login needs
  them); pure Work Zone deployments don't.
- `$XSSERVICENAME(<mta-id>-jobscheduler)` must match the MTA resource name
  exactly — watch ID prefixes of parallel stacks. Without a Job Scheduler
  resource, use a security descriptor WITHOUT the scheduler scope (the
  placeholder would fail instance creation).
- Top-level `authorities: ["$XSAPPNAME.integration"]` grants the technical
  client scope to the instance's own client-credentials tokens (service-key
  clients) — no role collection, no user assignment; the template stays out
  of the role collections on purpose.
- Ghost instance after a failed deploy: an xsuaa instance stuck in
  `create failed` makes updates fail with 404 — recovery:
  `cf delete-service <instance> -f`, then redeploy.

## Parallel stacks (ID prefix)

- A prefix on all module/resource IDs creates a parallel stack with its own
  instances; the old MTA stays untouched.
- Exception: the html5-apps-repo app-host is BOUND, not created — the app
  ID is unique per subaccount, a parallel app-host fails with CODE 1001
  ("application metadata ... already exists"). The app-deployer updates the
  existing app in place.
- `ServiceInstanceName` in the destinations content module is the REAL CF
  instance name (resolved against the space), NOT the MTA resource name.

## WebSockets

- AppRouter (standalone AND Work Zone managed) only forwards WebSocket
  upgrades when `"websockets": { "enabled": true }` is set in the respective
  xs-app.json — both the standalone router's file and the app's own file
  shipped in the UI5 zip. Missing flag: no HTTP 101 on
  `wss://<host>/ws/...`, client degrades to polling.

## PostgreSQL / Object Store / deploy mechanics

- `standard` plan: `memory` only 2 or 4 GB (8+ is `premium` and rejected);
  `storage` is immutable after creation; `public_access: false` (private —
  destructive schema recovery via CF task requires tunneling).
- Free plan: create WITHOUT a config body; 120-day lifecycle; downgrade
  standard → free impossible, upgrade anytime without data loss.
- The postgres-deployer INSERTs the `db/data` CSVs on EVERY run — remove
  the module when the database is already initialized (duplicates).
- The deployer refuses destructive schema deltas ("Dropping tables is not
  supported") — drop incompatible legacy tables via CF task, then full
  redeploy.
- `mbt build` runs `npm ci` in the project root (replaces `node_modules`)
  — run `npm install` afterwards if local dev tools are missing.
- A deploy in progress stops all apps (503 during the window is expected) —
  check `cf mta-ops` / `cf apps`.
- App URL: HTML5 apps live under the app-ID path (app ID from
  `webapp/manifest.json` `sap.app/id`, special characters removed) —
  `https://<approuter-route>/<app-id>/index.html`; a 503 on `/index.html`
  means wrong path, not a broken deploy. `GET /` on the backend answers
  "Cannot GET /" (normal — the API lives at `/odata/v4/<service>`).
- Secrets (API keys, passwords) NEVER go into the MTA:
  `cf set-env <srv-app> <SECRET_ENV> <value> && cf restart <srv-app>`.
