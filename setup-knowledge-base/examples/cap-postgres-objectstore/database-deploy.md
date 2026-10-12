# Database Deployment — SAP BTP PostgreSQL Service (`postgresql-db`)

The production database is the **PostgreSQL on SAP BTP, hyperscaler option**
service (CF marketplace name `postgresql-db`), consumed by CAP through
`@cap-js/postgres` (`requires.db`, kind `postgres`). MTA wiring: the
`tickets-attachments-postgres` resource (plan **free**, options
`.deploy/pg-options-free.json`) and the `gen/pg` db-deployer task; overall
deployment: `deployment.md`.

Sources: SAP Help Portal "PostgreSQL on SAP BTP, Hyperscaler Option" (Service
Plans, Sizing, Parameters, Free-Tier Scope of Use) — retrieved 2026-10-07.

---

## Service plans

| Plan | Compute ratio | Memory sizes | Character |
|---|---|---|---|
| **free** | fixed resources | not configurable | No-cost evaluation; 120-day lifecycle; no sizing parameters |
| **standard** | 1 CPU : 2 GB | blocks of 2 GB, sizes 1-2 → **2 or 4 GB only** | Baseline; dev/test and small production |
| **premium** | 1 CPU : 4 GB | blocks of 4 GB, sizes 2/4/8/16 → **8, 16, 32 or 64 GB** | Medium to large production |

Storage (disk): configurable in **blocks of 5 GB** on AWS/GCP; power of 2,
**minimum 32 GB** on Azure. Storage is billed as its own entitlement
(`storage` / `storage_ha`) and **cannot be changed during a plan update** —
only at creation. `multi_az: true` doubles the compute consumption and
requires the HA storage entitlement.

**Plan-sizing pitfall**: `standard` accepts only 2 or 4 GB memory — 8 GB and
up are `premium` sizes and are rejected by the broker. Match `memory` to the
plan before deploying.

## Free-tier plan (this project's default)

- **Fixed resources, no sizing parameters** — create WITHOUT a config body
  (`.deploy/pg-options-free.json` is an empty `{}` for exactly that reason)
- **120-day lifecycle**: 90 days of active availability, then 30 days in a
  backup-only state (bindings/keys deactivated); after 120 days SAP may
  delete the instance permanently — alerts arrive via the Alert Notification
  service before the 90- and 120-day marks
- Upgrade to standard or premium at any time **without data loss**
  (`cf update-service <instance> -p <plan> -c '<params>'`; memory can grow,
  storage cannot change during the update). A downgrade paid → free is not
  supported. Not available on SAP Cloud Infrastructure (SCI) landscapes
- Intended for evaluation/POC — this template's data would be deleted with
  the instance at the end of the lifecycle; switch the mta.yaml resource to
  `service-plan: standard` for productive use

## Instance parameters (create-service config)

Reference for the standard/premium options file (`memory` must match the
plan's block sizes, see above):

| Parameter | Meaning |
|---|---|
| `engine_version` | PostgreSQL major version as a string (e.g. `"16"`) |
| `memory` | RAM in GB — standard 2/4, premium 8/16/32/64 |
| `storage` | Disk in GB — blocks of 5 (AWS/GCP), power of 2 >= 32 (Azure); immutable after creation |
| `multi_az` | High availability flag (2x compute, requires storage_ha) |
| `backup_retention_period` | Automatic backup retention in days |
| `cross_region_backup` | Copy backups to a secondary region |
| `audit_log_level` | pgAudit categories, e.g. `["DDL", "ROLE"]` |
| `maintenance_window` | `{ day_of_week, duration, start_hour_utc }` |
| `db_parameters` | Custom PostgreSQL parameters as name/value pairs |
| `locale` | Instance locale |
| `public_access` / `allow_access` / `ignore_default_ips` | Public IP access control (default false — the instance is reachable only from within CF) |
| `region` | Hyperscaler region of the instance (must match the CF landscape) |

## Schema deployer behavior (`gen/pg`)

The db-deployer module runs `cds-deploy` as a one-shot CF task. It computes a
**delta against the `cds_model` baseline stored in the instance** — it does
not introspect live tables. Consequences:

- A second deploy with an unchanged model is a clean no-op
- The cds-compiler for PostgreSQL **refuses destructive deltas**
  (`Error: Dropping tables is not supported`) — renames/removals require
  manual intervention
- Runbook for an incompatible old schema (instance is private,
  `public_access: false` — every direct DB operation must run as a CF task):
  push a one-off task app (`pg` dependency only, `--no-route`), bind the
  postgres instance, `cf run-task --command "node inspect.js"`, decide
  drop vs. export/restore, drop the app's tables including `cds_model` +
  `cds_outbox_messages` (CASCADE), then redeploy for a clean full deploy
- Gotchas: a task app needs a working `start` script or `--command` to stage;
  `cf push --command` with nested quotes trips the CLI usage parser

## PostgreSQL 63-byte identifier limit

PostgreSQL truncates unquoted identifiers at 63 bytes. A localized entity
whose `<ns>_<Entity>_texts` table truncates to the same 63 bytes as its base
table fails with a misleading `42P07 relation ... already exists` on an empty
database. This project's namespace contributes the prefix
`com_example_tickets_` (21 bytes) — current entities (`Tickets`,
`TicketAttachments`, 28 and 37 bytes total) are far below the limit. When
adding entities, keep base table names well under 63 bytes and check compiled
DDL names for truncated collisions (`localized_...` views truncate too).
