# Dev CLI (scripts/cli.ts)

`npm run cli` opens an **interactive menu** (scripts/lib/menu.ts) when started
without a command; every menu action is also available as a direct subcommand.
The menu buffers piped input, so `printf '5\n\ny\n6\n' | npm run cli` works:
options 1-5 dispatch the actions below, `q`/`exit`/EOF leave the loop, and a
failing action is caught and re-renders the menu.

| # | Menu entry | Command equivalent |
|---|---|---|
| 1 | Verify connection | `verify` (HeadBucket) |
| 2 | Print current configuration | `config` (masked credential table) |
| 3 | List all files in the current bucket | `list` (ListObjectsV2) |
| 4 | Delete ALL files (typed DELETE confirmation) | `delete-all` (batched DeleteObjects) |
| 5 | Deploy the MTA (prompts for space + dry run) | `deploy` (see ../deployment.md) |
| 6 | Exit | — |

Commander entry (scripts/cli.ts); all actions live in `scripts/lib/`. Talks to
the bucket directly through `@aws-sdk/client-s3` (no imports from `srv/` at
runtime) and reads the `STORE_*` variables from the environment plus a local
`.env` file (`loadEnvFile`). Unlike the service-side factory, the CLI's store
actions require a configured bucket — there is no development-driver fallback
(`STORE_PROVIDER` unset fails with `CONFIG_INCOMPLETE`).

`deploy` (scripts/lib/deploy.ts): `cf target -s <space>` →
`npx mbt build --filename mta.yaml` →
`cf deploy mta_archives/tickets-attachments_<version>.mtar`; the mtar name is
parsed from the MTA's `ID`/`version`. `--dry-run` prints the commands only.

The S3 adapter (`IStorageAdapter` in `@types/scripts.d.ts`) is the extension
point for Azure Blob / GCP Cloud Storage.
