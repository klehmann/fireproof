# Changes API and Advanced Indexing

The `db.changes()` API provides efficient change tracking and synchronization capabilities. Combined with Fireproof's powerful indexing system, you can build sophisticated query patterns similar to HCL Notes/Domino views and CouchDB map-reduce indexes.

## Changes API

### Overview

The `changes()` method returns all document changes since a given clock head (vector clock position). This is the foundation for:
- **Change replication**: Track what has changed for sync
- **Event sourcing**: Replay changes in order
- **Audit logs**: Track document history
- **Real-time updates**: Subscribe to changes as they happen

### Basic Usage

```typescript
import { fireproof } from '@fireproof/core';

const db = fireproof('my-app');

// Get all changes from the beginning
const allChanges = await db.changes();

// Get changes since a specific clock position
const recentChanges = await db.changes(sinceClock, { limit: 100 });

// With dirty reads (may include uncommitted changes)
const dirtyChanges = await db.changes(sinceClock, { dirty: true });
```

### Response Structure

**Location**: `core/types/base/types.ts`

```369:373:core/types/base/types.ts
export interface ChangesResponse<T extends DocTypes> {
  readonly clock: ClockHead;
  readonly rows: ChangesResponseRow<T>[];
  readonly name?: string;
}
```

**Example Response**:

```typescript
{
  clock: [{ "/": "bafyrei..." }],  // Current clock head
  rows: [
    {
      key: "doc1",
      value: { _id: "doc1", title: "Hello", value: 42 },
      clock: { "/": "bafyreia..." }  // Event CID for this change
    },
    {
      key: "doc2",
      value: { _id: "doc2", _deleted: true },  // Deletion
      clock: { "/": "bafyreib..." }
    },
    {
      key: "doc3",
      value: { _id: "doc3", title: "Updated", status: "active" },
      clock: { "/": "bafyreic..." }
    }
  ],
  name: "my-app"
}
```

### Row Structure

```363:367:core/types/base/types.ts
export interface ChangesResponseRow<T extends DocTypes> {
  readonly key: string;
  readonly value: DocWithId<T>;
  readonly clock?: ClockLink;
}
```

**Fields**:
- **`key`**: Document ID (`_id`)
- **`value`**: Full document or deletion marker
- **`clock`**: Event CID (Content Identifier) for this change

### Handling Deletions

Deletions are represented with `_deleted: true` marker:

```typescript
{
  key: "deleted-doc",
  value: { _id: "deleted-doc", _deleted: true },
  clock: { "/": "bafyrei..." }
}
```

**Location**: `core/base/database.ts`

```132:136:core/base/database.ts
    const rows: ChangesResponseRow<T>[] = result.map(({ id: key, value, del, clock }) => ({
      key,
      value: (del ? { _id: key, _deleted: true } : { _id: key, ...value }) as DocWithId<T>,
      clock,
    }));
```

**Processing Deletions**:

```typescript
const changes = await db.changes();

for (const row of changes.rows) {
  if (row.value._deleted) {
    console.log(`Document ${row.key} was deleted`);
    // Handle deletion (remove from UI, cleanup, etc.)
  } else {
    console.log(`Document ${row.key} was updated:`, row.value);
    // Handle update (sync to UI, process, etc.)
  }
}
```

### Change Options

```358:361:core/types/base/types.ts
export interface ChangesOptions {
  readonly dirty?: boolean;
  readonly limit?: number;
}
```

- **`dirty`**: Include uncommitted changes (may be rolled back)
- **`limit`**: Maximum number of changes to return

### Incremental Sync Pattern

The classic incremental sync pattern:

```typescript
class ChangeTracker {
  private lastClock: ClockHead = [];
  
  async sync() {
    // Get changes since last sync
    const changes = await db.changes(this.lastClock);
    
    // Process each change
    for (const row of changes.rows) {
      if (row.value._deleted) {
        await this.handleDeletion(row.key);
      } else {
        await this.handleUpdate(row.key, row.value);
      }
    }
    
    // Update clock position
    this.lastClock = changes.clock;
  }
  
  private async handleDeletion(id: string) {
    // Remove from local cache, UI, etc.
    console.log(`Deleted: ${id}`);
  }
  
  private async handleUpdate(id: string, doc: DocWithId<MyType>) {
    // Update local cache, UI, etc.
    console.log(`Updated: ${id}`, doc);
  }
}
```

### Subscribing to Real-Time Changes

Combine `changes()` with subscriptions for real-time updates:

```typescript
// Initial load
const initialChanges = await db.changes();

// Subscribe to future changes
const unsubscribe = db.subscribe(async (updates) => {
  // Updates is an array of changed documents
  for (const doc of updates) {
    if (doc._deleted) {
      handleDeletion(doc._id);
    } else {
      handleUpdate(doc);
    }
  }
});

// Later: cleanup
unsubscribe();
```

## Advanced Indexing Patterns

Fireproof's indexing system is similar to CouchDB map-reduce and HCL Notes views. You can build powerful lookup indexes using map functions.

### Index Basics

**Location**: `core/base/indexer.ts`

Indexes are defined with a **map function** that emits key-value pairs:

```typescript
// Simple field index
const index = db.query('title');  // Indexes by 'title' field

// Custom map function
const index = db.query((doc, emit) => {
  emit(doc.category, doc);  // Index by category
});
```

### Map Function Signature

```356:356:core/types/base/types.ts
export type MapFn<T extends DocTypes> = (doc: DocWithId<T>, emit: EmitFn) => DocFragment | unknown;
```

**Parameters**:
- **`doc`**: The document being indexed
- **`emit(key, value?)`**: Function to emit index entries

**Return**:
- If function returns a value (and doesn't call `emit`), that value becomes the key
- If function calls `emit`, can emit multiple entries per document

### Pattern 1: Simple Field Index

Index a single field (similar to CouchDB):

```typescript
// Index by title field
const result = await db.query('title');

// Query by key
const exact = await db.query('title', { key: 'Hello World' });

// Range query
const range = await db.query('title', { 
  range: ['A', 'M']  // All titles starting A-M
});

// Prefix query
const prefix = await db.query('title', { 
  prefix: 'Hello'  // All titles starting with "Hello"
});
```

### Pattern 2: Compound Keys (Multi-Field Index)

Index by multiple fields (like SQL composite index):

```typescript
interface Task {
  status: string;
  priority: number;
  title: string;
}

// Compound index: [status, priority]
const result = await db.query<Task, [string, number]>(
  (doc, emit) => {
    emit([doc.status, doc.priority]);
  }
);

// Query by compound key
const highPriorityActive = await db.query('status-priority', {
  key: ['active', 10]  // status='active', priority=10
});

// Range query on compound key
const activeHighPriority = await db.query('status-priority', {
  range: [['active', 5], ['active', Infinity]]  // active with priority >= 5
});
```

### Pattern 3: Emit Multiple Entries (Array Indexing)

Emit multiple entries per document (useful for tags, categories, etc.):

```typescript
interface Article {
  title: string;
  tags: string[];
  author: string;
}

// Emit one entry per tag
const result = await db.query<Article, string>(
  (doc, emit) => {
    doc.tags.forEach(tag => {
      emit(tag, { title: doc.title, author: doc.author });  // Store partial doc as value
    });
  }
);

// Query all articles with tag 'javascript'
const jsArticles = await db.query('tags', { key: 'javascript' });
```

### Pattern 4: Normalized Lookups (Reverse Index)

Create reverse lookups (many-to-one relationships):

```typescript
interface User {
  _id: string;
  email: string;
  name: string;
}

interface Order {
  _id: string;
  userId: string;
  total: number;
  items: string[];
}

// Create email → user lookup
const emailIndex = db.query<User, string>(
  (user, emit) => {
    emit(user.email, user._id);  // Email → User ID
  }
);

// Create user → orders lookup
const userOrdersIndex = db.query<Order, string>(
  (order, emit) => {
    emit(order.userId, order._id);  // User ID → Order IDs
  }
);

// Usage: Find user by email, then their orders
const userByEmail = await emailIndex.query({ key: 'user@example.com' });
if (userByEmail.rows.length > 0) {
  const userId = userByEmail.rows[0].value;  // User ID
  const orders = await userOrdersIndex.query({ key: userId });
}
```

### Pattern 5: Date/Time Range Queries

Index and query by date ranges:

```typescript
interface Event {
  title: string;
  startDate: string;  // ISO 8601 string
  endDate: string;
  category: string;
}

// Index by start date
const dateIndex = db.query<Event, string>(
  (event, emit) => {
    emit(event.startDate);  // ISO string sorts correctly
  }
);

// Query events in date range
const upcomingEvents = await dateIndex.query({
  range: [
    new Date().toISOString(),        // From now
    new Date(2025, 11, 31).toISOString()  // To end of 2025
  ],
  limit: 100
});
```

### Pattern 6: Full-Text Search Index (Keywords)

Create keyword-based full-text search:

```typescript
interface Document {
  title: string;
  content: string;
  author: string;
}

// Tokenize and index keywords
const searchIndex = db.query<Document, string>(
  (doc, emit) => {
    const text = `${doc.title} ${doc.content}`.toLowerCase();
    const words = text.split(/\s+/);
    const uniqueWords = [...new Set(words)];
    
    uniqueWords.forEach(word => {
      if (word.length > 2) {  // Ignore short words
        emit(word, { title: doc.title, author: doc.author });
      }
    });
  }
);

// Search for documents containing word
const results = await searchIndex.query({ key: 'javascript' });
```

### Pattern 7: Aggregation Indexes (Count, Sum, etc.)

Pre-compute aggregations:

```typescript
interface Sale {
  product: string;
  quantity: number;
  price: number;
  date: string;
}

// Index by product with aggregated values
const productStats = db.query<Sale, string, { totalQuantity: number, totalRevenue: number }>(
  (sale, emit) => {
    emit(sale.product, {
      totalQuantity: sale.quantity,
      totalRevenue: sale.quantity * sale.price
    });
  }
);

// Note: Fireproof doesn't auto-aggregate, but you can:
// 1. Store aggregated values in documents
// 2. Use changes() to recalculate on updates
// 3. Build secondary aggregation layer
```

### Pattern 8: Hierarchical/Tree Indexes

Index tree structures (folders, categories, etc.):

```typescript
interface File {
  path: string;  // e.g., "/home/docs/file.txt"
  name: string;
  size: number;
}

// Index by path components
const pathIndex = db.query<File, string>(
  (file, emit) => {
    const parts = file.path.split('/').filter(p => p);
    parts.forEach((part, i) => {
      emit(parts.slice(0, i + 1).join('/'), file);  // Emit each path level
    });
  }
);

// Query files in directory
const docsFiles = await pathIndex.query({ prefix: '/home/docs' });
```

### Pattern 9: Conditional Indexes (Filtered)

Index only documents matching certain conditions:

```typescript
interface Task {
  status: 'todo' | 'done' | 'archived';
  priority: number;
  assignee: string;
}

// Only index active (non-archived) tasks by assignee
const assigneeIndex = db.query<Task, string>(
  (task, emit) => {
    if (task.status !== 'archived') {
      emit(task.assignee, task);
    }
  }
);

// Query tasks assigned to user
const myTasks = await assigneeIndex.query({ key: 'alice' });
```

### Pattern 10: Multi-Dimensional Indexes

Combine multiple dimensions for complex queries:

```typescript
interface Product {
  category: string;
  price: number;
  rating: number;
  inStock: boolean;
}

// Multi-dimensional: [category, priceRange, rating]
const productIndex = db.query<Product, [string, string, number]>(
  (product, emit) => {
    const priceRange = product.price < 10 ? 'low' : 
                     product.price < 50 ? 'mid' : 'high';
    emit([product.category, priceRange, product.rating], product);
  }
);

// Query: electronics, mid-price, high rating
const results = await productIndex.query({
  range: [
    ['electronics', 'mid', 4.0],
    ['electronics', 'mid', Infinity]
  ]
});
```

## Query Options

**Location**: `core/types/base/types.ts`

```325:333:core/types/base/types.ts
export interface QueryOpts<K extends IndexKeyType> {
  readonly descending?: boolean;
  readonly limit?: number;
  includeDocs?: boolean;
  readonly range?: [IndexKeyType, IndexKeyType];
  readonly key?: DocFragment;
  readonly keys?: DocFragment[];
  prefix?: IndexKeyType;
}
```

**Options**:
- **`key`**: Exact match query
- **`keys`**: Multiple exact matches (OR query)
- **`range`**: Range query `[start, end]` (inclusive)
- **`prefix`**: Prefix match (strings, arrays)
- **`limit`**: Maximum results
- **`descending`**: Reverse sort order
- **`includeDocs`**: Include full documents in results (default: false)

## Comparison with HCL Notes/Domino

Fireproof indexes are similar to Notes views:

| Notes/Domino | Fireproof |
|--------------|-----------|
| View column formula | Map function return value |
| Multiple columns | Compound key array |
| Categorized views | Group by first key component |
| Selection formula | Conditional emit |
| View rebuild | Index updates on changes() |

**Notes View Example**:
```
Column 1: Category (sorted)
Column 2: Title (sorted)
Selection: Status = "Active"
```

**Fireproof Equivalent**:
```typescript
const view = db.query((doc, emit) => {
  if (doc.status === 'active') {
    emit([doc.category, doc.title]);
  }
});
```

## Comparison with CouchDB

Fireproof is very similar to CouchDB:

| CouchDB | Fireproof |
|---------|-----------|
| Map function | Map function (MapFn) |
| Reduce function | Not built-in (aggregate in app layer) |
| Design documents | Index created via `query()` |
| View queries | `query()` with options |
| All docs | `allDocs()` |

**Key Difference**: Fireproof doesn't have reduce functions built-in. Aggregate data by:
1. Storing aggregated values in documents
2. Processing `changes()` to update aggregates
3. Using secondary aggregation indexes

## Best Practices

### 1. Name Your Indexes

```typescript
// Named index (uses function string as name)
const index = db.query((doc, emit) => emit(doc.status));

// Better: Explicitly name it
const statusIndex = db.query('status');  // Field name
// or use a descriptive variable name that matches usage
```

### 2. Reuse Indexes

```typescript
// Create once, reuse
const categoryIndex = db.query('category');

// Use for multiple queries
const electronics = await categoryIndex.query({ key: 'electronics' });
const clothing = await categoryIndex.query({ key: 'clothing' });
```

### 3. Combine with Changes API

```typescript
// Maintain index incrementally
const index = db.query('status');
let lastClock: ClockHead = [];

async function updateIndex() {
  const changes = await db.changes(lastClock);
  
  // Process changes to update application-side aggregates
  for (const row of changes.rows) {
    if (!row.value._deleted) {
      // Update your aggregates based on changes
    }
  }
  
  lastClock = changes.clock;
}
```

### 4. Handle Deletions in Indexes

```typescript
// Index automatically handles deletions (removes entries)
// But you can filter in map function too
const activeIndex = db.query((doc, emit) => {
  if (!doc._deleted && doc.status === 'active') {
    emit(doc.category);
  }
});
```

### 5. Use Compound Keys for Sorting

```typescript
// Sort by date, then by priority
const sorted = db.query((doc, emit) => {
  emit([doc.date, doc.priority]);  // Automatically sorted
});

// Query in range (date between X and Y)
const results = await sorted.query({
  range: [
    ['2024-01-01', -Infinity],
    ['2024-12-31', Infinity]
  ]
});
```

## Performance Considerations

1. **Index Updates**: Indexes update automatically on document changes
2. **Incremental Updates**: Indexes use `changes()` internally for efficiency
3. **Storage**: Indexes are stored separately and synced like documents
4. **Query Speed**: Prolly trees provide O(log n) query performance
5. **Memory**: Indexes load on-demand (lazy loading)

## Time-Travel Capabilities

The `changes()` API enables **time-travel queries** - a powerful out-of-the-box feature in Fireproof. Since every change is tracked with a clock head, you can query the state of your database at any point in time.

**Key Insight**: The `since` parameter in `changes()` acts as a time machine - set it to any clock head to see what changed from that point forward, or reconstruct the complete state at that historical moment.

**Example**: 
```typescript
// Get state at specific time
const historicalState = await db.changes(targetClockHead);

// Reconstruct complete snapshot
const snapshot = await reconstructSnapshotAt(db, targetClockHead);
```

**Full Documentation**: See **[Time-Travel Queries](./TIME_TRAVEL.md)** for comprehensive guide on:
- Time-travel vector search
- Historical state reconstruction
- Temporal analysis and debugging
- Version-aware queries

## Related Documentation

- **[Time-Travel Queries](./TIME_TRAVEL.md)**: Complete guide to time-travel capabilities
- **[CRDT and Conflict Resolution](./CRDT.md)**: How changes are merged
- **[Metadata and Synchronization](./METADATA_SYNC.md)**: How changes are synced
- **[Write-Ahead Log (WAL)](./WAL.md)**: How changes are queued for sync

