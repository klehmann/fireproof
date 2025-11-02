# File Attachments in Fireproof

Fireproof has special handling for file attachments stored in the `_files` property of documents. This document explains how files are stored, encoded, and synchronized across different gateways.

**Important Note**: Public files (`_publicFiles`) automatically receive URLs using web3.storage's IPFS gateway format (`https://{CID}.ipfs.w3s.link/`). This URL is hardcoded in `core/base/crdt-helpers.ts` and requires files to be accessible via IPFS to work.

## Overview

File attachments in Fireproof are handled differently from regular document data:

1. **Separate Storage**: Files are stored separately from CAR files (document data)
2. **Dual Store System**: Two independent stores - CAR store for documents, File store for attachments
3. **Special Encoding**: Files use raw encoding (not IPLD DAG-CBOR)
4. **Async Synchronization**: Files sync via a separate WAL queue
5. **CID-Based**: Each file gets a Content Identifier (CID) for content-addressed storage

## File Attachment Structure

### Document Format

Files are stored in the `_files` property of documents:

```typescript
interface Document {
  _id: string;
  _files?: {
    [filename: string]: File | DocFileMeta;
  };
  _publicFiles?: {
    [filename: string]: File | DocFileMeta;
  };
  // ... other fields
}
```

### File Metadata

When a file is processed, the `File` object is replaced with metadata:

```typescript
interface DocFileMeta {
  type: string;           // MIME type
  size: number;           // File size in bytes
  cid: AnyLink;           // Content Identifier
  car?: AnyLink;          // CAR file containing the file
  lastModified?: number;  // File modification timestamp
  url?: string;           // Optional URL for public files
  file?: () => Promise<File>;  // Async getter function
}
```

## File Encoding Process

### 1. Initial Upload (`processFileset`)

When a document with `_files` is saved:

**Location**: `core/base/crdt-helpers.ts:processFileset`

```typescript
// File is converted from File object to metadata
const { cid, blocks: fileBlocks } = await store.encodeFile(file);

// Blocks are added to transaction
for (const block of fileBlocks) {
  t.putSync(await fileBlock2FPBlock(block));
}

// File object is replaced with metadata
files[filename] = { 
  cid, 
  type: file.type, 
  size: file.size, 
  lastModified: file.lastModified 
};
```

### 2. File Encoding (`encodeFile`)

**Location**: `core/runtime/files.ts`

Files are encoded using the **raw codec** (not DAG-CBOR):

```typescript
export async function encodeFile(blob: Blob): Promise<{ cid: AnyLink; blocks: AnyBlock[] }> {
  // 1. Convert blob to Uint8Array
  const data = await top_uint8(blob);
  
  // 2. Encode with raw codec (simple binary, no structure)
  const bytes = raw.encode(data);
  
  // 3. Create CID (content-addressed identifier)
  const hash = await hasher.digest(bytes);
  const cid = CID.create(1, raw.code, hash);
  
  // 4. Return single block with CID
  const block = { cid, bytes };
  return { cid, blocks: [block] };
}
```

**Key Differences from Document Encoding:**
- Documents use **DAG-CBOR** (structured, can link to other blocks)
- Files use **raw codec** (simple binary blob)
- Files are single blocks (no DAG structure)
- Files get a CID based on their content hash

### 3. File Commit (`commitFiles`)

**Location**: `core/blockstore/commitor.ts`

Files are committed to a separate FileStore:

```typescript
export async function commitFiles(
  fileStore: FileStore,
  walStore: WALStore,
  t: CarTransaction,
  done: TransactionMeta,
): Promise<CarGroup> {
  // Extract file CIDs from metadata
  const { files: roots } = makeFileCarHeader(done);
  
  // Create CAR file containing file blocks
  const cars = await prepareCarFilesFiles(codec, roots, t);
  
  // Save to file store
  for (const car of cars) {
    await fileStore.save({ cid, bytes });
    await walStore.enqueueFile(cid);  // Add to WAL queue
    cids.push(cid);
  }
  
  return cids;
}
```

## Storage Architecture

### Dual Store System

Fireproof maintains **two separate storage systems**:

1. **CAR Store** (`store: "car"`)
   - Document data
   - CRDT operations
   - DAG structures
   - Metadata
   - Location: `data/tenant/ledger/car-files/`

2. **File Store** (`store: "file"`)
   - File attachments
   - Raw file data
   - File CAR files
   - Location: `data/tenant/ledger/file-store/`

### Store Separation

The separation is enforced at multiple levels:

**1. Store Factory** (`core/blockstore/store-factory.ts`):
```typescript
async function fileStoreFactory(ctx: SerdeGatewayCtx, uai: UrlAndInterceptor): Promise<FileStore> {
  // FileStore URLs must have store=file parameter
  const storeUrl = uai.url.build().setParam(PARAM.STORE, "file").URI();
  const gateway = await getInterceptableGateway(ctx, storeUrl, uai);
  return new FileStoreImpl(sthis, uai.url, { gateway, loader });
}
```

**2. FileStore Implementation** (`core/blockstore/store.ts`):
```typescript
export class FileStoreImpl extends DataStoreImpl implements FileStore {
  readonly storeType = "file";  // Explicitly marked as file store
  // ... implementation
}
```

## Synchronization Flow

### Local Storage

1. **User uploads file** via `_files` property
2. **File encoded** to CID and blocks
3. **Blocks stored locally** (IndexedDB/file system)
4. **Metadata saved** to document
5. **File CID added** to WAL queue

**Location**: `core/base/crdt-helpers.ts:processFileset`

### Remote Synchronization

Files are synced separately from documents via the WAL queue:

**Location**: `core/blockstore/store.ts:processWAL`

```typescript
// Process fileOperations queue
await pMap(
  fileOperations,
  async ({ cid: fileCid }) => {
    // 1. Load file from local store
    const fileBlock = await this.loader.attachedStores.local().active.file.load(fileCid);
    
    // 2. Save to all remote stores using FileStore
    await this.loader.attachedStores.forRemotes((x) => 
      x.active.file.save(fileBlock, { public: false })
    );
    
    // 3. Remove from queue after success
    inplaceFilter(this.walState.fileOperations, (op) => op.cid !== fileCid);
  }
);
```

### Gateway Communication

When syncing files, the gateway receives:

- **Store Parameter**: `store: "file"` (distinguishes from `store: "car"`)
- **File Data**: Raw bytes of the file block
- **CID**: Content identifier for the file

**Server-side** (`cloud/backend/base/`):

```typescript
// Server checks store parameter
if (msg.methodParam.store === 'file') {
  // Route to file store
  // Save to: data/tenant/ledger/file-store/
  await storage.saveFile(tenantInfo, key, fileData);
} else if (msg.methodParam.store === 'car') {
  // Route to CAR store
  // Save to: data/tenant/ledger/car-files/
  await storage.saveCarFile(tenantInfo, key, carData);
}
```

## File Decoding Process

### Lazy Loading

Files are **not automatically loaded** when documents are retrieved. Instead, metadata is attached including URLs for public files and async getter functions:

**Location**: `core/base/crdt-helpers.ts:readFileset`

```typescript
function readFileset(blocks: EncryptedBlockstore, files: DocFiles, isPublic = false) {
  for (const filename in files) {
    const fileMeta = files[filename] as DocFileMeta;
    if (fileMeta.cid) {
      // 1. Public files get automatic IPFS gateway URL
      if (isPublic) {
        fileMeta.url = `https://${fileMeta.cid.toString()}.ipfs.w3s.link/`;
      }
      
      // 2. Attach async getter function for lazy loading
      if (fileMeta.car) {
        fileMeta.file = async () => {
          // Load file from blockstore using CAR and CID
          const result = await blocks.ebOpts.storeRuntime.decodeFile(
            {
              get: async (cid: AnyLink) => {
                return await blocks.getFile(fileMeta.car!, cid);
              },
            },
            fileMeta.cid,
            fileMeta,
          );
          return result.unwrap();
        };
      }
    }
    files[filename] = fileMeta;
  }
}
```

**Key Points**:
- **Public files** (`_publicFiles`): Get automatic `url` property with IPFS gateway URL (`https://{CID}.ipfs.w3s.link/`)
- **Private files** (`_files`): No automatic URL, must use `file()` async getter
- **Both types**: Have `file()` async getter for loading from blockstore

### File Decoding (`decodeFile`)

**Location**: `core/runtime/files.ts`

```typescript
export async function decodeFile(
  blocks: BlockGetter,
  cid: AnyLink,
  meta: DocFileMeta
): Promise<Result<File>> {
  // 1. Get block data from storage
  const bytes = await blocks.get(cid);
  
  // 2. Decode from raw codec
  const data = raw.decode(bytes) as Uint8Array;
  
  // 3. Recreate File object with original metadata
  return new File([data], "file", {
    type: meta.type,
    lastModified: meta.lastModified || 0,
  });
}
```

### Usage in Application

```typescript
// Document retrieved
const doc = await db.get("doc-id");

// File metadata is available immediately
const fileMeta = doc._files?.image;

// File content loaded on-demand
if (fileMeta?.file) {
  const file = await fileMeta.file();
  // Use file (display image, download, etc.)
}
```

## WAL Queue for Files

### File Operations Queue

Files have a **separate queue** in the WAL (Write-Ahead Log):

**Location**: `core/blockstore/store.ts`

```typescript
interface WALState {
  operations: [];        // Document operations
  noLoaderOps: [];       // Operations without loader
  fileOperations: [      // File attachment operations
    { cid: AnyLink; public: boolean }
  ];
}
```

### Enqueuing Files

When a file is committed:

```typescript
// File CID added to WAL queue
this.walState.fileOperations.push({ 
  cid: fileCid, 
  public: publicFile 
});

// WAL state saved (including fileOperations)
await walStore.save(this.walState);
```

### Processing Files

Files are processed independently of document operations:

```typescript
// Process documents first
await processDocumentOperations();

// Then process files separately
await processFileOperations();

// Files processed in parallel with concurrency limit
await pMap(fileOperations, async ({ cid }) => {
  // ... sync file to remote stores
}, { concurrency: concurrencyLimit });
```

## Gateway Implementation

### FileStore Interface

FileStore extends BaseStore with special file handling:

```typescript
interface FileStore extends BaseStore {
  readonly storeType: "file";
  
  // File-specific operations
  save(block: { cid: AnyLink; bytes: Uint8Array }): Promise<void>;
  load(cid: AnyLink): Promise<Uint8Array | null>;
}
```

### Gateway URL Format

File stores are identified by the `store=file` parameter:

```
indexeddb://my-app?store=file
file:///path/to/storage?store=file
http://server.com/storage?store=file
```

### Cloud Gateway

When syncing to cloud, the `store` parameter is passed:

```typescript
// Gateway checks store parameter
const store = url.getParam(PARAM.STORE);

if (store === "file") {
  // Use file store endpoints
  // PUT /upload-file/{tenant}/{ledger}/{key}
  // GET /fp-file?cid={cid}
} else if (store === "car") {
  // Use CAR store endpoints
  // PUT /fp?car={cid}
  // GET /fp?car={cid}
}
```

## Key Differences: Files vs Documents

| Aspect | Documents | Files |
|--------|-----------|-------|
| **Encoding** | DAG-CBOR (structured) | Raw (binary) |
| **Structure** | DAG (can link blocks) | Single block |
| **Store Type** | `store: "car"` | `store: "file"` |
| **Storage Location** | `car-files/` | `file-store/` |
| **Sync Queue** | `operations[]` | `fileOperations[]` |
| **Processing** | With documents | Separately |
| **Loading** | Immediate | Lazy (on-demand) |
| **CID Usage** | Document references | Direct file content |
| **Public URLs** | N/A | Auto-generated for `_publicFiles` (IPFS gateway) |

## Public File URLs

### How Public File URLs Work

Public files stored in `_publicFiles` automatically receive a URL when documents are loaded:

**URL Generation**: `core/base/crdt-helpers.ts:readFileset`

```typescript
if (isPublic) {
  fileMeta.url = `https://${fileMeta.cid.toString()}.ipfs.w3s.link/`;
}
```

**URL Format**: 
- Pattern: `https://{CID}.ipfs.w3s.link/`
- Example: `https://bafybeihdwdcefgh4dqkjv67uzcmw7ojee6xedzdetojuzjevtenxquvyku.ipfs.w3s.link/`

**Important Notes**:

1. **Hardcoded Gateway**: The web3.storage IPFS gateway (`w3s.link`) is currently hardcoded in the codebase. This is not configurable.

2. **Requires IPFS Upload**: For the URL to actually work, the file must be accessible via IPFS. Simply marking a file as `public` does not automatically upload it to IPFS - you must configure Fireproof to sync files to an IPFS-compatible storage (e.g., web3.storage gateway).

3. **CID-Based**: The URL uses the file's CID as a subdomain, which is a standard IPFS gateway convention.

4. **Private Files**: Files in `_files` (private) do NOT get automatic URLs. They must be accessed via the `file()` async getter function.

**Usage**:

```typescript
// Document with public file
const doc = await db.get("doc-id");

// Public file has URL automatically
if (doc._publicFiles?.image?.url) {
  // Can use URL directly (if file is on IPFS)
  <img src={doc._publicFiles.image.url} />
}

// Or load file programmatically
if (doc._publicFiles?.image?.file) {
  const file = await doc._publicFiles.image.file();
  const objectUrl = URL.createObjectURL(file);
  // Use objectUrl for display
}
```

## Special Considerations

### File Size

Files can be large, so:
- Files are stored as single blocks (no DAG overhead)
- Files are processed separately to avoid blocking document sync
- File operations use parallel processing with concurrency limits

### File Deduplication

Files are content-addressed (CID-based):
- Same file content = same CID
- Duplicate uploads are automatically deduplicated
- Sharing files between documents reuses the same CID

### Public Files

Files can be marked as `public`:
- Stored in `_publicFiles` property
- Automatically get public URLs (IPFS gateway format)
- Separate from private files

**Public File URLs**:

When files are stored in `_publicFiles`, Fireproof automatically generates a public URL using web3.storage's IPFS gateway format:

**Location**: `core/base/crdt-helpers.ts:readFileset`

```252:280:core/base/crdt-helpers.ts
function readFileset(blocks: EncryptedBlockstore, files: DocFiles, isPublic = false) {
  for (const filename in files) {
    const fileMeta = files[filename] as DocFileMeta;
    if (fileMeta.cid) {
      // Public files get automatic IPFS gateway URL
      if (isPublic) {
        fileMeta.url = `https://${fileMeta.cid.toString()}.ipfs.w3s.link/`;
      }
      // ... rest of file loading logic
    }
  }
}
```

The URL is generated at line 257 of `core/base/crdt-helpers.ts`.

**URL Format**: `https://{CID}.ipfs.w3s.link/`

Where:
- `{CID}` is the content identifier of the file
- Uses web3.storage's IPFS gateway (w3s.link)
- The CID becomes a subdomain (e.g., `bafybei...` → `bafybei....ipfs.w3s.link`)

**Note**: This URL format is currently **hardcoded** and assumes files are accessible via web3.storage's IPFS gateway. For files to actually be accessible via this URL, they must be uploaded to IPFS (typically via web3.storage or another IPFS provider).

**Private Files**:
- Files in `_files` property do NOT get automatic URLs
- Must be accessed via the `file()` async getter function
- Loaded from local or attached stores

### File Deletion

When documents are deleted:
- File metadata is removed from document
- File blocks remain in storage (referenced by CID)
- Garbage collection can remove unreferenced files

## Implementation Files

Key files for file attachment handling:

- `core/runtime/files.ts` - File encoding/decoding
- `core/base/crdt-helpers.ts` - File processing in CRDT operations
- `core/blockstore/store.ts` - FileStore implementation and WAL processing
- `core/blockstore/commitor.ts` - File commit to storage
- `core/blockstore/store-factory.ts` - FileStore factory
- `core/types/base/types.ts` - File type definitions
- `core/types/blockstore/types.ts` - FileStore interface

## Summary

File attachments in Fireproof are handled specially:

1. **Separate Storage**: Files stored separately from documents via `store=file` parameter
2. **Raw Encoding**: Files use raw codec (simple binary) vs DAG-CBOR for documents
3. **Async Sync**: Files sync via separate WAL queue (`fileOperations`)
4. **Lazy Loading**: Files loaded on-demand via async getter function
5. **Content Addressing**: Files identified by CID (content hash)
6. **Dual System**: CAR store for documents, File store for attachments

This architecture allows:
- Efficient handling of large files
- Independent synchronization of files and documents
- Automatic deduplication via content addressing
- Flexible loading (on-demand vs immediate)

