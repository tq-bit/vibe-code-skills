# TicketService

Defined in `srv/ticket-service.cds` (projections of `Tickets` and
`TicketAttachments`), implemented in `srv/ticket-service.ts`.

- Served at `/odata/v4/ticket` (CAP 10 default paths).
- Mocked auth configured in `package.json` under `cds.requires.auth` (user `admin` / `admin`).
- SQLite development profile, PostgreSQL production profile (`cds.requires['[production]'].db`).

## Content lifecycle (srv/ticket-service.ts)

The database stores attachment metadata only; `content` (LargeString, base64)
stays null in the DB. The object key is always the attachment's `ID`.

- **Write path** — `persistContent` on CREATE/UPDATE of `TicketAttachments`
  and on deep insert via `Tickets` (child rows bypass the attachment
  handlers): writes the base64 string as UTF-8 bytes to the object store,
  sets `size` to the decoded byte length, nulls `content`.
- **Read path** — `srv.after('READ')` on `TicketAttachments` restores the
  base64 string; expanded rows bypass those handlers, so the restore also
  runs `srv.after('READ')` on `Tickets` over `ticket.attachments`.
- **Delete path** — remote object first, then the row: failing store calls
  reject 502 so the row survives for retry. Ticket deletes remove the
  remote objects explicitly (composition cascade does not run child handlers).
