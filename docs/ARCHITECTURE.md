# Fireproof Architecture

This document provides an in-depth overview of the Fireproof architecture, including its core components, data flow, and design decisions.

## High-Level Architecture

Fireproof is built as a layered architecture with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────┐
│              User Applications                          │
│  (React Apps, Node.js Apps, Deno Apps, etc.)          │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│              use-fireproof / @fireproof/core              │
│  (React Hooks, Database API, Query Interface)           │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│                  Core Base Layer                         │
│  (Ledger, CRDT, Database, Indexer, Write Queue)         │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│              Blockstore Layer                            │
│  (Encrypted Storage, Transactions, CAR Files)          │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│              Gateway Layer                                │
│  (Storage Gateways: File, IndexedDB, Cloud, Memory)    │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│              Storage Backends                            │
│  (File System, IndexedDB, S3, Cloudflare, etc.)         │
└─────────────────────────────────────────────────────────┘
```

## Core Components

### 1. Ledger (`core/base`)

The Ledger is the central component that manages document storage and versioning. It provides:

- **Document Operations**: `put`, `get`, `del`
- **Query Interface**: Indexed queries with range support
- **Change Subscriptions**: Real-time updates
- **CRDT Integration**: Conflict-free replication

Key Files:
- `ledger.ts` - Main ledger implementation
- `database.ts` - Database wrapper around ledger
- `crdt.ts` - CRDT implementation for multi-writer scenarios

### 2. Blockstore (`core/blockstore`)

The Blockstore provides encrypted, content-addressed storage:

- **CAR Files**: Content-Addressed Archives (IPFS format)
- **Encryption**: End-to-end encryption of stored data
- **Transactions**: Atomic operations with rollback
- **Replication**: Sync across devices and networks

Key Files:
- `store.ts` - Main blockstore implementation
- `transaction.ts` - Transaction management
- `loader.ts` - Block loading and caching
- `commitor.ts` - CAR file commitment

### 3. Gateways (`core/gateways`)

Gateways abstract storage backends:

- **File Gateway**: Local file system storage
- **IndexedDB Gateway**: Browser persistent storage
- **Memory Gateway**: In-memory storage (testing)
- **Cloud Gateway**: Remote storage sync

Each gateway implements the `Gateway` interface and handles serialization/deserialization.

### 4. Runtime (`core/runtime`)

Runtime utilities and platform abstractions:

- **Crypto**: Encryption/decryption operations
- **File Encoding**: File attachment handling
- **System Container**: Platform detection and utilities
- **Task Manager**: Async task coordination

### 5. Key Management (`core/keybag`)

Handles encryption key storage and retrieval:

- **Key Generation**: Automatic key creation
- **Key Storage**: Secure key persistence
- **Key Sharing**: Cross-device key export/import

## Data Flow

### Write Flow

```
1. User calls db.put(doc)
   ↓
2. WriteQueue batches the operation
   ↓
3. Ledger processes the update
   ↓
4. CRDT merges with existing state
   ↓
5. Blockstore transaction starts
   ↓
6. Data encoded as IPLD/CBOR blocks
   ↓
7. Blocks encrypted with symmetric key
   ↓
8. Gateway serializes and stores
   ↓
9. Transaction commits as CAR file
   ↓
10. Subscribers notified
```

### Read Flow

```
1. User calls db.get(id) or db.query(index)
   ↓
2. Ledger checks local cache
   ↓
3. If not found, queries Blockstore
   ↓
4. Blockstore loads from Gateway
   ↓
5. Data decrypted
   ↓
6. Blocks decoded from IPLD
   ↓
7. CRDT state reconstructed
   ↓
8. Result returned to user
```

### Sync Flow

```
1. Remote change detected via Gateway
   ↓
2. New CAR file downloaded
   ↓
3. Blocks extracted and decrypted
   ↓
4. CRDT merges with local state
   ↓
5. Ledger updates indices
   ↓
6. Subscribers notified of changes
```

## Data Structures

### Document Format

```typescript
interface Document {
  _id: string;              // Document ID
  [key: string]: any;        // User data
  _files?: {                 // File attachments (private)
    [name: string]: FileRef;
  };
  _publicFiles?: {           // Public file attachments
    [name: string]: FileRef;
  };
}
```

**Note**: File attachments are stored and synchronized separately from document data. See [File Attachments Documentation](./ATTACHMENTS.md) for details.

### Storage Format

- **CAR Files**: Content-Addressed Archives containing IPLD blocks
- **Metadata**: JSON metadata stored separately
- **File Store**: Separate storage for file attachments (see [ATTACHMENTS.md](./ATTACHMENTS.md))
- **Encryption**: Symmetric encryption (AES) with per-blockstore keys
- **Indexing**: Prolly trees for efficient range queries

**Important**: Fireproof uses a dual store system:
- **CAR Store** (`store: "car"`) - Document data, CRDT operations, DAG structures
- **File Store** (`store: "file"`) - File attachments, raw file data

Files are stored separately and synchronized via a different mechanism than documents.

### CRDT Implementation

Fireproof uses a Merkle clock-based CRDT:

- **Vector Clocks**: Causal ordering of operations
- **Merkle Trees**: Hash-based verification (from @web3-storage/pail)
- **Automatic Merging**: Conflict-free merge operations

## Encryption Model

- **Symmetric Keys**: Each blockstore has its own encryption key
- **Content Addressing**: Blocks identified by CID (hash)
- **Key Sharing**: Keys can be exported/imported for multi-device access
- **Gateway Security**: Keys transmitted via metadata channel (configurable)

## Query System

### Indexes

Fireproof supports secondary indexes on any field:

```typescript
const result = await db.query("fieldName", {
  range: ["start", "end"],
  limit: 100,
  descending: true
});
```

### Index Implementation

- **Prolly Trees**: Probabilistic B-trees for efficient range queries
- **Charwise Encoding**: Lexicographic ordering for strings
- **Automatic Updates**: Indexes updated on document changes

## Protocols

### Cloud Protocol (`core/protocols/cloud`)

HTTP/WebSocket protocol for cloud sync:

- **CAR Upload/Download**: Binary data transfer
- **Metadata Operations**: JSON metadata sync
- **Authentication**: Token-based auth with Clerk integration
- **Real-time**: WebSocket support for live updates

### Dashboard Protocol (`core/protocols/dashboard`)

Protocol for the web dashboard application.

## Performance Optimizations

### Write Queue

- **Batching**: Multiple writes batched into single transaction
- **Async Processing**: Non-blocking write operations
- **Queue Management**: Configurable queue options

### Caching

- **Block Cache**: Recently accessed blocks cached in memory
- **Index Cache**: Query results cached when possible
- **Lazy Loading**: Blocks loaded on-demand

### Compaction

- **Deduplication**: Remove duplicate blocks
- **CAR Optimization**: Merge small CAR files
- **Garbage Collection**: Remove unreferenced blocks

## Platform Support

### Browser

- Uses IndexedDB for persistence
- Web Crypto API for encryption
- WebSocket for real-time sync

### Node.js

- File system gateway for storage
- Node crypto module
- HTTP/WebSocket clients

### Deno

- Native Deno file APIs
- Deno crypto APIs
- Built-in fetch/WebSocket

### Cloudflare Workers

- D1 database for metadata
- R2 for CAR file storage
- Workers runtime APIs

## Testing Strategy

- **Unit Tests**: Individual component testing
- **Integration Tests**: Cross-component testing
- **Smoke Tests**: End-to-end environment testing
- **React Tests**: Component testing with Vitest

## Security Considerations

- **Encryption**: All data encrypted at rest
- **Key Management**: Keys never transmitted unencrypted
- **Content Addressing**: Tamper-proof via hash verification
- **Access Control**: Token-based authentication for cloud sync

