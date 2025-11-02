# Metadata and Synchronization in Fireproof

Metadata is a critical component of Fireproof's synchronization system. It serves as the "coordination layer" that tells clients what data needs to be synced and in what order, without requiring servers to understand the encrypted content.

## What is Metadata?

**Location**: `core/types/blockstore/types.ts`

Metadata (`DbMeta`) is a lightweight data structure that contains pointers to encrypted CAR files:

```typescript
interface DbMeta {
  cars: AnyLink[];  // Array of CAR file CIDs (Content Identifiers)
}
```

**Key Points**:
- **Lightweight**: Only contains CIDs, not the actual data
- **Encrypted**: Metadata itself is encrypted before being sent to servers
- **Content-Addressed**: CAR files are identified by their CID (hash), ensuring integrity

### Metadata Structure

When metadata is sent over the network, it's packaged as `DbMetaEvent`:

```434:438:core/types/blockstore/types.ts
export interface DbMetaEvent {
  readonly eventCid: CarClockLink;
  readonly parents: CarClockHead;
  readonly dbMeta: DbMeta;
}
```

- **`eventCid`**: The CID of this metadata event
- **`parents`**: Parent metadata events (for causal ordering)
- **`dbMeta`**: The actual metadata (list of CAR file CIDs)

## Role of Metadata During Sync

Metadata serves three primary functions during synchronization:

### 1. Discovery: What Data Exists?

When a client connects to a server or another client:
1. Client requests metadata from the server
2. Server responds with list of `DbMeta` entries
3. Client compares metadata with local state
4. Client determines which CAR files are missing

**Flow**:
```
Client → reqGetMeta → Server
Server → EventGetMeta → Client (list of DbMeta entries)
Client → Identifies missing CAR files
Client → reqGetData → Server (for each missing CAR)
```

### 2. Ordering: What's the Latest State?

Metadata includes a **clock head** (`CRDTMeta.head`) that represents the current state:

```298:300:core/types/base/types.ts
export interface CRDTMeta {
  readonly head: ClockHead;
}
```

The clock head is a vector clock that points to the latest events. By comparing clock heads:
- Clients can determine if they're up-to-date
- Clients can identify which updates they're missing
- The CRDT can merge concurrent changes correctly

### 3. Efficiency: Download Only What's Needed

Instead of downloading all data, clients:
1. Download metadata (small, fast)
2. Compare with local state
3. Download only missing CAR files
4. Merge changes using CRDT logic

**Example**:
- Local clock head: `[cid1, cid2]`
- Remote clock head: `[cid1, cid2, cid3, cid4]`
- Client knows: Need to download CAR files referenced by `cid3` and `cid4`

## Multiple Heads (Concurrent Branches)

The clock head is an **array of CIDs**, not a single value:

```201:201:core/types/base/types.ts
export type ClockHead = ClockLink[];
```

### What Does Multiple Heads Mean?

**Multiple heads represent concurrent branches** in the CRDT's event history.

**Scenario**: Two clients make changes while offline, then sync:

```
Time 0: Both clients have head = [A]

Client 1 (offline):
  → Puts document → head = [A, B]
  
Client 2 (offline):
  → Puts document → head = [A, C]
  
Time 1: Both clients come online

Client 1 metadata: head = [B] (after A)
Client 2 metadata: head = [C] (after A)

Both have:
  - Same parent: A
  - Different heads: B vs C
  - No knowledge of each other's changes
```

### How Multiple Heads Are Merged

When metadata with multiple heads is received:

1. **Sort heads**: Heads are sorted for consistent comparison
2. **Advance clock**: The CRDT clock is advanced using `@web3-storage/pail`'s `advance()` function
3. **Merge state**: The CRDT's `root()` function computes merged state from all heads
4. **Single head**: After merge, the head typically becomes a single CID pointing to the merged state

**Location**: `core/base/crdt-clock.ts`

```122:163:core/base/crdt-clock.ts
  async int_applyHead(newHead: ClockHead, prevHead: ClockHead, localUpdates: boolean) {
    // ... validation ...
    const ogHead = sortClockHead(this.head);
    newHead = sortClockHead(newHead);
    if (compareClockHeads(ogHead, newHead)) {
      return;
    }
    // ... 
    const advancedHead = await advanceBlocks(this.logger, newHead, tblocks, this.head);
    const result = await root(toPailFetcher(tblocks), advancedHead);
    // ...
  }
```

**After Merge**:
- Both clients download each other's CAR files
- Both merge the concurrent branches
- Both converge to: `head = [merged_D]`
- State is consistent across both clients

### Visual Example

```
Initial:        A
                |
After offline:  A
               / \
              B   C  (concurrent branches)
               \ /
               D      (after merge)
```

**Head Progression**:
- `[A]` → Initial state
- `[B]` or `[C]` → Client 1 or Client 2's local head
- `[B, C]` → Metadata shows both heads (concurrent)
- `[D]` → After merge, single head

### When Do Multiple Heads Appear?

Multiple heads appear when:

1. **Concurrent Writes**: Two clients write at the same time without seeing each other's changes
2. **Network Partitions**: Clients can't communicate, make independent changes
3. **Sync Delays**: Client A writes, Client B writes before receiving A's metadata
4. **Initial Sync**: New client connecting to existing database with multiple concurrent branches

### Reading Multiple Heads

When querying changes, the system processes all heads:

```335:365:core/base/crdt-helpers.ts
async function gatherUpdates<T extends DocTypes>(
  blocks: BlockFetcher,
  eventsFetcher: EventFetcher<Operation>,
  head: ClockHead,
  since: ClockHead,
  updates: DocUpdate<T>[] = [],
  keys: Set<string>,
  didLinks: Set<string>,
  limit: number,
  logger: Logger,
): Promise<DocUpdate<T>[]> {
  if (limit <= 0) return updates;
  // ...
  for (const link of head) {
    if (didLinks.has(link.toString())) continue;
    didLinks.add(link.toString());
    const { value: event } = await eventsFetcher.get(link);
    // Process event...
    // Recursively process parent events
    if (event.parents) {
      updates = await gatherUpdates(blocks, eventsFetcher, event.parents, since, updates, keys, didLinks, limit, logger);
    }
  }
  return updates;
}
```

**Key Points**:
- All heads in the array are processed
- Parent events are recursively traversed
- Last-write-wins conflict resolution ensures consistency

## Metadata Synchronization Flow

### Upload (Local → Remote)

1. **Local Write**: Document saved locally, CAR file created
2. **Metadata Created**: `DbMeta` created with CAR file CID(s)
3. **Metadata Encrypted**: Metadata encrypted with ledger key
4. **Metadata Uploaded**: `reqPutMeta` sent to server
5. **WAL Enqueued**: Metadata added to WAL for reliable sync

**Location**: `core/base/crdt.ts`

```95:130:core/base/crdt.ts
  async bulk<T extends DocTypes>(updates: DocUpdate<T>[]): Promise<CRDTMeta> {
    // ... process updates ...
    const done = await this.blockstore.commitTransaction(tblocks, {
      head: advancedHead,
      meta: {
        head: advancedHead,
      },
    }, opts);
    // ... returns metadata with head ...
  }
```

### Download (Remote → Local)

1. **Request Metadata**: Client requests metadata via `reqGetMeta` or `bindGetMeta`
2. **Receive Metadata**: Server responds with `EventGetMeta` containing `DbMeta[]`
3. **Compare Heads**: Client compares remote clock head with local clock head
4. **Identify Missing**: Client identifies which CAR files need downloading
5. **Download CAR Files**: Client requests missing CAR files via `reqGetData`
6. **Merge State**: Client merges remote changes using `applyMeta()`

**Location**: `core/blockstore/loader.ts`

```502:543:core/blockstore/loader.ts
  async mergeDbMetaIntoClock(meta: DbMeta, activeStore: ActiveStore): Promise<CarGroup[]> {
    // ... deduplication ...
    const carHeader = await this.loadCarHeaderFromMeta<TransactionMeta>(meta, activeStore);
    // ... load referenced CAR files ...
    await this.ebOpts.applyMeta(carHeader.meta);
    return cgs;
  }
```

## Metadata Storage

### Client Side (Local)

- **IndexedDB**: Metadata stored in browser's IndexedDB
- **File System**: Metadata stored in files (Node.js)
- **Memory**: Metadata stored in memory (testing)

### Server Side

- **Encrypted Storage**: Servers store encrypted metadata
- **No Decryption**: Servers never see decrypted metadata or data
- **Blind Replication**: Servers can replicate metadata without understanding content

**Example Server Implementation**:

```364:406:fireproof-test/src/server/fireproof-backend.ts
  private async handlePutMeta(tenantInfo : TenantLedger, _ctx: any, msg: any): Promise<Response> {
    // ...
    if (meta && meta.metas) {
      const tenantLedgerMetaStore = await this.storage.loadMetaStore(tenantInfo);
      
      for (const metaEntry of meta.metas) {
        const key = `main/${metaEntry.cid}`;
        tenantLedgerMetaStore.set(key, {
          data: metaEntry.data,
          parents: metaEntry.parents
        });
      }
      
      await this.storage.saveMetaStore(tenantInfo, tenantLedgerMetaStore);
    }
    // ...
  }
```

## Key Insights

1. **Metadata is the Sync Coordinator**: Without metadata, clients don't know what data exists or what's missing

2. **Multiple Heads = Concurrent Branches**: Multiple CIDs in clock head means concurrent changes that need merging

3. **Automatic Merging**: The CRDT automatically merges concurrent branches into a single consistent state

4. **Efficient Sync**: Clients only download what they need by comparing metadata first

5. **Encrypted but Coordinated**: Metadata is encrypted, but its structure (CIDs, heads) enables efficient coordination

6. **Server-Blind**: Servers store metadata but can't decrypt it, enabling privacy-preserving sync

## Related Documentation

- **[CRDT and Conflict Resolution](./CRDT.md)**: How CRDTs merge concurrent changes
- **[Write-Ahead Log (WAL)](./WAL.md)**: How metadata is queued for reliable sync
- **[Network Topology](./NETWORK_TOPOLOGY.md)**: How clients and servers exchange metadata

