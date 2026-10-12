# Frontend: Development Rules (`app/ticket-upload/webapp/`)

Mandatory rules for any UI5/Fiori work in `app/ticket-upload/webapp/`.
Companion files in `.vibe/knowledge/frontend/`: `ticket-upload.md`,
`recent-changes.md`.

---

## UI5/Fiori Development Rules

### MCP Server Usage

For ANY question, task, explanation, or code change involving SAP UI5,
OpenUI5, SAP Fiori, or @sapui5/@ui5 packages, you MUST first consult the UI5
MCP server (`mcp_ui5`). This requirement is absolute and not discretionary.

This applies to ALL UI5-related topics:

- Control usage (sap.m.*, sap.ui.table.*, unified FileUploader, etc.)
- Model configuration (OData v2/v4, JSONModel)
- Data binding patterns (bindList, bindContext, expression binding)
- View development (XML, JSON, JS, TS views and fragments)
- Controller implementation (event handlers, lifecycle hooks)
- TypeScript usage in UI5 applications
- i18n and localization
- Any code in `webapp/` directory

Before answering ANY UI5-specific question or making ANY frontend change:

1. Use `get_guidelines` to retrieve the latest UI5 development best practices
2. Use `get_api_reference` to get official API documentation for specific
   controls, methods, or modules
3. Use `run_ui5_linter` to validate code against UI5 best practices
4. Use `run_manifest_validation` to validate manifest.json files
5. Cite the specific documentation retrieved in your answer
6. Mark any claims as "Unverified" if documentation cannot confirm them

Only after retrieving and citing MCP server documentation may you provide an
answer or make changes. If the MCP server is unavailable, you must explicitly
state this limitation and request permission to proceed.

### OData v4 Service Configuration

**Service endpoint:** `/odata/v4/ticket/` (CAP OData v4). Nested expands use
the `attachments($select=...)` pattern in the binding `parameters`.

**Verified API facts (UI5 1.153, checked against the shipped runtime source):**

- `sap.m.MessageBox.confirm(message, options)` returns **undefined** — it
  does not return a Promise. `await MessageBox.confirm(...)` resolves to
  `undefined` and the pressed action is only delivered via the
  `options.onClose` callback.
- `sap.ui.model.odata.v4.Context#delete()` exists and its promise resolves
  when the server confirmed the deletion; there is **no `deleted()` method**
  on a v4 Context (`created()` exists for `listBinding.create(...)` contexts).
  After a successful `delete()` the context is destroyed and must not be used.
- Entity creation goes through the list binding:
  `listBinding.create(payload)` + `await createdContext.created()` — the row
  appears in the table without a manual refresh and the model handles the
  CSRF token exchange.

### Element ID Management

**MANDATORY:** before creating any new UI5 control/element:

1. **Check for duplicate IDs**: search the codebase with
   `grep -r "id=\"<elementId>\""`
2. **ID naming convention**: camelCase matching the existing IDs
   (`titleInput`, `ticketsTable`)
3. **ID scope**: ensure IDs are unique across the entire application

### View Development Guidelines

- Follow the MVC pattern with views in XML and controllers in TypeScript
- Use OData v4 binding patterns (bindList, bindContext)
- All user-visible texts come from `webapp/i18n/i18n.properties` — key
  convention `<View>.<UiControl>.<FirstThreeWords>` (enforced by the
  `implement-ui5-i18n-conventions` skill; shared keys use
  `App.Shared.*`)
- Add corresponding i18n translations for every new text element in the same
  change

---

## Deployment

The UI ships via the MTA into the HTML5 Application Repository — in production
it is NOT served by the CAP server (`cds-plugin-ui5` is dev-only). The
approuter/Work Zone wiring is documented in **../deployment.md**.
