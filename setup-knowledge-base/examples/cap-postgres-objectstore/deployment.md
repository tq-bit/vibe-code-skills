# Deployment — Cloud Foundry MTA, Authorization & SAP Build Work Zone

Everything needed to ship this project to Cloud Foundry and integrate it into
SAP Build Work Zone: the MTA structure, service configuration, the XSUAA role
model, the production TypeScript wiring and the platform pitfalls. The MTA
describes a manual `mbt build && cf deploy` — nothing is deployed
automatically from this repository. PostgreSQL plans and the schema deployer:
`database-deploy.md`.

---

## Prerequisites

- A Cloud Foundry space in a BTP subaccount with **entitlements assigned
  before the first deploy** (service creation fails without them):
  `postgresql-db` (plan free / standard / premium), `objectstore` (plan
  standard — paid, billed per 100 GB units), `xsuaa` (application),
  `destination` (lite), `html5-apps-repo` (app-host + app-runtime), plus CF
  runtime quota
- Local tools: Node >= 20, `cf` CLI v8+ with the `multiapps` plugin
  (`cf plugins`), `mbt` (MTA Build Tool), `@sap/cds-dk`, UI5 CLI
- An authenticated `cf login` session — the deploy process never logs in
  itself

## MTA structure (mta.yaml, ID tickets-attachments)

| Module | Purpose |
|---|---|
| `tickets-attachments-srv` | CAP backend from `gen/srv` (npm-ci builder; bound to auth, postgres, objectstore, destination) |
| `tickets-attachments-db-deployer` | `gen/pg` one-shot task (`npm start` → `cds-deploy`) applying the schema after instance creation |
| `tickets-attachments-ui` | `ui5 build preload` + `ui5-task-zipper` → `app/ticket-upload/dist/ticket-upload.zip` |
| `tickets-attachments-app-deployer` | Pushes the UI zip to the HTML5 Application Repository; registers the dynamic `srv-api` destination (ForwardAuthToken) |
| `tickets-attachments-approuter` | `.deploy/app-router/` — single user-facing entry: `/odata/v4/*` → srv-api (xsuaa + CSRF), everything else → html5-apps-repo-rt |
| `tickets-attachments-destinations` | Work Zone content-broker destinations (html5-repository + OAuth2UserTokenExchange, keyed by `sap.cloud.service`) |

Resources created: xsuaa (`application`, descriptor `.deploy/xs-security.json`),
`postgresql-db`/**free** (`.deploy/pg-options-free.json` — intentionally
empty, see `database-deploy.md`), `objectstore`/**standard**, html5-apps-repo
(app-host + app-runtime), destination/lite. All deployment-only assets live in
`.deploy/`; the mta.yaml resource `path` entries point there and `mbt`
packages the files into the `.mtar`.

## Build & deploy

Guided: `npm run cli` menu option 5 (prompts for space + dry run) or
`npm run cli -- deploy [--space <space>] [--dry-run]` (scripts/lib/deploy.ts).
Manual equivalent:

```sh
cf target -s <space>
npx mbt build --filename mta.yaml
cf deploy mta_archives/tickets-attachments_1.0.0.mtar   # ID_version parsed from mta.yaml
```

Build outputs (verified against the produced archive):

- `cds build --production` (run by mbt's before-all) generates `gen/srv` —
  package-lock for the `npm-ci` builder, `tsx` as a **runtime** dependency,
  and the `.ts` handler sources — and `gen/pg` with the `cds-deploy` start
  task and `vcap: { label: postgresql-db }`
- MTA module `properties` become environment variables in the CF manifest
- The UI zip ships with manifest.json at its root; the app-deployer copies it
  into `gen/app/`

## TypeScript handlers in production — REQUIRED

The srv module sets `CDS_TYPESCRIPT: tsx` and
`NODE_OPTIONS: --import tsx/cjs` (mta.yaml properties; `gen/srv`'s own
`npm start` sets them as well). **Deploying without them silently strips all
custom backend behavior**: @sap/cds only resolves `.ts` handler siblings when
`CDS_TYPESCRIPT` is set (source-verified in @sap/cds 10: `bin/serve.js` +
`lib/srv/factory.js`), and the tsx CJS loader is required so `require()` of
`.ts` files with extensionless relative imports works. `cds watch` (cds-dk)
masks this locally via its tsx auto-detection — plain `cds-serve` in CF does
not. Use `npm start` locally; the FAQ has the details.

## Requires / Bindings

`NODE_ENV=production` (CF nodejs buildpack default) activates the
`[production]` profile: `auth` (kind xsuaa), `db` (kind postgres via
`@cap-js/postgres`), `object-store` (kind objectstore). CAP injects a bound
CF service's credentials into `cds.requires.<name>.credentials` when the
requires `kind` matches the VCAP_SERVICES label (documented matching order:
name → binding_name → tags → kind vs. tags → kind vs. label). The object
store binding is consumed through `srv/util/env.ts` (binding-first APP_ENV —
see backend/object-store.md for the credential shapes and the AWS-only
driver, which makes the deployment region's hyperscaler decisive).

## Authorization (roles & XSUAA)

Role model (`.deploy/xs-security.json`): scopes `$XSAPPNAME.admin` and
`$XSAPPNAME.user`, role templates `admin`/`user`, role collections
`tickets-admin` / `tickets-user`.

- **`role-template-references` MUST be prefixed strings** —
  `["$XSAPPNAME.admin"]`. Bare names (`["admin"]`) and object forms
  (`[{ "name": "admin" }]`) are rejected by the xsuaa broker.
- **xsappname rules**: max 100 chars, no dots (letters, digits, `_`, `-`,
  `/` only). This project keys it on the space:
  `tickets-attachments-${space}`; uniqueness is enforced **per subaccount** —
  never deploy into two spaces of the same subaccount simultaneously.
- After deployment, assign the role collections in the BTP cockpit
  (Subaccount → Security → Role Collections → add your user with the default
  IdP) and **log in again** — the old JWT does not carry the new scopes.
- Secrets never go into the MTA: set them after the deploy with
  `cf set-env <srv-app> <NAME> <value>` + `cf restart <srv-app>`. (This
  project has no secret env vars today.)

## SAP Build Work Zone integration

- UI inbound `tickets-create` in `app/ticket-upload/webapp/manifest.json`,
  `sap.cloud.service: com.example.tickets.service` in the manifest and in the
  destination content (html5-repository + OAuth2UserTokenExchange
  destinations)
- The approuter route `^/odata/v4/(.*)$ -> srv-api` serves the app's backend
  calls with xsuaa + CSRF protection (`csrfProtection: true`; the app fetches
  the token via `x-csrf-token: fetch`)
- The app appears automatically in Work Zone's HTML5 Apps content provider;
  assign `tickets-admin` / `tickets-user`, then add the app to a site

## Pitfalls (platform, verified across projects)

1. **`mbt build` mutates local node_modules.** The before-all `npm ci` in the
   project root replaces `node_modules`; worse, the UI module's `npm ci` in
   `app/ticket-upload` (a workspace of the root) prunes the root dependencies
   to the UI-only tree. Recovery after a build: `npm ci` in the project root.
   Open fix proposal: backend/recent-changes.md.
2. **Ghost instance after a failed deploy**: a xsuaa instance stuck in
   `create failed` makes updates fail with 404 — recover via
   `cf delete-service <instance> -f`, then redeploy.
3. **Running deploys stop all apps** — during the `cf deploy` window, expect
   503s; check `cf mta-ops` / `cf apps`.
4. **HTML5 apps are served under the app-ID path**, not `/`:
   `https://<approuter-route>/<app-id-without-dots>/index.html`. A 404/503 on
   `/index.html` is a wrong path, not a broken deployment. The srv route
   answering `GET /` with "Cannot GET /" is normal — the API lives under
   `/odata/v4/ticket`.
5. **Entitlements first** — service creation inside the deployer fails when
   the plan is not entitled on the subaccount.
6. The PostgreSQL free plan has a **120-day lifecycle** — see
   `database-deploy.md` before using this template productively.
