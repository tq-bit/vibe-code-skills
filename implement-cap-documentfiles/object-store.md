# Object Store (srv/store/)

Provider-agnostic read/write/delete access to a BTP objectstore service
binding. Plans: `s3-standard` -> aws, `azure-container` -> azure,
`gcs-standard` -> gcp. Adapters wrap the provider-native SDKs
(aws -> `@aws-sdk/client-s3`, azure -> `@azure/storage-blob`,
gcp -> `@google-cloud/storage`).

## Contract (@types/ObjectStore.d.ts)

```typescript
export type TStoreProvider = 'aws' | 'azure' | 'gcp' | 'development';

export type TObjectStoreErrorCode =
  | 'MISSING_CONFIGURATION' | 'NOT_FOUND' | 'READ_FAILED' | 'WRITE_FAILED' | 'DELETE_FAILED';

export type TStoredObject = { content: Uint8Array; contentType?: string; etag?: string };
export type TWriteOptions = { contentType?: string; failIfExists?: boolean };

export interface IObjectStore<TClient = unknown> {
  readonly provider: TStoreProvider;
  readonly client: TClient;          // provider-native SDK client
  readonly bucket: string;           // bucket (AWS/GCP) or container (Azure)
  read(objectKey: string): Promise<TStoredObject>;             // throws NOT_FOUND / READ_FAILED
  write(objectKey: string, content: Uint8Array, options?: TWriteOptions): Promise<void>;  // throws WRITE_FAILED
  delete(objectKey: string): Promise<void>;                    // checks existence first; throws DELETE_FAILED
}
```

Credential types (discriminated on `provider`): `TAwsCredentials`
(bucket, region, accessKeyId, secretAccessKey, host?), `TAzureCredentials`
(accountName, containerName, sasToken), `TGcpCredentials`
(bucket, serviceAccountKey). Adapters narrow the union themselves; the
contract stays credential-free.

`ObjectStoreError` (message, code) and the runtime provider list
`STORE_PROVIDERS` live in `srv/store/core/object-store.interface.ts`.

## Factory (srv/store/archive.factory.ts)

```typescript
export function createArchiveStore(): IObjectStore  // memoized, never throws
```

- Driven by `STORE_PROVIDER`; AWS additionally requires `STORE_REGION`,
  `STORE_ACCESS_KEY_ID`, `STORE_SECRET_ACCESS_KEY`, `STORE_BUCKET`
  (checked via `env.requireProviderEnv`), `STORE_HOST` optional.
- The `DevelopmentObjectStore` is the default fallback: unset provider logs
  info; unknown provider, missing env vars, and driver-less providers
  (azure, gcp) log a warn and also fall back.
- Consumers import `createArchiveStore` from `srv/store/archive.factory`
  directly — there is no barrel `index.ts` for `srv/store/`.

## Driver conventions (core/)

- File naming: `core/<PROVIDER>.driver.ts` (e.g. `AWS.driver.ts`); class
  `AwsObjectStore implements IObjectStore<S3Client>`, constructed from the
  provider's credential type. No driver reads the environment itself —
  validation and credential assembly happen in the factory.
- AWS specifics: a schemeless binding host is normalized to
  `https://<host>` and enables path-style addressing; `failIfExists` maps to
  `IfNoneMatch: '*'`; `delete` checks existence first via
  HeadObjectCommand — a 404 skips the delete with an info log.
- `DEVELOPMENT.driver.ts` — `DevelopmentObjectStore implements
  IObjectStore<null>`: no-op fallback, every operation only logs via
  `logger.info` (read resolves with empty content).
- Error mapping: 404 on read -> `NOT_FOUND`, everything else ->
  `READ_FAILED` / `WRITE_FAILED` / `DELETE_FAILED`.
