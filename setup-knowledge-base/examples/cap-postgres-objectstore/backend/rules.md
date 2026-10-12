# Backend: Development Rules (`db/`, `srv/`, `scripts/`)

Mandatory rules for any CAP backend work in `db/`, `srv/` and `scripts/`.
Companion files in `.vibe/knowledge/backend/`: `ticket-service.md`,
`object-store.md`, `cli.md`, `recent-changes.md`.

---

## SAP CAP Backend Development Rules

### MCP Server Usage

For ANY question, task, explanation, or code change involving SAP CAP, CDS,
@sap/cds, or @cap-js/* packages, you MUST first consult the CAP MCP server
(`mcp_cds`) using `search_docs` or `search_model`. This requirement is
absolute and not discretionary.

This applies to ALL CAP-related topics:

- CDS syntax (entities, types, enums, associations, projections, annotations)
- Service definitions and handlers (before, after, on)
- Event emitting and listening (cds.emit, srv.on)
- Request/Response objects (req.data, req.user, req.reject, req.timestamp)
- Query language (SELECT, UPDATE, DELETE, CQL)
- Transaction handling (cds.tx, cds.run)
- Runtime behavior and version-specific features
- Any code in `db/` or `srv/` directories
- @sap/cds common types (cuid, managed, etc.)
- Service binding resolution (cds.requires, VCAP_SERVICES, kind matching)

Only after retrieving and citing MCP server documentation may you provide an
answer or make changes. If the MCP server is unavailable, you must explicitly
state this limitation and request permission to proceed.

### Verification Requirements

Before answering ANY CAP-specific question or making ANY backend change:

1. Use `search_docs` to verify documented CAP behavior
2. Use `search_model` to inspect the compiled CDS project model when relevant
3. Cite the specific documentation retrieved in your answer
4. Mark any claims as "Unverified" if documentation cannot confirm them

### Change Guidelines

- Keep persistent domain models in `db/` and service definitions, projections,
  and handlers in `srv/`
- Do not edit outside these directories unless specifically requested or
  absolutely necessary
- Use metadata annotations in every service
- Implement service handlers as default-exported functions using `Service` as
  a callback
- Validate input in `before` handlers where appropriate and use
  `req.reject(...)` to propagate errors
- Handle expected errors explicitly; do not use empty catch blocks or broad
  error suppression
- Flag authorization impact and request approval before changing `@requires`,
  `@restrict`, roles, or access-control configuration
- The object-store handlers never write base64 to the database — content goes
  to the store, the row keeps metadata only (see `ticket-service.md`)

---

## Authorization & Production Deployment

Auth is mocked locally (`admin` / `admin` in `package.json` under
`cds.requires.auth`); production uses xsuaa with the role collections
`tickets-admin` / `tickets-user` (`.deploy/xs-security.json`). The full
Cloud Foundry MTA structure, the XSUAA role model, service plans and the
production profile wiring are documented in **../deployment.md**; PostgreSQL
plans and the schema deployer in **../database-deploy.md**.

- `[production]` requires (package.json): `auth` (kind xsuaa), `db`
  (kind postgres via @cap-js/postgres), `object-store` (kind objectstore) —
  CAP injects a bound CF service's credentials into
  `cds.requires.<name>.credentials` when the requires `kind` matches the
  VCAP_SERVICES label
- The object store is consumed through `srv/util/env.ts` (binding-first
  `APP_ENV`, see `object-store.md`) — the factory itself never touches
  `cds.requires`
