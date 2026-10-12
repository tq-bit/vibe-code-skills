---
name: implement-cap-documentfiles
description: Load this skill when implementing or changing the DocumentMetadata entity, its DocumentService/JobService exposure, or the object store archiving flow in the CAP backend (db/, srv/) — archiving fileContent, restoring it on read/expand, and deleting remote archive objects. Read before adding document fields, archive handlers, JobService actions, or object store providers.
---

# CAP Document Files (DocumentMetadata + Object Store Archiving)

Implements the pattern used by `DocumentMetadata` in this project: file content
lives in the database until a scheduled job archives it to the BTP object
store, after which the database keeps only metadata plus the object key.

## Lifecycle

```
upload -> DocumentMetadata.fileContent (base64 or JSON array of base64 PNG pages)
  | archiveDocumentContents (JobService action, scheduled by SAP Job Scheduler)
  |   writes content as UTF-8 bytes to object key = document ID
  |   persists fileArchiveId AND CLEARS fileContent in ONE atomic UPDATE
  v
archived document (fileContent = null, fileArchiveId = <ID>)
  | READ with fileContent -> readFileFromArchive (AFTER READ restore)
  | $expand=document(...) -> readFileFromArchiveOnExpand (AFTER READ on related entity)
  | direct DB reads (extraction pipeline) -> readArchivedDocumentContent helper
  | DELETE -> checkDocumentLockBeforeDelete, then deleteRemoteArchiveObjectBeforeDelete
  | cleanup (deleteDocumentContents action) -> remote object first, then row
```

## Where things live

| File | Purpose |
|------|---------|
| `db/schema.cds` | `DocumentMetadata` entity (fileName, fileType, fileSize, fileContent, fileArchiveId, isLocked, schemas, extractions) |
| `srv/documents/DocumentsApi.cds` | `DocumentService` projection of the entity |
| `srv/documents/DocumentsApi.ts` | Handler registration (AFTER READ / BEFORE DELETE wiring) |
| `srv/documents/DocumentMetadataHandler/` | One module per use-case, re-exported via `index.ts` |
| `srv/jobs/JobsApi.cds` / `JobsApi.ts` | Unbound `JobService` actions triggered by the SAP Job Scheduler |
| `srv/store/` | Provider-agnostic object store (`archive.factory.ts`, `core/` drivers) |
| `@types/ObjectStore.d.ts` | `IObjectStore` contract, provider credential types, error codes |

## Support files

Read the file matching the part you are implementing:

- `entity-and-service.md` — CDS entity, service projections, action declarations, handler registration wiring
- `archive-handlers.md` — archive, restore, expand-restore, and remote-delete handler logic
- `object-store.md` — `IObjectStore` contract, factory pattern, driver conventions, `STORE_*` env keys

## Rules that must not be violated

- The object key is always the document's `ID`; persist `fileArchiveId` and clear `fileContent` in the SAME atomic UPDATE.
- The archive holds exactly the original database string as UTF-8 bytes — no transformation, losslessly restorable.
- Remote delete BEFORE row delete; on store failure reject 502 (keep the row, allow retry).
- Raw `DELETE.from(DocumentMetadata)` via `cds.run` bypasses service handlers — cleanup code must delete remote objects explicitly.
- Registered as `srv.after` for READ restores (a BEFORE READ handler cannot modify result rows); expanded rows bypass DocumentMetadata READ handlers, so the expand restore runs on the queried entity.
