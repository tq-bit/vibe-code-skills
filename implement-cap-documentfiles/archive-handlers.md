# Archive Handler Logic

Modules live in `srv/documents/DocumentMetadataHandler/`, one file per
use-case, re-exported via the folder's `index.ts`. Helpers throw custom
errors from `srv/util/errors.ts` / `ObjectStoreError`; exported handlers map
them to `req.reject` status codes. All snippets are the implemented logic.

## Archive (archiveDocumentContents.ts)

Archives every `DocumentMetadata` with `fileArchiveId IS NULL` and non-null
`fileContent`. Sequential per document; per-document failures are logged and
skipped (retried on the next scheduled run):

```typescript
export async function archiveDocumentContents(): Promise<number> {
  const { DocumentMetadata } = cds.entities('DocumentService');

  const pendingDocuments = await SELECT
    .from(DocumentMetadata)
    .columns('ID')
    .where({ fileArchiveId: null, fileContent: { '!=': null } }) as { ID: string }[];

  const store = createArchiveStore();
  let archivedCount = 0;

  for (const { ID } of pendingDocuments) {
    try {
      // 1. Read the content (joins the ambient transaction)
      const document = await SELECT.one
        .from(DocumentMetadata)
        .columns('ID', 'fileContent')
        .where({ ID, fileArchiveId: null }) as { ID: string; fileContent: string };
      if (!document) continue; // Already archived by a concurrent run

      // 2. Write the object to the archive; object key = document ID
      const objectKey = document.ID;
      await store.write(objectKey, fileContentToArchiveBytes(document.fileContent));

      // 3. Persist the archive id AND clear the content in ONE atomic UPDATE
      await UPDATE(DocumentMetadata)
        .set({ fileArchiveId: objectKey, fileContent: null })
        .where({ ID: objectKey });
      archivedCount++;
    } catch (error) {
      logger.error(`archiveDocumentContents - Failed to archive document ${ID}: ${error}`);
    }
  }
  return archivedCount;
}

// The archive holds exactly the original database string, as UTF-8 bytes —
// losslessly restorable for both content shapes (plain base64, JSON array of
// base64 PNG pages)
function fileContentToArchiveBytes(fileContent: string): Uint8Array {
  return Buffer.from(fileContent, 'utf-8');
}
```

## Restore (readFileFromArchive.ts)

Three exports:

- `readArchivedDocumentContent(fileArchiveId)` — reads the object and decodes UTF-8; throws `ObjectStoreError` (`NOT_FOUND` / `READ_FAILED`). Used by direct DB readers (extraction pipeline) that bypass service handlers.
- `restoreArchivedDocumentContent(document, req)` — shared row-restore: skips when `fileContent` is present or the document was never archived; reads `fileArchiveId` from the database (it is not part of every column selection); on failure rejects 404 (`NOT_FOUND`) or 502.
- `readFileFromArchive(each, req)` — `srv.after('READ', 'DocumentMetadata')`:

```typescript
export async function readFileFromArchive(each: { ID: string; fileContent: string | null }, req: Request) {
  const columns = req.query?.SELECT?.columns;
  // Without an explicit column selection every column is returned
  const requestsFileContent = !columns || columns.some((column) => column.ref?.includes('fileContent'));
  if (!requestsFileContent) return;

  await restoreArchivedDocumentContent(each, req);
}
```

## Restore on $expand (readFileFromArchiveOnExpand.ts)

`AFTER READ` on `DocumentExtraction` and `DocumentSchema`: expanded document rows
bypass the DocumentMetadata READ handlers, so the restore runs on the queried
entity — but only when the expand's column projection explicitly selects
`fileContent` (expands WITHOUT a `$select` never restore, so list bindings do
not cause an archive read per row):

```typescript
function expandRequestsFileContent(req: Request): boolean {
  const columns = req.query?.SELECT?.columns ?? [];
  return columns.some((column) => {
    if (column.ref?.[column.ref.length - 1] !== 'document' || !Array.isArray(column.expand)) return false;
    return column.expand.some((expandColumn) => expandColumn.ref?.includes('fileContent'));
  });
}

export async function readFileFromArchiveOnExpand(results, req): Promise<void> {
  if (!expandRequestsFileContent(req)) return;
  const rows = Array.isArray(results) ? results : [results];
  await Promise.all(
    rows.filter((row) => row?.document?.ID)
      .map(async (row) => restoreArchivedDocumentContent(row.document, req))
  );
}
```

## Remote delete (deleteRemoteArchiveObject.ts)

- `deleteRemoteArchiveObject(fileArchiveId)` — shared remote-delete helper (also used by the cleanup action).
- `deleteRemoteArchiveObjectBeforeDelete(req)` — `BEFORE DELETE`, registered AFTER `checkDocumentLockBeforeDelete`: deletes the remote object first; on store failure rejects 502 so the row is kept and the delete can be retried. Never-archived documents delete without a remote call.

```typescript
const document = await SELECT.one
  .from(DocumentMetadata)
  .columns('ID', 'fileArchiveId')
  .where({ ID: documentId });
if (!document?.fileArchiveId) return; // Never archived

try {
  await deleteRemoteArchiveObject(document.fileArchiveId);
} catch (error) {
  return req.reject(502, `Cannot delete remote archive object '${document.fileArchiveId}' of document ${documentId}: ${message}`);
}
```

## Unlocked cleanup (deleteUnlockedDocuments.ts)

JobService action `deleteDocumentContents`: deletes the remote object first;
on failure the document is skipped (warn) and retried on the next run. Uses a
raw `DELETE.from(DocumentMetadata)` which does NOT trigger service-level
handlers — hence the explicit remote deletes:

```typescript
const deletableIds: string[] = [];
for (const document of unlockedDocuments) {
  if (!document.fileArchiveId) { deletableIds.push(document.ID); continue; }
  try {
    await deleteRemoteArchiveObject(document.fileArchiveId);
    deletableIds.push(document.ID);
  } catch (error) {
    logger.warn(`Skipping document ${document.ID}: remote archive object could not be deleted: ${error}`);
  }
}
await DELETE.from(DocumentMetadata).where({ ID: { in: deletableIds } });
```
