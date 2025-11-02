# Write-Ahead Log (WAL) in Fireproof

The Write-Ahead Log (WAL) is a critical component of Fireproof's synchronization system. It manages asynchronous upload of CAR files and file attachments to remote stores, ensuring reliable replication even when the network is intermittent or the application closes unexpectedly.

## Overview

The WAL acts as a persistent queue that:
- **Tracks pending operations** that need to be synced to remote stores
- **Survives application restarts** - operations are persisted and resumed
- **Handles retries automatically** - failed operations are retried with exponential backoff
- **Processes asynchronously** - doesn't block the main application flow

## Purpose

When documents are saved or files are uploaded:
1. Data is **immediately saved locally** (CAR store, File store, etc.)
2. Operations are **queued in WAL** for later synchronization
3. WAL **processes asynchronously** in the background
4. Operations are **retried** if network requests fail
5. Operations are **removed** from queue only after successful upload

This design provides:
- **Fast local writes** - no waiting for network
- **Reliable sync** - guaranteed eventual consistency
- **Offline support** - operations queue while offline, sync when back online
- **Crash recovery** - WAL persists to disk, survives restarts

## WAL State Structure

The WAL maintains three separate queues for different types of operations:

**Location**: `core/types/blockstore/types.ts`

```487:494:core/types/blockstore/types.ts
export interface WALState {
  readonly operations: DbMeta[];
  readonly noLoaderOps: DbMeta[];
  readonly fileOperations: {
    readonly cid: AnyLink;
    readonly public: boolean;
  }[];
}
```

### 1. `operations` - Document Data (CAR files)

Contains metadata about CAR files that need to be synced:

```typescript
interface DbMeta {
  cars: AnyLink[];  // Array of CAR file CIDs
}
```

**When added**: After committing document transactions (normal commits)

**Processing**: CAR files are loaded from local store and uploaded to remote stores

### 2. `noLoaderOps` - CAR Files Without Loader

Same structure as `operations`, but processed when the loader isn't available:

```typescript
interface DbMeta {
  cars: AnyLink[];
}
```

**When added**:
- When `opts.noLoader` is true (commit without loader)
- When `opts.compact` is true (compact operations)

**Processing**: Similar to `operations` but doesn't require loader to be active

### 3. `fileOperations` - File Attachments

Contains file CIDs and their public/private status:

```typescript
{
  cid: AnyLink;      // File CID
  public: boolean;   // Whether file is public
}
```

**When added**: After encoding file attachments (via `enqueueFile()`)

**Processing**: File blocks are loaded from local file store and uploaded to remote file stores

## WAL Store Implementation

**Location**: `core/blockstore/store.ts:WALStoreImpl`

The WAL store manages the state and processing:

```418:451:core/blockstore/store.ts
export class WALStoreImpl extends BaseStoreImpl implements WALStore {
  readonly storeType = "wal";
  
  readonly walState: WALState = { operations: [], noLoaderOps: [], fileOperations: [] };
  readonly processQueue: CommitQueueIf<unknown>;

  async ready(): Promise<void> {
    // Load persisted WAL state from storage
    const walState = await this.load();
    if (walState) {
      this.walState.operations.push(...walState.operations);
      this.walState.fileOperations.push(...walState.fileOperations);
    }
  }
}
```

## Operation Enqueuing

### Document Operations

**Location**: `core/blockstore/store.ts:enqueue`

```453:468:core/blockstore/store.ts
async enqueue(dbMeta: DbMeta, opts: CommitOpts) {
  await this.ready();
  if (opts.compact) {
    this.walState.operations.splice(0, this.walState.operations.length);
    this.walState.noLoaderOps.splice(0, this.walState.noLoaderOps.length);
    this.walState.noLoaderOps.push(dbMeta);
  } else if (opts.noLoader) {
    this.walState.noLoaderOps.push(dbMeta);
  } else {
    this.walState.operations.push(dbMeta);
  }
  await this.save(this.walState);
  if (!opts.noLoader) {
    void this.process();
  }
}
```

**How it works**:
1. Operation added to appropriate queue (`operations`, `noLoaderOps`)
2. WAL state saved to persistent storage
3. Processing triggered (unless `noLoader`)

**Called from**: `core/blockstore/loader.ts:writeWAL`

```147:150:core/blockstore/loader.ts
async writeWAL(cids: AnyLink[]): Promise<void> {
  await this.attached.local().active.wal.enqueue({ cars: cids }, this.opts);
}
```

### File Operations

**Location**: `core/blockstore/store.ts:enqueueFile`

```470:474:core/blockstore/store.ts
async enqueueFile(fileCid: AnyLink, publicFile = false) {
  await this.ready();
  this.walState.fileOperations.push({ cid: fileCid, public: publicFile });
  // await this.save(this.walState)
}
```

**How it works**:
1. File CID and public flag added to `fileOperations` queue
2. Note: Save is commented out (may be optimized to batch saves)

**Called from**: File encoding process (via `FileStore.enqueueFile()`)

## WAL Processing

### Processing Flow

**Location**: `core/blockstore/store.ts:_doProcess`

The WAL processes operations asynchronously:

```491:612:core/blockstore/store.ts
async _doProcess() {
  if (!this.loader) return;
  
  const operations = [...this.walState.operations];
  const noLoaderOps = [...this.walState.noLoaderOps];
  const fileOperations = [...this.walState.fileOperations];

  if (operations.length + noLoaderOps.length + fileOperations.length === 0) return;

  const concurrencyLimit = 3;

  // Process each queue type with retry logic
  // 1. Process noLoaderOps
  // 2. Process operations
  // 3. Process fileOperations
  
  // Save state after processing
  await this.save(this.walState);
}
```

### Processing Details

**Processing Order**:
1. **noLoaderOps** - Processed first (compacted/offline operations)
2. **operations** - Normal document operations
3. **fileOperations** - File attachments

**Processing Characteristics**:
- **Concurrent**: Up to 3 operations processed in parallel (configurable via `concurrencyLimit`)
- **Retry Logic**: Each operation retries up to 5 times with exponential backoff
- **Error Handling**: Failed operations remain in queue for retry
- **State Persistence**: WAL state saved after each processing cycle

### CAR File Processing

For each `DbMeta` in `operations` or `noLoaderOps`:

```typescript
// For each CAR CID in the operation
for (const cid of dbMeta.cars) {
  // Load CAR from local store
  const car = await this.loader.attachedStores.local().active.car.load(cid);
  
  if (!car) {
    // Error if CAR missing and should exist
    throw new Error("missing local car");
  }
  
  // Upload to all remote stores
  await this.loader.attachedStores.forRemotes((x) => x.active.car.save(car));
}

// Remove from queue after successful upload
inplaceFilter(this.walState.operations, (op) => op !== dbMeta);
```

### File Processing

For each file in `fileOperations`:

```typescript
// Load file from local file store
const fileBlock = await this.loader.attachedStores.local().active.file.load(fileCid);

if (!fileBlock) {
  throw new Error("missing file block");
}

// Upload to all remote file stores
await this.loader.attachedStores.forRemotes((x) => 
  x.active.file.save(fileBlock, { public: false })
);

// Remove from queue after successful upload
inplaceFilter(this.walState.fileOperations, (op) => op.cid !== fileCid);
```

**Note**: The `public` flag from `fileOperations` is currently not used (always passed as `{ public: false }`). This may be a bug or intentional limitation.

### Automatic Retry

Operations use `pRetry` with exponential backoff:

```typescript
const retryableUpload = <T>(fn: () => Promise<T>, description: string) =>
  pRetry(fn, {
    retries: 5,
    onFailedAttempt: (error) => {
      // Log retry attempt
      this.logger.Warn()
        .Msg(`Attempt ${error.attemptNumber} failed. ${error.retriesLeft} retries left.`);
    },
  });
```

**Retry Behavior**:
- **Max retries**: 5 attempts
- **Backoff**: Exponential (handled by `pRetry`)
- **Persistence**: Failed operations remain in WAL state
- **Recovery**: Operations resume on next `process()` call

## Triggering Processing

### Automatic Processing

Processing is automatically triggered when:
1. **Operations enqueued**: After `enqueue()` (unless `opts.noLoader`)
2. **Recursive processing**: After each `_doProcess()` if queue not empty

```476:489:core/blockstore/store.ts
async process() {
  await this.ready();
  await this.processQueue.enqueue(async () => {
    try {
      await this._doProcess();
    } catch (e) {
      this.logger.Error().Any("error", e).Msg("error processing wal");
    }
    // Continue processing if queue not empty
    if (this.walState.operations.length || 
        this.walState.fileOperations.length || 
        this.walState.noLoaderOps.length) {
      setTimeout(() => void this.process(), 0);
    }
  });
}
```

### Manual Processing

You can manually trigger processing:

```typescript
const walStore = db.ledger.crdt.blockstore.loader.attachedStores.local().active.wal;
await walStore.process();
```

## Persistence

### WAL State Storage

WAL state is persisted to the WAL store:

**Save**:
```633:652:core/blockstore/store.ts
async save(state: WALState) {
  const filepath = await this.gateway.buildUrl({ loader: this.loader }, this.url(), "main");
  await this.gateway.put({ loader: this.loader }, filepath.Ok(), {
    type: "wal",
    payload: state,
  } as FPEnvelopeWAL);
}
```

**Load**:
```614:632:core/blockstore/store.ts
async load(): Promise<WALState | Falsy> {
  const filepath = await this.gateway.buildUrl({ loader: this.loader }, this.url(), "main");
  const bytes = await this.gateway.get(filepath.Ok(), this.loader);
  if (!bytes.isOk()) {
    return;
  }
  if (bytes.Ok().type !== "wal") {
    throw this.logger.Error().Msg("unexpected type").AsError();
  }
  return bytes.Ok().payload;
}
```

**Storage Location**:
- **IndexedDB**: `indexeddb://{ledger-name}?store=wal`
- **File System**: `file://{path}/{ledger-name}?store=wal`
- **Memory**: `memory://{ledger-name}?store=wal` (ephemeral)

## Integration with Other Stores

### CAR Store

CAR files are:
1. **Saved locally** immediately via `car.save()`
2. **Enqueued to WAL** via `wal.enqueue()`
3. **Synced asynchronously** when WAL processes

### Meta Store

Metadata (list of CAR CIDs):
1. **Saved locally** immediately
2. **Synced after CAR uploads** (last operation's metadata sent)

### File Store

File attachments are:
1. **Saved locally** immediately via `file.save()`
2. **Enqueued to WAL** via `wal.enqueueFile()`
3. **Synced asynchronously** when WAL processes

### Relationship Flow

```
Document Save:
  ↓
1. CAR file saved to local CAR store
  ↓
2. Metadata saved to local Meta store
  ↓
3. Operation enqueued to WAL
  ↓
4. WAL processes (async):
    ↓
    a. Load CAR from local store
    ↓
    b. Upload CAR to remote stores
    ↓
    c. Remove from WAL queue
    ↓
    d. Save updated WAL state

File Upload:
  ↓
1. File saved to local File store
  ↓
2. File CID enqueued to WAL (fileOperations)
  ↓
3. WAL processes (async):
    ↓
    a. Load file from local store
    ↓
    b. Upload file to remote stores
    ↓
    c. Remove from WAL queue
    ↓
    d. Save updated WAL state
```

## WAL State Management

### State Lifecycle

1. **Initialization**: Load persisted state on `ready()`
2. **Enqueuing**: Add operations, save state
3. **Processing**: Copy queues, process, update state
4. **Cleanup**: Remove successful operations, save state
5. **Recovery**: Re-load state on restart, resume processing

### State Cleanup

Operations are removed from state only after:
- ✅ Successful upload to remote stores
- ✅ All retries completed successfully
- ✅ No errors during processing

Failed operations remain in state for retry on next processing cycle.

## Error Handling

### Network Failures

- Operations remain in WAL queue
- Automatic retry with exponential backoff
- State persisted after each attempt
- Processing resumes on next cycle

### Missing Local Data

If local CAR/file is missing:
- Error thrown and logged
- Operation remains in queue
- May indicate data corruption or race condition

### Processing Errors

If `_doProcess()` throws:
- Error logged
- State saved (including failed operations)
- Processing stops until next trigger
- Failed operations remain for retry

## Best Practices

### 1. Monitor WAL Queue Size

Large WAL queues may indicate:
- Network connectivity issues
- Slow remote storage
- High write throughput

Monitor via:
```typescript
const walStore = db.ledger.crdt.blockstore.loader.attachedStores.local().active.wal;
await walStore.ready();
console.log('Pending operations:', walStore.walState.operations.length);
console.log('Pending files:', walStore.walState.fileOperations.length);
```

### 2. Handle Offline Scenarios

WAL automatically handles offline:
- Operations queue while offline
- Resume processing when online
- No manual intervention needed

### 3. Backup WAL State

For critical applications, consider backing up WAL state to ensure no operations are lost.

### 4. Understand Compact Operations

Compact operations (`opts.compact = true`):
- Clear all existing operations
- Add to `noLoaderOps` queue
- Used for database compaction

## WAL vs. Direct Sync

### WAL (Current Approach)

✅ **Pros**:
- Fast local writes (no network wait)
- Automatic retry and recovery
- Persistent across restarts
- Handles offline gracefully

❌ **Cons**:
- Eventual consistency (not immediate)
- Additional storage for WAL state
- Complexity in state management

### Direct Sync (Alternative)

✅ **Pros**:
- Immediate consistency
- Simpler state management
- No queue storage needed

❌ **Cons**:
- Slower writes (waits for network)
- No offline support
- No automatic retry
- Blocks on network failures

## Related Files

- `core/blockstore/store.ts` - WALStoreImpl implementation
- `core/types/blockstore/types.ts` - WALState and WALStore interfaces
- `core/blockstore/loader.ts` - Integration with commit process
- `core/blockstore/commitor.ts` - Commit orchestration

## Summary

The WAL is Fireproof's reliability mechanism for asynchronous synchronization. It ensures that all local writes eventually reach remote stores, even in the face of network failures, application crashes, or offline scenarios. By decoupling local writes from remote sync, Fireproof provides both fast local performance and reliable eventual consistency.

