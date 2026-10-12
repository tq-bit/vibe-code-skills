# Entity and Service Declarations

Declarations for DocumentMetadata and its archive-related service exposure.

## Entity (db/schema.cds)

```cds
// DocumentMetadata entity
// Stores metadata about uploaded files in Database before being archived
entity DocumentMetadata : cuid, managed {
  fileName      : String(255);
  fileType      : String(100);
  fileSize      : Integer;
  // Single attachment per document, base64
  fileContent   : LargeString;
  // After archiving - ObjectStoreId
  fileArchiveId : String(255);
  isLocked      : Boolean;
  schemas       : Association to many DocumentSchema
                    on schemas.document = $self;
  extractions   : Association to many DocumentExtraction
                    on extractions.document = $self;
}
```

Field semantics:

- `fileContent` — plain base64 for non-PDF uploads, or a JSON array of base64 PNG pages after the BEFORE CREATE PDF conversion. Cleared (`null`) once archived; restored from the archive on demand.
- `fileArchiveId` — object key of the archived content in the BTP object store; equals the document's `ID`; `null` means never archived.
- `isLocked` — maintained by `DocumentLockHandler` and the `checkUnusedDocuments` job.

## Service projection (srv/documents/DocumentsApi.cds)

```cds
service DocumentService {
  entity DocumentMetadata as projection on my.DocumentMetadata;
  // ... further projections and actions
}
```

## JobService actions (srv/jobs/JobsApi.cds)

The archiving actions are unbound OData actions of `JobService` (served at
`/odata/v4/job/`), invoked by the SAP Job Scheduler:

```cds
service JobService {
  action archiveDocumentContents();
  action checkUnusedDocuments();
  action deleteDocumentContents();
  action processDocumentExtractions();
}
```

## Handler registration (srv/documents/DocumentsApi.ts)

```typescript
// Restore fileContent of archived documents on read
srv.after('READ', 'DocumentMetadata', async (each, req) => {
  await Promise.all(each.map(async (document: any) => {
    await MetadataHandler.readFileFromArchive(document, req);
  }));
});

// Expanded rows bypass the DocumentMetadata READ handlers -
// the restore must run on the queried entity
srv.after('READ', 'DocumentExtraction', async (results, req) => {
  await MetadataHandler.readFileFromArchiveOnExpand(results, req);
});
srv.after('READ', 'DocumentSchema', async (results, req) => {
  await MetadataHandler.readFileFromArchiveOnExpand(results, req);
});

// Lock check FIRST, remote archive object delete SECOND
srv.before('DELETE', 'DocumentMetadata', (req) => MetadataHandler.checkDocumentLockBeforeDelete(req));
srv.before('DELETE', 'DocumentMetadata', (req) => MetadataHandler.deleteRemoteArchiveObjectBeforeDelete(req));
```

## JobService wiring (srv/jobs/JobsApi.ts)

```typescript
export default (srv: Service) => {
  srv.on('archiveDocumentContents', (req) => executeJobAction(req, archiveDocumentContents,
    (archivedCount) => `Archived ${archivedCount} DocumentMetadata content(s)`));
  srv.on('deleteDocumentContents', (req) => executeJobAction(req, deleteUnlockedDocumentMetadatas,
    (deletedCount) => `Deleted ${deletedCount} unlocked DocumentMetadata entries`));
};
```

`executeJobAction` logs the run start/result and rejects with 500 on unexpected
errors so the Job Scheduler records a failed run; per-item failures are handled
inside the delegated handler modules.
