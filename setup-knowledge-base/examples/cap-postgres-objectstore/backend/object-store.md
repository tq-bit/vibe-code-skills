# Backend: Object Store (`srv/store/`)

Provider-agnostic read/write/delete access to an S3-compatible bucket, used
by the service handlers for attachment content. Handlers: `ticket-service.md`;
rules: `rules.md`.

- `object-store-factory.ts` — `createObjectStore()`, memoized, never throws:
  falls back to the development driver on unset/unknown/incomplete config.
- `core/aws-driver.ts` — `AwsObjectStore implements IObjectStore<S3Client>`
  (`@aws-sdk/client-s3`); normalizes schemeless BTP binding hosts to
  `https://` with path-style addressing; 404 on read maps to `NOT_FOUND`.
- `core/development-driver.ts` — `DevelopmentObjectStore implements
  IObjectStore<null>`: local filesystem fallback below the root directory
  (default `.object-store/` in the working directory), so the attachment
  flow works end to end without cloud credentials.
- Contract (`@types/object-store.d.ts`): `read` / `write` / `delete` with
  `ObjectStoreError` codes `MISSING_CONFIGURATION`, `NOT_FOUND`,
  `READ_FAILED`, `WRITE_FAILED`, `DELETE_FAILED`.

Configuration precedence (srv/util/env.ts): the BTP objectstore service
binding (`cds.requires['object-store'].credentials`, production) first,
`STORE_*` environment variables (local development, see `.env.example`) as
fallback.

## BTP objectstore service — plan and credential facts (verified 2026-10-08)

- The IaaS-specific plans (`s3-standard`, `azure-standard`, `gcs-standard`)
  were removed on 2024-01-15 (SAP KBA 3501087). The current unified plan is
  **`standard`** — which hyperscaler it provisions depends on the
  subaccount's region IaaS. The plan is paid (billed per 100 GB units).
- `env.ts` derives the provider from the binding credential shape:
  - AWS: `access_key_id`, `secret_access_key`, `bucket`, `region`
    (+ optional `host`, the region-specific S3 endpoint)
  - Azure: `account_name`, `container_name`, `sas_token`
  - GCP: `bucket`, `client_email`, `private_key`
- **Only the AWS driver exists.** On an Azure- or GCP-backed region the
  factory detects `azure`/`gcp`, finds no driver, and silently falls back to
  the local development store (warn in the log) — the deployment succeeds but
  attachment content lands on the container disk, not a bucket. The code
  comments (`s3-standard`/`azure-container`/`gcs-standard`) predate the plan
  consolidation; the credential shapes, not the plan names, drive the
  detection.
