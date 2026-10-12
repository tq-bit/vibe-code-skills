# UI app (app/ticket-upload)

One-page UI5 freestyle TypeScript app (`com.example.tickets`) — no routing:
the component loads `view/App` as the `rootView` (a router with a
`controlId` target cannot resolve its own shell view and fails with
"Control with ID app could not be found"). Built with `@ui5/cli` +
`ui5-tooling-transpile` and zipped via `ui5-task-zipper` for the HTML5
Application Repository (`npm run build` inside `app/ticket-upload`).

- `webapp/manifest.json` — OData v4 model on `/odata/v4/ticket/`,
  `sap.cloud.service: com.example.tickets.service`, FLP/Work Zone inbound
  `tickets-create`.
- `webapp/controller/App.controller.ts` — upload flow: FileReader
  (readAsDataURL, prefix stripped) -> base64 deep insert
  `{title, description, attachments:[{fileName, mimeType, size, content}]}`.
  Creation goes through the tickets table's OData v4 list binding
  (`listBinding.create(...)` + `await context.created()`) — the row appears
  in the table without a refresh and the model handles the CSRF token
  exchange itself.
- Delete flow (2026-10-09): an Actions column holds a delete button per
  ticket row, and every attachment link sits next to a small delete button
  in the attachment cell. Both handlers confirm via
  `MessageBox.confirm(text, { onClose })` (no Promise — see `rules.md`) and
  then delegate to the shared `deleteEntity` helper:
  `await context.delete()` + success/failure toast; attachment deletes also
  refresh the tickets list binding to re-pull the expanded attachments.
- `webapp/controller/BaseController.ts` — `getAppResourceBundleText` helper;
  all display texts come from `webapp/i18n/i18n.properties`.
- Ticket list: `sap.m.Table` bound to `/Tickets` with
  `$select` + `$expand=attachments($select=...)`; attachment names render
  as links (click downloads the base64 content as a Blob).
- Local dev: `ui5 serve` (fiori-tools-proxy proxies `/odata` to
  `localhost:4004`) OR the CAP server itself — `cds-plugin-ui5` (devDep)
  mounts the app at `/com.example.tickets/index.html` during `npm start`/
  `cds watch`. Opening `webapp/index.html` via file:// fails with a
  "script load error" for `Component.js`.
- `xs-app.json` is zipped into the app (ui5-task-zipper `additionalFiles`)
  for the HTML5 repo runtime routing.
- After webapp changes, `npm run build` in `app/ticket-upload` refreshes
  `dist/` and `dist/ticket-upload.zip` (the deployable artifact); the browser
  may cache the old transpiled controller — hard-refresh after a restart.
