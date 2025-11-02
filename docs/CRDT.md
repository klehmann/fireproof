# CRDT (Conflict-Free Replicated Data Type) in Fireproof

Fireproof uses a **Merkle clock-based CRDT** implemented via the `@web3-storage/pail` library to enable conflict-free replication across multiple devices and writers, even when data is encrypted end-to-end.

## Overview

Fireproof's CRDT implementation provides:
- **Automatic Conflict Resolution**: No manual conflict resolution needed
- **Eventual Consistency**: All replicas converge to the same state
- **Encrypted Merging**: Merges work correctly even with end-to-end encryption
- **Causal Ordering**: Operations maintain causal relationships
- **Multi-Writer Support**: Multiple devices can write simultaneously

## CRDT Type: Merkle Clock-Based

Fireproof uses a **Merkle clock-based CRDT** from `@web3-storage/pail`:

**Key Characteristics**:
- **Merkle Tree Structure**: Hash-based DAG (Directed Acyclic Graph)
- **Vector Clocks**: Causal ordering of operations
- **Event-Based**: Each operation is an event in the clock
- **Commutative, Associative, Idempotent**: Operations can be applied in any order

**Library**: `@web3-storage/pail` (v0.6.2)

```1:2:core/base/crdt-clock.ts
import { advance } from "@web3-storage/pail/clock";
import { root } from "@web3-storage/pail/crdt";
```

## Architecture: Dual-Layer CRDT

Fireproof employs a **dual-layer CRDT architecture**:

### Inner CRDT (Encrypted Data)

**Purpose**: Manages the actual document data

**Characteristics**:
- **Encrypted**: All blocks encrypted with symmetric keys
- **Content-Addressed**: Blocks identified by CID (hash)
- **Immutable**: Each update creates new blocks
- **Event Log**: Operations stored as events in Merkle tree

**Location**: `core/base/crdt.ts`, `core/base/crdt-helpers.ts`

### Outer CRDT (Metadata)

**Purpose**: Manages pointers and metadata about encrypted payloads

**Characteristics**:
- **Unencrypted Metadata**: Contains pointers to encrypted blocks
- **Clock Head**: Vector clock head points to latest state
- **Snapshot Pointers**: Links to encrypted CRDT snapshots
- **Synchronization**: Enables merging across clients

**Location**: Meta store, WAL queue

## How It Works

### 1. Document Operations

When a document is written:

```162:175:core/base/crdt.ts
async bulk<T extends DocTypes>(updates: DocUpdate<T>[]): Promise<CRDTMeta> {
  await this.ready();
  updates = updates.map((dupdate: DocUpdate<T>) => ({
    ...dupdate,
    value: sanitizeDocumentFields(dupdate.value),
  }));

  if (this.clock.head.length === 0) {
    // INJECT GENESIS Block
    const value = { id: PARAM.GENESIS_CID, value: { _id: PARAM.GENESIS_CID } };
    await this._bulk([value]);
  }
  return await this._bulk(updates);
}
```

**Process**:
1. **Sanitize**: Clean document fields (handle Date objects, etc.)
2. **Genesis Block**: Create if first write
3. **Apply to CRDT**: Call `applyBulkUpdateToCrdt()` which uses `pail.put()`

### 2. CRDT Operations (from @web3-storage/pail)

**Put Operation**:

```111:142:core/base/crdt-helpers.ts
export async function applyBulkUpdateToCrdt<T extends DocTypes>(
  store: StoreRuntime,
  tblocks: CarTransaction,
  head: ClockHead,
  updates: DocUpdate<T>[],
  logger: Logger,
): Promise<CRDTMeta> {
  let result: Result | null = null;
  if (updates.length > 1) {
    const batch = await Batch.create(toPailFetcher(tblocks), head);
    for (const update of updates) {
      const link = await writeDocContent(store, tblocks, update, logger);
      await batch.put(toString(update.id, logger), link);
    }
    result = await batch.commit();
  } else if (updates.length === 1) {
    const link = await writeDocContent(store, tblocks, updates[0], logger);
    result = await put(toPailFetcher(tblocks), head, toString(updates[0].id, logger), link);
  }
  if (!result) throw logger.Error().Uint64("updates.len", updates.length).Msg("Missing result").AsError();

  if (result.event) {
    for (const block of [
      ...result.additions,
      result.event,
    ]) {
      tblocks.putSync(await anyBlock2FPBlock(block));
    }
  }
  return { head: result.head };
}
```

**What Happens**:
1. **Document Encoded**: Document serialized and encrypted as block
2. **CID Generated**: Content-addressed identifier created
3. **CRDT Put**: `pail.put(key, link)` called with document ID and CID
4. **Event Created**: New event in Merkle clock with operation
5. **Head Updated**: Clock head updated to point to new event

### 3. Clock Head and Vector Clocks

The **clock head** is a vector clock representing the current state:

```24:25:core/base/crdt-clock.ts
readonly head: ClockHead = [];
```

**ClockHead Structure**:
```typescript
type ClockHead = AnyLink[];  // Array of event CIDs
```

**Properties**:
- **Multiple Heads**: Can have multiple heads (concurrent branches)
- **Causal Ordering**: Events maintain parent-child relationships
- **Convergence**: All heads eventually merge to single state

### 4. Merging Process

When remote changes arrive:

```122:180:core/base/crdt-clock.ts
async int_applyHead(newHead: ClockHead, prevHead: ClockHead, localUpdates: boolean) {
  // Sort heads for comparison
  const ogHead = sortClockHead(this.head);
  newHead = sortClockHead(newHead);
  
  // Skip if already merged
  if (compareClockHeads(ogHead, newHead)) {
    return;
  }
  
  // Advance clock with new events
  const advancedHead = await advanceBlocks(this.logger, newHead, tblocks, this.head);
  
  // Compute root (merge result)
  const result = await root(toPailFetcher(tblocks), advancedHead);
  
  // Add any new blocks to transaction
  const fpBlocks = await Promise.all(result.additions.map(anyBlock2FPBlock));
  for (const fp of fpBlocks) {
    tblocks.putSync(fp);
  }
  
  // Commit and update head
  if (!noLoader) {
    await this.blockstore.commitTransaction(tblocks, { head: advancedHead }, { add: false, noLoader });
  }
  this.setHead(advancedHead);
}
```

**Merge Steps**:
1. **Advance Clock**: `advance()` adds new events to clock
2. **Compute Root**: `root()` merges all heads to single state
3. **Load Blocks**: Fetch any missing blocks
4. **Update Head**: Set new merged head

**Key Functions**:
- `advance()`: Adds events to clock (from @web3-storage/pail)
- `root()`: Computes merged state from clock head (from @web3-storage/pail)

## Conflict Resolution

### Automatic Resolution (Last-Write-Wins)

Fireproof uses **last-write-wins** semantics based on **causal ordering**:

**How It Works**:
1. **Event Ordering**: Events ordered by causal relationship (parent-child)
2. **Per-Document**: Conflicts resolved per document ID
3. **Latest Event Wins**: When multiple events update same document, most recent (by causal order) wins
4. **Automatic**: No user intervention needed

**Example**:

```
Device A: put("doc1", { text: "Hello" })
Device B: put("doc1", { text: "World" })

Result: Both events in clock, latest wins when reading
```

### Gathering Updates

When reading, updates are gathered from clock:

```335:384:core/base/crdt-helpers.ts
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
  // ... iterate through events in clock
  for (const link of head) {
    const { value: event } = await eventsFetcher.get(link);
    if (!event) continue;
    const { type } = event.data;
    let ops = [] as PutOperation[];
    if (type === "batch") {
      ops = event.data.ops as PutOperation[];
    } else if (type === "put") {
      ops = [event.data] as PutOperation[];
    }
    for (let i = ops.length - 1; i >= 0; i--) {
      const { key, value } = ops[i];
      if (!keys.has(key)) {
        // Only add first occurrence of key (latest wins)
        const docValue = await getValueFromLink<T>(blocks, value, logger);
        updates.push({ id: key, value: docValue.doc, del: docValue.del, clock: link });
        keys.add(key);  // Prevent duplicates
      }
    }
    // Recursively process parent events
    if (event.parents) {
      updates = await gatherUpdates(blocks, eventsFetcher, event.parents, since, updates, keys, didLinks, limit, logger);
    }
  }
  return updates;
}
```

**Conflict Resolution Logic**:
- **Reverse Iteration**: Process events from latest to earliest (`i >= 0; i--`)
- **First Occurrence Wins**: First time a document ID is seen is the latest version
- **Skip Duplicates**: `keys.has(key)` ensures only latest version included

### Manual Conflict Resolution

Fireproof does **not** currently support manual conflict resolution. All conflicts are resolved automatically using last-write-wins.

**Why Automatic**:
- **CRDT Properties**: Operations are designed to be commutative
- **Eventual Consistency**: All replicas converge to same state
- **Simplicity**: No user intervention needed

**Future Considerations**:
- Custom merge functions per document type
- Conflict callbacks for user-defined resolution
- Three-way merge strategies

## Encryption and Merging

### How Encryption Works with CRDTs

**Key Insight**: Encryption happens at the **block level**, not the operation level.

**Process**:
1. **Document → Block**: Document serialized to IPLD block
2. **Block Encrypted**: Block encrypted with symmetric key
3. **CID Generated**: Content-addressed identifier (hash of encrypted block)
4. **Operation Logged**: Put operation logged in CRDT (references CID)
5. **Merge on CIDs**: CRDT merges based on CIDs, not decrypted content

**Location**: `core/blockstore/commitor.ts` - Encryption happens during commit

### Why This Works

**CRDT Operations are CID-Based**:
- Operations reference encrypted blocks by CID
- Merge logic operates on CIDs, not content
- Decryption happens **after** merge, when reading

**Example Flow**:

```
Device A:
  1. put("doc1", {...}) → encrypt → CID_A → event_A
  2. Upload: event_A (points to CID_A)

Device B:
  1. put("doc1", {...}) → encrypt → CID_B → event_B
  2. Upload: event_B (points to CID_B)

Merge:
  1. Both events_A and event_B in clock
  2. root() computes: latest event wins (e.g., event_B)
  3. Read: fetch block at CID_B → decrypt → return document
```

**Important**: The server/storage never sees decrypted content, only encrypted blocks and event metadata.

### Decryption During Read

When reading documents, decryption happens **after** merge:

```282:292:core/base/crdt-helpers.ts
async function getValueFromLink<T extends DocTypes>(blocks: BlockFetcher, link: AnyLink, logger: Logger): Promise<DocValue<T>> {
  const block = await blocks.get(link);
  if (!block) throw logger.Error().Str("link", link.toString()).Msg(`Missing linked block`).AsError();
  const { value } = (await asyncBlockDecode({ bytes: block.bytes, hasher, codec })) as { value: DocValue<T> };
  const cvalue = {
    ...value,
    cid: link,
  };
  readFiles(blocks as EncryptedBlockstore, cvalue);
  return cvalue;
}
```

**Steps**:
1. **Get Block**: Fetch encrypted block by CID
2. **Decrypt**: Block decrypted during `asyncBlockDecode()`
3. **Return**: Decrypted document returned

## Synchronization Flow

### Local Write

```
User: db.put(doc)
  ↓
WriteQueue: batch operations
  ↓
CRDT.bulk(): apply to CRDT
  ↓
Blockstore: encrypt & commit
  ↓
Clock.applyHead(): update head (localUpdates=true)
  ↓
WAL.enqueue(): queue for sync
  ↓
WAL.process(): upload to remote
```

### Remote Sync

```
Remote: CAR file arrives
  ↓
Blockstore: decrypt & load blocks
  ↓
applyMeta(): get new head
  ↓
Clock.applyHead(): merge (localUpdates=false)
  ↓
root(): compute merged state
  ↓
notifyWatchers(): trigger subscriptions
```

### Merge on Sync

```105:119:core/base/crdt.ts
applyMeta: async (meta: TransactionMeta) => {
  const crdtMeta = meta as CRDTMeta;
  if (!crdtMeta.head) throw this.logger.Error().Msg("missing head").AsError();
  this.logger
    .Debug()
    .Str("newHead", crdtMeta.head.map((h) => h.toString()).join(","))
    .Msg("APPLY_META: Calling applyHead for REMOTE sync");
  await this.clock.applyHead(crdtMeta.head, []);
},
```

**What Happens**:
1. **New Head Received**: Remote head contains new events
2. **Advance Clock**: Add remote events to local clock
3. **Merge**: Compute root of merged clock
4. **Load Blocks**: Fetch any missing encrypted blocks
5. **Update Head**: Set merged head as current state

## CRDT Properties

### Commutativity

Operations can be applied in any order:

```
put("doc1", A) then put("doc1", B)
= 
put("doc1", B) then put("doc1", A)

Result: Same final state (B wins in both cases)
```

### Associativity

Multiple merges produce same result:

```
(merge(A, B), C)
=
(A, merge(B, C))

Result: Same final merged state
```

### Idempotency

Applying same operation multiple times has same effect:

```
merge(A, A) = A
```

### Eventual Consistency

All replicas converge to same state given:
- All events eventually delivered
- Network partitions eventually resolve
- All nodes process all events

## Performance Considerations

### Clock Growth

**Problem**: Clock can grow large with many operations

**Solutions**:
- **Compaction**: Merge old events (see compaction strategies)
- **Garbage Collection**: Remove unreferenced blocks
- **Head Pruning**: Keep only recent heads when possible

### Block Fetching

**Problem**: Merging requires fetching many blocks

**Optimizations**:
- **Caching**: Recently accessed blocks cached
- **Lazy Loading**: Blocks loaded only when needed
- **Batch Fetching**: Multiple blocks fetched in parallel

### Transaction Batching

**Optimization**: Multiple operations batched in single transaction:

```119:126:core/base/crdt-helpers.ts
if (updates.length > 1) {
  const batch = await Batch.create(toPailFetcher(tblocks), head);
  for (const update of updates) {
    const link = await writeDocContent(store, tblocks, update, logger);
    await batch.put(toString(update.id, logger), link);
  }
  result = await batch.commit();
}
```

## Related Files

- `core/base/crdt.ts` - CRDT implementation
- `core/base/crdt-clock.ts` - Clock management and merging
- `core/base/crdt-helpers.ts` - Helper functions for CRDT operations
- `@web3-storage/pail` - Underlying CRDT library

## Summary

Fireproof uses a **Merkle clock-based CRDT** from `@web3-storage/pail` that:

1. **Enables Conflict-Free Replication**: Multiple writers without conflicts
2. **Works with Encryption**: Merges on encrypted blocks (CIDs), decrypts on read
3. **Automatic Resolution**: Last-write-wins based on causal ordering
4. **Eventual Consistency**: All replicas converge to same state
5. **Commutative Operations**: Operations can be applied in any order

The dual-layer architecture (inner encrypted CRDT + outer metadata CRDT) allows secure, decentralized replication while maintaining automatic conflict resolution.

