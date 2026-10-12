# AGENTS.md — cds-attachments-demo

Project instructions for agents working in this repository. A minimal SAP CAP
template: one service, one entity — attachments implemented by hand (no
attachment plugin) on top of a provider-agnostic object store, with a
one-page UI5 upload app and a Cloud Foundry MTA.

## Knowledge Directory

Domain knowledge is maintained in separate files inside `.vibe/knowledge/`.
Read the file matching your task's domain before answering questions or
making changes:

| File | Scope | Read before working on |
|------|-------|------------------------|
| `.vibe/knowledge/index.md` | Entry point: mermaid mindmap of the knowledge structure, a flowchart of the cross-references between all knowledge files, and a table of when to read each | Start of any knowledge-driven task |
| `.vibe/knowledge/backend/rules.md` | Backend rules: CAP MCP server consultation, verification requirements, change guidelines, deployment pointers | Any file in `db/`, `srv/`, `scripts/` |
| `.vibe/knowledge/backend/ticket-service.md` | TicketService handlers and the attachment content lifecycle (store, restore, delete) | Any task touching `srv/ticket-service.cds` or `srv/ticket-service.ts` |
| `.vibe/knowledge/backend/object-store.md` | Object store factory, drivers and the BTP objectstore binding facts | Any task touching `srv/store/`, `srv/util/env.ts` or the objectstore binding |
| `.vibe/knowledge/backend/cli.md` | Dev CLI: interactive menu and commands (verify, config, list, delete-all, deploy) | Any task touching `scripts/` |
| `.vibe/knowledge/backend/recent-changes.md` | Backend changes not yet documented in the other files (open debt) | Backend work context |
| `.vibe/knowledge/frontend/rules.md` | Frontend rules: UI5 MCP server consultation, verified OData v4 API facts, element ID and i18n guidelines | Any file in `app/ticket-upload/webapp/` |
| `.vibe/knowledge/frontend/ticket-upload.md` | App structure, upload and delete flows, local dev servers | Any task touching the views, controllers, i18n or manifest |
| `.vibe/knowledge/frontend/recent-changes.md` | Frontend changes not yet documented in the other files | Frontend work context |
| `.vibe/knowledge/entities.md` | Domain model: every entity with its properties and associations | Any task requiring the project's entities |
| `.vibe/knowledge/deployment.md` | Cloud Foundry MTA, prerequisites, TypeScript production wiring, XSUAA roles, Work Zone, platform pitfalls | Any deployment, mta.yaml, `.deploy/`, binding or production task |
| `.vibe/knowledge/database-deploy.md` | PostgreSQL service plans, free-tier lifecycle, instance parameters, schema deployer behavior | Any task changing the database plan, options or schema deployment |
| `.vibe/knowledge/faq.md` | Brief questions and brief answers | Quick lookups; extend with new brief Q&As instead of long-form knowledge sections |

The domain rules files (`.vibe/knowledge/backend/rules.md`,
`.vibe/knowledge/frontend/rules.md`) contain the mandatory MCP server
consultation rules:

- **Backend (CAP MCP server, `mcp_cds`)**: required for any CAP question or
  change — tools `search_docs`, `search_model`
- **Frontend (UI5 MCP server, `mcp_ui5`)**: required for any UI5 question or
  change — tools `get_guidelines`, `get_api_reference`, `run_ui5_linter`,
  `run_manifest_validation`
- Hint: Non-trivial means either: 1. A new file is created OR 2. The planned
  change consists of 50 or more LoC OR 3. Two or more files are changed

If the relevant MCP server is unavailable, state this limitation explicitly
and request permission to proceed.

## Maintenance

Extend the knowledge files (not this file) whenever:

- a new service, entity or service binding is added → the matching file in
  `.vibe/knowledge/backend/` (new entity also → `.vibe/knowledge/entities.md`)
- a new UI5 control pattern or view is implemented → the matching domain file
- a new development rule is added or an existing rule is modified → the
  matching domain `rules.md`
- a deployment option, service plan or `.deploy/` asset changes →
  `.vibe/knowledge/deployment.md` (database specifics → `database-deploy.md`)
- low-effort, high-impact findings are made → `.vibe/knowledge/faq.md`

Update the knowledge table in this file and the index in
`.vibe/knowledge/index.md` whenever a knowledge file is added or renamed.
Keep this file limited to general instructions and references to the
knowledge directory.
