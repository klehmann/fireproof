# Core Packages

The core packages contain the fundamental database implementation, storage abstractions, and runtime utilities.

## @fireproof/core-base

**Location:** `core/base/`

**Purpose:** Contains the core ledger implementation, CRDT logic, database wrapper, and indexing functionality.

**Key Components:**
- **Ledger** (`ledger.ts`): Main ledger implementation managing document storage and versioning
- **Database** (`database.ts`): Database wrapper providing high-level API
- **CRDT** (`crdt.ts`): Conflict-free replicated data type implementation
- **Indexer** (`indexer.ts`): Index management and query processing
- **Write Queue** (`write-queue.ts`): Batching and queuing of write operations

**Dependencies:**
- `@fireproof/core-blockstore` - Storage abstraction
- `@fireproof/core-keybag` - Key management
- `@fireproof/core-runtime` - Runtime utilities
- `@web3-storage/pail` - Merkle tree implementation
- `prolly-trees` - Index data structure
- `@ipld/dag-cbor` - IPLD encoding

**Key Exports:**
- `LedgerFactory()` - Creates ledger instances
- `DatabaseImpl` - Database implementation
- `CRDTImpl` - CRDT implementation

## @fireproof/core-blockstore

**Location:** `core/blockstore/`

**Purpose:** Provides encrypted, content-addressed block storage using CAR (Content-Addressed Archive) files.

**Key Components:**
- **Store** (`store.ts`): Main blockstore implementation
- **Transaction** (`transaction.ts`): Atomic transaction handling
- **Loader** (`loader.ts`): Block loading and caching
- **Committer** (`commitor.ts`): CAR file commitment
- **Gateway Registration** (`register-store-protocol.ts`): Protocol registration

**Features:**
- Encrypted block storage
- CAR file management
- Transaction support with rollback
- Block caching
- Protocol-based storage backends

**Dependencies:**
- `@fireproof/core-gateways-*` - All gateway implementations
- `@ipld/car` - CAR file format
- `@ipld/dag-cbor` - Block encoding
- `@web3-storage/pail` - Merkle tree support

**Storage Format:**
- Blocks stored as CAR files
- Metadata stored separately
- Encryption at block level

## @fireproof/core

**Location:** `core/core/`

**Purpose:** Main entry point package that re-exports the core functionality.

**Key Exports:**
- Re-exports from `@fireproof/core-base`
- `fireproof()` factory function

**Usage:**
```typescript
import { fireproof } from "@fireproof/core";
const db = fireproof("my-app");
```

## @fireproof/core-runtime

**Location:** `core/runtime/`

**Purpose:** Runtime utilities, cryptographic operations, and platform abstractions.

**Key Components:**
- **Crypto** (`keyed-crypto.ts`): Encryption/decryption operations
- **File Encoding** (`files.ts`): File attachment handling
- **System Container** (`sys-container.ts`, `memory-sys-container.ts`): Platform detection
- **Task Manager** (`task-manager.ts`): Async task coordination
- **Commit Queue** (`commit-queue.ts`): Transaction queue management
- **Block Encoding** (`async-block-encode.ts`): Async block encoding utilities

**Dependencies:**
- `jose` - JWT and cryptographic operations
- `cborg` - CBOR encoding
- `multiformats` - CID handling

**Features:**
- Platform-agnostic crypto operations
- File encoding/decoding
- System detection (browser, Node.js, Deno)
- Async task management

## @fireproof/core-keybag

**Location:** `core/keybag/`

**Purpose:** Manages encryption keys for blockstore encryption.

**Key Components:**
- **KeyBag** (`key-bag.ts`): Key storage and retrieval
- **Memory KeyBag** (`key-bag-memory.ts`): In-memory key storage

**Features:**
- Key generation
- Key persistence via gateways
- Key export/import for multi-device access
- Secure key storage

**Dependencies:**
- `@fireproof/core-gateways-file` - File-based key storage
- `@fireproof/core-gateways-indexeddb` - Browser key storage
- `@fireproof/core-runtime` - Crypto utilities

## @fireproof/core-types-base

**Location:** `core/types/base/`

**Purpose:** TypeScript type definitions for core database functionality.

**Key Types:**
- `Database` - Database interface
- `Ledger` - Ledger interface
- `CRDT` - CRDT interface
- `DocUpdate` - Document update type
- `ConfigOpts` - Configuration options
- `ListenerFn` - Change listener type

## @fireproof/core-types-blockstore

**Location:** `core/types/blockstore/`

**Purpose:** TypeScript type definitions for blockstore functionality.

**Key Types:**
- `Store` - Blockstore interface
- `Gateway` - Gateway interface
- `SerdeGateway` - Serialization gateway interface
- `FPEnvelope` - Fireproof envelope format
- `TaskManager` - Task manager interface

## @fireproof/core-types-runtime

**Location:** `core/types/runtime/`

**Purpose:** TypeScript type definitions for runtime utilities.

**Key Types:**
- `CodecInterface` - Codec interface for encoding/decoding

## @fireproof/core-types-protocols-cloud

**Location:** `core/types/protocols/cloud/`

**Purpose:** TypeScript type definitions for cloud sync protocol.

**Key Types:**
- Message types for cloud protocol
- Gateway control types
- Data, metadata, and WAL message types

