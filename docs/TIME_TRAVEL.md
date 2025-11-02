# Time-Travel Queries in Fireproof

Fireproof's CRDT clock system provides **out-of-the-box time-travel capabilities**. Every change is tracked with a clock head (vector clock position), allowing you to query the state of your database at any point in time. This is a unique feature that enables powerful historical analysis, debugging, auditing, and version-aware search.

## Table of Contents

- [How Time-Travel Works](#how-time-travel-works)
- [Basic Time-Travel Queries](#basic-time-travel-queries)
- [Time-Travel Vector Search](#time-travel-vector-search)
- [Historical State Reconstruction](#historical-state-reconstruction)
- [Use Cases](#use-cases)
- [Implementation Patterns](#implementation-patterns)
- [Advanced Techniques](#advanced-techniques)

## How Time-Travel Works

### The CRDT Clock as a Time Machine

Fireproof uses a **Merkle clock** (based on `@web3-storage/pail`) that tracks every change with a Content Identifier (CID). The clock head is a vector clock representing the current state:

```typescript
type ClockHead = ClockLink[];  // Array of event CIDs
```

**Key Insight**: Every document change creates a new clock entry. By specifying a clock head, you're effectively saying "show me the state of the database at this point in time."

### Important: Causal Ordering vs Time Ordering

**Critical Understanding**: 
- **Clocks are causally ordered, NOT time-ordered**
- Clock heads represent **sets of events** that have been seen/merged
- There is **no built-in timestamp** in clock heads or events
- To check if one clock is "before" another, you check if its events are included in the other clock (subset check)
- For time-based queries, you need to **store timestamps separately** and map them to clock heads

**Available Clock Comparison Functions** (from `core/base/crdt-clock.ts`):

```210:226:core/base/crdt-clock.ts
function sortClockHead(clockHead: ClockHead) {
  return clockHead.sort((a, b) => a.toString().localeCompare(b.toString()));
}

function compareClockHeads(head1: ClockHead, head2: ClockHead) {
  return head1.toString() === head2.toString();
}
```

- `compareClockHeads()`: Checks if two clock heads are **equal** (same events)
- `sortClockHead()`: Sorts clock heads for consistent comparison
- **No built-in "before/after" comparison**: Must implement based on subset checking (see examples below)

### Complete History Preservation

**Location**: `core/base/database.ts`

Every document change is preserved in Fireproof's CRDT:
- **Never deleted**: Old versions remain accessible via clock heads
- **Content-addressed**: Each state is identified by CID (immutable)
- **Causally ordered**: Clock heads represent a consistent snapshot

```128:138:core/base/database.ts
  async changes<T extends DocTypes>(since: ClockHead = [], opts: ChangesOptions = {}): Promise<ChangesResponse<T>> {
    await this.ready();
    this.logger.Debug().Any("since", since).Any("opts", opts).Msg("changes");
    const { result, head } = await this.ledger.crdt.changes(since, opts);
    const rows: ChangesResponseRow<T>[] = result.map(({ id: key, value, del, clock }) => ({
      key,
      value: (del ? { _id: key, _deleted: true } : { _id: key, ...value }) as DocWithId<T>,
      clock,
    }));
    return { rows, clock: head, name: this.name };
  }
```

The `since` parameter is your **time machine dial** - set it to any clock head to see changes from that point forward.

## Basic Time-Travel Queries

### 1. Get State at Specific Time

```typescript
// Current state
const current = await db.changes();

// State at specific clock head (historical point)
const historical = await db.changes(targetClockHead);

// Difference between two points in time
const diff = await db.changes(earlierClockHead);
// This shows all changes from 'earlier' to current state
```

### 2. Time-Travel Document Retrieval

**Location**: `core/base/crdt-clock.ts` and `core/base/crdt-helpers.ts`

The codebase provides `compareClockHeads()` for equality checks and `gatherUpdates()` shows how to check if events are included in a clock head. However, there's no built-in "isClockBeforeOrEqual" function - we need to implement it based on causality.

```typescript
/**
 * Check if a clock link (event CID) is included in a clock head
 * Based on gatherUpdates logic in crdt-helpers.ts
 */
function isClockLinkInHead(link: ClockLink, head: ClockHead): boolean {
  const headStrings = head.map(l => l.toString());
  return headStrings.includes(link.toString());
}

/**
 * Check if clock1 represents a state before or equal to clock2
 * This checks if all events in clock1 are included in clock2
 */
function isClockBeforeOrEqual(clock1: ClockHead, clock2: ClockHead): boolean {
  // If clocks are equal (using compareClockHeads logic)
  if (compareClockHeads(clock1, clock2)) {
    return true;
  }
  
  // Check if all events in clock1 are in clock2
  // If clock1 is a subset/prefix of clock2, it's "before"
  const clock2Strings = clock2.map(l => l.toString());
  return clock1.every(link => clock2Strings.includes(link.toString()));
}

/**
 * Compare clock heads for equality
 * From core/base/crdt-clock.ts:224
 */
function compareClockHeads(head1: ClockHead, head2: ClockHead): boolean {
  return head1.toString() === head2.toString();
}

async function getDocumentAtTime(
  db: Database, 
  docId: string, 
  atClock: ClockHead
): Promise<DocWithId<any> | null> {
  // Use changes() with empty array to get all changes
  const allChanges = await db.changes([], { limit: Infinity });
  
  // Filter to changes that happened at/before target clock
  // A change is "at" the target clock if its clock link is included in the clock head
  const relevantChanges = allChanges.rows.filter(row => {
    if (!row.clock) return false;
    // Check if this change's clock link is in the target clock head
    return isClockLinkInHead(row.clock, atClock);
  });
  
  // Find the document's state at that time
  const docChange = relevantChanges
    .filter(r => r.key === docId)
    .reverse()[0];  // Latest change before/at target time
  
  if (docChange?.value._deleted) {
    return null;  // Document was deleted
  }
  
  return docChange?.value;
}
```

**Note**: Clock heads are **causally ordered**, not time-ordered. A clock head represents a set of events that have been seen/merged. To check if an event happened "before" another, you need to check if it's an ancestor in the causal graph, not compare timestamps (since clocks don't contain timestamps).

### 3. Compare Two Points in Time

```typescript
/**
 * Compare two clock heads for equality
 * From core/base/crdt-clock.ts:224-226
 */
function compareClockHeads(head1: ClockHead, head2: ClockHead): boolean {
  return head1.toString() === head2.toString();
}

/**
 * Check if a clock link is included in a clock head
 * Based on gatherUpdates logic from core/base/crdt-helpers.ts:348-352
 */
function isClockLinkInHead(link: ClockLink, head: ClockHead): boolean {
  const headStrings = head.map(l => l.toString());
  return headStrings.includes(link.toString());
}

async function compareStates(
  db: Database,
  clock1: ClockHead,
  clock2: ClockHead
) {
  // Changes from clock1 to clock2 (using built-in changes() API)
  const changes1to2 = await db.changes(clock1);
  
  // Get all changes to reconstruct states
  const allChanges = await db.changes([], { limit: Infinity });
  
  // Build state at clock1 (all changes with clock links in clock1)
  const stateAtClock1 = new Map<string, DocWithId<any>>();
  for (const row of allChanges.rows) {
    if (row.clock && isClockLinkInHead(row.clock, clock1)) {
      if (row.value._deleted) {
        stateAtClock1.delete(row.key);
      } else {
        stateAtClock1.set(row.key, row.value);
      }
    }
  }
  
  // Build state at clock2
  const stateAtClock2 = new Map<string, DocWithId<any>>();
  for (const row of allChanges.rows) {
    if (row.clock && isClockLinkInHead(row.clock, clock2)) {
      if (row.value._deleted) {
        stateAtClock2.delete(row.key);
      } else {
        stateAtClock2.set(row.key, row.value);
      }
    }
  }
  
  // Compare states
  return {
    added: Array.from(stateAtClock2.keys()).filter(id => !stateAtClock1.has(id)),
    removed: Array.from(stateAtClock1.keys()).filter(id => !stateAtClock2.has(id)),
    modified: Array.from(stateAtClock2.entries()).filter(([id, doc2]) => {
      const doc1 = stateAtClock1.get(id);
      return doc1 && JSON.stringify(doc1) !== JSON.stringify(doc2);
    }).map(([id]) => id)
  };
}
```

## Time-Travel Vector Search

### Concept: Vector Search at Historical Points

The magic of time-travel vector search: **Query embeddings from the past**.

**Why it's powerful**:
- Documents change over time (embeddings change)
- Historical queries: "What were similar articles last month?"
- Trend analysis: "How did similar documents evolve?"
- A/B testing: Compare search results across time

### Implementation Pattern

```typescript
class TimeTravelVectorSearch {
  constructor(
    private db: Database,
    private vectorIndex: VectorIndexService
  ) {}
  
  /**
   * Search for similar documents at a specific point in time
   */
  async searchAtTime(
    queryEmbedding: number[],
    atClock: ClockHead,
    k: number = 10
  ) {
    // 1. Get all documents as they existed at target clock
    const historicalDocs = await this.getDocumentsAtClock(atClock);
    
    // 2. Extract embeddings that existed at that time
    const historicalEmbeddings = historicalDocs
      .filter(doc => doc.embedding)
      .map(doc => ({
        id: doc._id,
        embedding: doc.embedding,
        doc: doc
      }));
    
    // 3. Build temporary vector index from historical embeddings
    const tempIndex = this.vectorIndex.createTemporary(historicalEmbeddings);
    
    // 4. Search in historical space
    const results = await tempIndex.search(queryEmbedding, k);
    
    // 5. Return results with historical documents
    return results.map(result => ({
      similarity: result.similarity,
      doc: historicalDocs.find(d => d._id === result.id),
      clock: atClock
    }));
  }
  
  /**
   * Compare search results across multiple time points
   */
  async searchAcrossTime(
    queryEmbedding: number[],
    timePoints: ClockHead[],
    k: number = 10
  ) {
    const results = await Promise.all(
      timePoints.map(clock => this.searchAtTime(queryEmbedding, clock, k))
    );
    
    return {
      query: queryEmbedding,
      results: timePoints.map((clock, i) => ({
        clock,
        timestamp: this.clockToTimestamp(clock),
        topResults: results[i]
      }))
    };
  }
  
  private async getDocumentsAtClock(clock: ClockHead): Promise<DocWithId<any>[]> {
    // Get all changes
    const allChanges = await this.db.changes([], { limit: Infinity });
    
    // Filter to changes that are part of the target clock
    // A change is "at" the clock if its clock link is in the clock head
    const relevantChanges = allChanges.rows.filter(row => {
      if (!row.clock) return false;
      // Check if this change's clock link is included in the target clock head
      const headStrings = clock.map(l => l.toString());
      return headStrings.includes(row.clock.toString());
    });
    
    // Reconstruct document state at that time
    const docsById = new Map<string, DocWithId<any>>();
    
    for (const row of relevantChanges) {
      if (row.value._deleted) {
        docsById.delete(row.key);
      } else {
        docsById.set(row.key, row.value);
      }
    }
    
    return Array.from(docsById.values());
  }
  
  /**
   * Convert clock to approximate timestamp
   * Note: Clocks don't contain timestamps - this requires external tracking
   */
  private clockToTimestamp(clock: ClockHead): Date {
    // This is a placeholder - you need to track timestamps separately
    // See ClockTimestampMapper pattern above
    return new Date(); // Fallback to current time
  }
}
```

### Example: Article Similarity Over Time

```typescript
// Track how article similarity changes over time
const articleId = "article-123";
const article = await db.get(articleId);
const articleEmbedding = article.embedding;

// Get clock heads at different times
const clocks = {
  january: [...],  // Clock head from January
  february: [...], // Clock head from February
  march: [...],     // Clock head from March
  current: await db.changes().then(r => r.clock)
};

// Find similar articles at each time point
const similarityTimeline = await Promise.all(
  Object.entries(clocks).map(async ([month, clock]) => {
    const similar = await timeTravelSearch.searchAtTime(
      articleEmbedding,
      clock,
      k: 5
    );
    
    return {
      month,
      clock,
      similarArticles: similar.map(r => ({
        id: r.doc._id,
        title: r.doc.title,
        similarity: r.similarity
      }))
    };
  })
);

// Analyze: How did similar articles change?
console.log("Similarity Evolution:");
similarityTimeline.forEach(({ month, similarArticles }) => {
  console.log(`${month}:`, similarArticles.map(a => a.title));
});
```

### Time-Travel Embedding Evolution

Track how document embeddings evolved:

```typescript
class EmbeddingEvolutionTracker {
  constructor(private db: Database) {}
  
  /**
   * Track how a document's embedding changed over time
   */
  async trackEmbeddingEvolution(docId: string) {
    const allChanges = await this.db.changes([], { limit: Infinity });
    
    // Find all versions of this document
    const docVersions = allChanges.rows
      .filter(r => r.key === docId && !r.value._deleted)
      .map(r => ({
        clock: r.clock,
        doc: r.value,
        embedding: r.value.embedding
      }));
    
    // Calculate similarity between consecutive versions
    const evolution = [];
    for (let i = 0; i < docVersions.length - 1; i++) {
      const current = docVersions[i];
      const next = docVersions[i + 1];
      
      if (current.embedding && next.embedding) {
        const similarity = cosineSimilarity(
          current.embedding,
          next.embedding
        );
        
        evolution.push({
          fromClock: current.clock,
          toClock: next.clock,
          similarity,
          change: next.doc.title !== current.doc.title ? 'content' : 'embedding-only'
        });
      }
    }
    
    return {
      docId,
      versions: docVersions.length,
      evolution,
      summary: {
        avgSimilarity: evolution.reduce((sum, e) => sum + e.similarity, 0) / evolution.length,
        majorChanges: evolution.filter(e => e.similarity < 0.8).length
      }
    };
  }
}
```

## Historical State Reconstruction

### Full Database Snapshot at Any Point

```typescript
class HistoricalSnapshot {
  constructor(private db: Database) {}
  
  /**
   * Reconstruct complete database state at a specific clock head
   */
  async getSnapshotAt(clock: ClockHead): Promise<Map<string, DocWithId<any>>> {
    const allChanges = await this.db.changes([], { limit: Infinity });
    
    // Filter to changes that are part of the target clock
    // Based on gatherUpdates logic: check if clock link is in head
    const relevantChanges = allChanges.rows.filter(row => {
      if (!row.clock) return false;
      const headStrings = clock.map(l => l.toString());
      return headStrings.includes(row.clock.toString());
    });
    
    // Build snapshot by replaying changes
    const snapshot = new Map<string, DocWithId<any>>();
    
    for (const row of relevantChanges) {
      if (row.value._deleted) {
        snapshot.delete(row.key);
      } else {
        snapshot.set(row.key, row.value);
      }
    }
    
    return snapshot;
  }
  
  /**
   * Get all indexes at a specific point in time
   */
  async getIndexesAt(clock: ClockHead): Promise<Map<string, IndexSnapshot>> {
    const snapshot = await this.getSnapshotAt(clock);
    const indexes = new Map<string, IndexSnapshot>();
    
    // Rebuild each index from historical documents
    for (const [indexName, indexFn] of this.db.indexers) {
      const indexEntries = [];
      
      for (const doc of snapshot.values()) {
        // Apply index function to historical document
        const entries = this.applyIndexFunction(doc, indexFn);
        indexEntries.push(...entries);
      }
      
      indexes.set(indexName, {
        name: indexName,
        entries: indexEntries,
        clock
      });
    }
    
    return indexes;
  }
}
```

### Point-in-Time Queries

```typescript
/**
 * Execute a query as it would have been at a specific time
 */
async function queryAtTime<T extends DocTypes>(
  db: Database,
  indexName: string,
  queryOpts: QueryOpts<any>,
  atClock: ClockHead
): Promise<IndexRows<T, any, any>> {
  // Get snapshot at target time
  const snapshot = new HistoricalSnapshot(db);
  const state = await snapshot.getSnapshotAt(atClock);
  
  // Get index at target time
  const indexes = await snapshot.getIndexesAt(atClock);
  const index = indexes.get(indexName);
  
  if (!index) {
    throw new Error(`Index ${indexName} not found at target clock`);
  }
  
  // Execute query on historical index
  return applyQueryToIndex(index, queryOpts);
}
```

## Use Cases

### 1. Content Auditing: "Who Changed What When?"

```typescript
async function auditTrail(db: Database, docId: string) {
  const allChanges = await db.changes([], { limit: Infinity });
  
  const history = allChanges.rows
    .filter(r => r.key === docId)
    .map(r => ({
      clock: r.clock,
      timestamp: clockToTimestamp(r.clock),
      action: r.value._deleted ? 'deleted' : 'modified',
      value: r.value._deleted ? null : r.value
    }));
  
  return {
    docId,
    history,
    created: history[0],
    lastModified: history[history.length - 1],
    versions: history.length
  };
}
```

### 2. Trend Analysis: "How Did Similarity Change?"

```typescript
// Track how document clusters evolved
async function analyzeSimilarityTrends(
  db: Database,
  timePoints: ClockHead[]
) {
  const clustersByTime = await Promise.all(
    timePoints.map(async (clock) => {
      const snapshot = await getSnapshotAt(db, clock);
      const embeddings = Array.from(snapshot.values())
        .filter(doc => doc.embedding)
        .map(doc => ({
          id: doc._id,
          embedding: doc.embedding
        }));
      
      // Cluster embeddings (k-means, etc.)
      const clusters = performClustering(embeddings);
      
      return {
        clock,
        timestamp: clockToTimestamp(clock),
        clusters: clusters.map(c => ({
          centroid: c.centroid,
          documentIds: c.documents.map(d => d.id),
          size: c.documents.length
        }))
      };
    })
  );
  
  // Analyze cluster evolution
  return {
    timeline: clustersByTime,
    analysis: {
      stableClusters: findStableClusters(clustersByTime),
      newClusters: findNewClusters(clustersByTime),
      dissolvedClusters: findDissolvedClusters(clustersByTime)
    }
  };
}
```

### 3. A/B Testing Search Results

```typescript
// Compare search results from different time periods
async function compareSearchResults(
  db: Database,
  queryEmbedding: number[],
  variantA: ClockHead,  // Control: current algorithm
  variantB: ClockHead   // Treatment: new algorithm
) {
  const resultsA = await timeTravelSearch.searchAtTime(
    queryEmbedding,
    variantA,
    k: 20
  );
  
  const resultsB = await timeTravelSearch.searchAtTime(
    queryEmbedding,
    variantB,
    k: 20
  );
  
  // Analyze differences
  return {
    overlap: calculateOverlap(resultsA, resultsB),
    rankChanges: calculateRankChanges(resultsA, resultsB),
    newResults: resultsB.filter(r => !resultsA.find(a => a.doc._id === r.doc._id)),
    droppedResults: resultsA.filter(r => !resultsB.find(b => b.doc._id === r.doc._id))
  };
}
```

### 4. Version-Aware Recommendations

```typescript
// "Users who liked this also liked..." at specific time
async function historicalRecommendations(
  db: Database,
  itemId: string,
  atClock: ClockHead
) {
  // Get item at target time
  const item = await getDocumentAtTime(db, itemId, atClock);
  
  // Get all user interactions up to that time
  const interactions = await getSnapshotAt(db, atClock);
  const relevantInteractions = Array.from(interactions.values())
    .filter(doc => doc.type === 'interaction' && doc.itemId === itemId);
  
  // Find similar items based on historical data
  const similarItems = await findSimilarItems(
    item,
    relevantInteractions,
    atClock
  );
  
  return {
    item,
    atTime: clockToTimestamp(atClock),
    recommendations: similarItems
  };
}
```

### 5. Debugging: "What Was the State When Bug Occurred?"

```typescript
// Reproduce bug by querying state at time of error
async function debugAtTime(
  db: Database,
  errorClock: ClockHead,
  errorDocId: string
) {
  const snapshot = await getSnapshotAt(db, errorClock);
  const errorDoc = snapshot.get(errorDocId);
  
  // Reconstruct exact state when error occurred
  return {
    errorClock,
    errorDoc,
    relatedDocs: findRelatedDocs(errorDoc, snapshot),
    indexes: await getIndexesAt(db, errorClock),
    fullState: snapshot
  };
}
```

## Implementation Patterns

### Pattern 1: Clock Head Timestamp Mapping

**Important**: Fireproof's CRDT clocks are **causally ordered, not time-ordered**. There is no built-in timestamp in clock heads or events. To map clocks to timestamps, you need to store timestamps separately.

```typescript
/**
 * Clock-Timestamp Mapper
 * 
 * Note: Fireproof clocks don't contain timestamps. You need to:
 * 1. Store timestamps when changes occur
 * 2. Map clock heads to the timestamp when they were created
 * 3. Use this mapping for time-based queries
 */
class ClockTimestampMapper {
  private clockToTime = new Map<string, Date>();
  private timeToClock = new Map<number, ClockHead[]>();
  
  /**
   * Record a clock head with current timestamp
   * Call this whenever you get a clock head from changes()
   */
  recordClock(clock: ClockHead, timestamp?: Date) {
    const clockKey = this.clockKey(clock);
    const time = timestamp || new Date();
    this.clockToTime.set(clockKey, time);
    
    // Also map time → clock (multiple clocks can have same time)
    const timeKey = time.getTime();
    if (!this.timeToClock.has(timeKey)) {
      this.timeToClock.set(timeKey, []);
    }
    this.timeToClock.get(timeKey)!.push(clock);
  }
  
  /**
   * Get timestamp for a clock head
   */
  async mapClockToTimestamp(db: Database, clock: ClockHead): Promise<Date | null> {
    const clockKey = this.clockKey(clock);
    
    // Check cache
    if (this.clockToTime.has(clockKey)) {
      return this.clockToTime.get(clockKey)!;
    }
    
    // Try to find timestamp by looking at when changes at this clock occurred
    // This requires having recorded clocks previously
    // For now, return current time as fallback (not ideal)
    const now = new Date();
    this.clockToTime.set(clockKey, now);
    return now;
  }
  
  /**
   * Find clock heads closest to a target time
   */
  async findClockByTimestamp(targetTime: Date): Promise<ClockHead[]> {
    const targetTimeKey = targetTime.getTime();
    
    // Find closest recorded time
    let closestTime: number | null = null;
    let minDiff = Infinity;
    
    for (const timeKey of this.timeToClock.keys()) {
      const diff = Math.abs(timeKey - targetTimeKey);
      if (diff < minDiff) {
        minDiff = diff;
        closestTime = timeKey;
      }
    }
    
    if (closestTime === null) {
      return [];
    }
    
    return this.timeToClock.get(closestTime) || [];
  }
  
  private clockKey(clock: ClockHead): string {
    // Sort and join for consistent key
    return clock.map(c => c.toString()).sort().join(',');
  }
}

// Usage: Track clocks as they occur
const mapper = new ClockTimestampMapper();

// When you get changes, record the clock
const changes = await db.changes();
mapper.recordClock(changes.clock);

// Later: Find clock at specific time
const clocksAtTime = await mapper.findClockByTimestamp(new Date('2024-01-15'));
```

**Alternative**: Store timestamps in documents themselves:

```typescript
// Store timestamp with each document
await db.put({
  _id: "doc-1",
  title: "Example",
  _timestamp: new Date().toISOString(),
  _clock: changes.clock  // Store clock head with document
});

// Query documents by timestamp (using Fireproof index)
const docsAtTime = await db.query('timestamp', {
  range: [
    new Date('2024-01-15').toISOString(),
    new Date('2024-01-16').toISOString()
  ]
});

// Extract clock heads from results
const clocksAtTime = docsAtTime.rows
  .map(r => r.value._clock)
  .filter(Boolean);
```

### Pattern 2: Incremental Snapshot Building

```typescript
class IncrementalSnapshotBuilder {
  private snapshots = new Map<string, Map<string, DocWithId<any>>>();
  
  /**
   * Build snapshot incrementally (more efficient than full rebuild)
   */
  async buildSnapshotIncremental(
    baseClock: ClockHead,
    targetClock: ClockHead
  ): Promise<Map<string, DocWithId<any>>> {
    // Get base snapshot (cached)
    const baseSnapshot = this.snapshots.get(baseClock.toString()) ||
      await this.buildFullSnapshot(baseClock);
    
    // Get changes from base to target
    const changes = await db.changes(baseClock);
    
    // Apply changes incrementally
    const targetSnapshot = new Map(baseSnapshot);
    
    for (const row of changes.rows) {
      if (row.value._deleted) {
        targetSnapshot.delete(row.key);
      } else {
        targetSnapshot.set(row.key, row.value);
      }
    }
    
    // Cache target snapshot
    this.snapshots.set(targetClock.toString(), targetSnapshot);
    
    return targetSnapshot;
  }
}
```

### Pattern 3: Time-Travel Index Caching

```typescript
class CachedTimeTravelIndex {
  private indexCache = new Map<string, IndexSnapshot>();
  
  async getIndexAt(clock: ClockHead): Promise<IndexSnapshot> {
    const clockKey = clock.map(c => c.toString()).join(',');
    
    if (this.indexCache.has(clockKey)) {
      return this.indexCache.get(clockKey)!;
    }
    
    // Build index from snapshot
    const snapshot = await getSnapshotAt(db, clock);
    const index = await buildIndexFromSnapshot(snapshot);
    
    this.indexCache.set(clockKey, index);
    return index;
  }
}
```

## Advanced Techniques

### 1. Time-Travel Vector Index Compression

Store historical vector indexes efficiently:

```typescript
// Store only deltas between time points
class CompressedTimeTravelVectorIndex {
  async getIndexAt(clock: ClockHead): Promise<VectorIndex> {
    // Find closest earlier snapshot
    const baseClock = await this.findClosestSnapshot(clock);
    const baseIndex = await this.loadIndex(baseClock);
    
    // Apply deltas from base to target
    const changes = await db.changes(baseClock);
    const delta = this.computeIndexDelta(changes);
    
    // Apply delta to base index
    return this.applyDelta(baseIndex, delta);
  }
}
```

### 2. Time-Range Queries

```typescript
/**
 * Query documents that existed during a time range
 */
async function queryTimeRange(
  db: Database,
  startClock: ClockHead,
  endClock: ClockHead
): Promise<DocWithId<any>[]> {
  // Get start snapshot
  const startSnapshot = await getSnapshotAt(db, startClock);
  
  // Get changes during range
  const changes = await db.changes(startClock);
  const endSnapshot = await getSnapshotAt(db, endClock);
  
  // Find documents that existed throughout range
  const persistedDocs = Array.from(startSnapshot.keys()).filter(id => {
    return endSnapshot.has(id) && !changes.rows.find(r => 
      r.key === id && r.value._deleted
    );
  });
  
  return persistedDocs.map(id => endSnapshot.get(id)!);
}
```

### 3. Temporal Clustering

```typescript
/**
 * Cluster documents based on temporal similarity
 * (Documents that changed at similar times)
 */
async function temporalClustering(
  db: Database,
  timeWindow: number
): Promise<Cluster[]> {
  const allChanges = await db.changes([], { limit: Infinity });
  
  // Group changes by time windows
  const timeGroups = new Map<number, string[]>();
  
  for (const row of allChanges.rows) {
    if (!row.clock) continue;
    const timestamp = clockToTimestamp(row.clock);
    const window = Math.floor(timestamp / timeWindow);
    
    if (!timeGroups.has(window)) {
      timeGroups.set(window, []);
    }
    timeGroups.get(window)!.push(row.key);
  }
  
  // Documents that changed together are temporally related
  return Array.from(timeGroups.entries()).map(([window, docIds]) => ({
    timeWindow: window,
    documents: docIds,
    size: docIds.length
  }));
}
```

## Time-Travel File Attachments

### Are File Attachments Preserved in History?

**Yes!** File attachments from past document versions are preserved and accessible.

### How It Works

**Location**: `core/base/crdt-helpers.ts`

When a document is saved with files:
1. **Files are encoded** and stored separately by CID (content-addressed)
2. **File metadata** (CID, type, size) is stored in the document block
3. **Each document version** is a separate immutable block with its own file metadata

When retrieving historical documents:
1. Document block is loaded from the CRDT at that clock head
2. **`readFiles()` is called** (line 290) which attaches file metadata with async getters
3. File CIDs in `_files` and `_publicFiles` are preserved from that version

```242:250:core/base/crdt-helpers.ts
export function readFiles<T extends DocTypes>(blocks: BaseBlockstore, { doc }: Partial<DocValue<T>>) {
  if (!doc) return;
  if (doc._files) {
    readFileset(blocks as EncryptedBlockstore, doc._files);
  }
  if (doc._publicFiles) {
    readFileset(blocks as EncryptedBlockstore, doc._publicFiles, true);
  }
}
```

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

### Accessing Historical File Attachments

```typescript
/**
 * Get document with files at a specific clock head
 */
async function getDocumentWithFilesAtTime(
  db: Database,
  docId: string,
  atClock: ClockHead
): Promise<DocWithId<any> | null> {
  // Get historical document
  const doc = await getDocumentAtTime(db, docId, atClock);
  
  if (!doc) return null;
  
  // Files are already attached via readFiles()
  // File metadata is in _files and _publicFiles with async getters
  if (doc._files) {
    console.log('Historical files:', Object.keys(doc._files));
    
    // Access file metadata
    for (const filename in doc._files) {
      const fileMeta = doc._files[filename] as DocFileMeta;
      console.log(`File: ${filename}`, {
        cid: fileMeta.cid.toString(),
        type: fileMeta.type,
        size: fileMeta.size
      });
      
      // Load the actual file (lazy loading)
      if (fileMeta.file) {
        const file = await fileMeta.file();
        console.log('File loaded:', file.name, file.size);
      }
    }
  }
  
  return doc;
}
```

### Example: Track File Changes Over Time

```typescript
/**
 * Track how files changed for a document over time
 */
async function trackFileHistory(
  db: Database,
  docId: string
): Promise<FileHistory[]> {
  const allChanges = await db.changes([], { limit: Infinity });
  
  // Get all versions of this document
  const docVersions = allChanges.rows
    .filter(r => r.key === docId && !r.value._deleted)
    .map(r => ({
      clock: r.clock,
      doc: r.value,
      files: r.value._files || {},
      publicFiles: r.value._publicFiles || {}
    }));
  
  // Extract file information from each version
  return docVersions.map(version => ({
    clock: version.clock,
    files: Object.keys(version.files).map(filename => {
      const meta = version.files[filename] as DocFileMeta;
      return {
        filename,
        cid: meta.cid?.toString(),
        type: meta.type,
        size: meta.size,
        lastModified: meta.lastModified
      };
    }),
    publicFiles: Object.keys(version.publicFiles).map(filename => {
      const meta = version.publicFiles[filename] as DocFileMeta;
      return {
        filename,
        cid: meta.cid?.toString(),
        type: meta.type,
        size: meta.size,
        url: meta.url  // Public files have URLs
      };
    })
  }));
}

// Usage
const fileHistory = await trackFileHistory(db, "article-123");
fileHistory.forEach(({ clock, files }) => {
  console.log(`At clock ${clock}:`, files.map(f => f.filename));
});
```

### Important Considerations

1. **File Blocks Must Exist**: Historical files are accessible **only if** the file blocks (identified by CID) still exist in the file store. If files are deleted from storage, they won't be accessible even if document metadata references them.

2. **Content-Addressed Storage**: Since files are stored by CID (hash), the same file content always has the same CID. If a file is removed and re-added with identical content, it will have the same CID and be accessible.

3. **File Store vs CAR Store**: 
   - **File Store**: Raw file attachments (by CID)
   - **CAR Store**: Document data blocks
   - Both must be preserved for full time-travel

4. **Lazy Loading**: Files are not loaded automatically - they're provided as async getter functions (`fileMeta.file()`). This allows accessing files on-demand without loading all files upfront.

### Complete Example: Time-Travel with Files

```typescript
/**
 * Get complete document state (including files) at a specific time
 */
async function getCompleteDocumentAtTime(
  db: Database,
  docId: string,
  atClock: ClockHead
) {
  // 1. Get document at that time
  const doc = await getDocumentAtTime(db, docId, atClock);
  if (!doc) return null;
  
  // 2. Load files if they exist
  const files: Record<string, File> = {};
  const publicFiles: Record<string, { file: File; url?: string }> = {};
  
  if (doc._files) {
    for (const filename in doc._files) {
      const fileMeta = doc._files[filename] as DocFileMeta;
      if (fileMeta.file) {
        try {
          files[filename] = await fileMeta.file();
        } catch (error) {
          console.warn(`Failed to load file ${filename}:`, error);
          // File block may not exist in storage
        }
      }
    }
  }
  
  if (doc._publicFiles) {
    for (const filename in doc._publicFiles) {
      const fileMeta = doc._publicFiles[filename] as DocFileMeta;
      if (fileMeta.file) {
        try {
          const file = await fileMeta.file();
          publicFiles[filename] = {
            file,
            url: fileMeta.url  // Public files have URLs
          };
        } catch (error) {
          console.warn(`Failed to load public file ${filename}:`, error);
        }
      }
    }
  }
  
  return {
    doc,
    files,
    publicFiles,
    clock: atClock
  };
}
```

## Best Practices

1. **Cache Snapshots**: Historical snapshots are expensive to compute - cache them
2. **Incremental Updates**: Build snapshots incrementally from base points
3. **Clock Head Storage**: Store important clock heads (milestones, releases) for quick access
4. **Selective History**: Not all documents need full history - mark important ones
5. **Compression**: Use delta encoding for historical vector indexes
6. **File Preservation**: Ensure file store blocks are not deleted if you need historical file access
7. **Lazy Loading**: Only load files when needed (use async getters) to avoid memory issues

## Related Documentation

- **[CRDT and Conflict Resolution](./CRDT.md)**: Understanding the clock system
- **[Metadata and Synchronization](./METADATA_SYNC.md)**: How clock heads work
- **[Changes API and Advanced Indexing](./CHANGES_API.md)**: Using changes() for queries
- **[Advanced Indexing Projects](./ADVANCED_INDEXING.md)**: Vector search integration

## Conclusion

Time-travel is not just a feature in Fireproof - it's a **fundamental capability** enabled by the CRDT clock system. Every change is preserved and queryable, enabling:

- ✅ Historical analysis
- ✅ Debugging at point-in-time
- ✅ Temporal search (vector and structured)
- ✅ Trend analysis
- ✅ Version-aware recommendations
- ✅ Complete audit trails

This is a **unique advantage** that other databases don't provide out of the box. Use it to build powerful time-aware applications!

