# Gateway Packages

Gateway packages provide storage backend implementations. Each gateway implements the `Gateway` interface and handles serialization/deserialization of data.

## @fireproof/core-gateways-base

**Location:** `core/gateways/base/`

**Purpose:** Base gateway interface and shared gateway functionality.

**Key Components:**
- Gateway interface definitions
- Base gateway implementation
- Shared serialization utilities

**Dependencies:**
- `@fireproof/core-runtime` - Runtime utilities
- `@ipld/dag-json` - JSON encoding for IPLD
- `@web3-storage/pail` - Merkle tree support

**Usage:**
Other gateways extend this package to implement specific storage backends.

## @fireproof/core-gateways-file

**Location:** `core/gateways/file/`

**Purpose:** Platform-agnostic file gateway that delegates to platform-specific implementations.

**Key Components:**
- File gateway abstraction
- Platform detection (Node.js vs Deno)

**Dependencies:**
- `@fireproof/core-gateways-file-node` - Node.js implementation
- `@fireproof/core-gateways-file-deno` - Deno implementation
- `@fireproof/core-gateways-base` - Base gateway

**Features:**
- Automatic platform detection
- Delegates to appropriate platform-specific gateway

## @fireproof/core-gateways-file-node

**Location:** `core/gateways/file-node/`

**Purpose:** Node.js-specific file system gateway implementation.

**Features:**
- Uses Node.js `fs` module
- File system persistence
- Directory creation
- File read/write operations

**Use Cases:**
- Server-side applications
- Node.js desktop apps
- Development and testing

## @fireproof/core-gateways-file-deno

**Location:** `core/gateways/file-deno/`

**Purpose:** Deno-specific file system gateway implementation.

**Features:**
- Uses Deno file APIs
- Native Deno file operations
- No Node.js dependencies

**Use Cases:**
- Deno applications
- Edge environments supporting Deno

## @fireproof/core-gateways-indexeddb

**Location:** `core/gateways/indexeddb/`

**Purpose:** Browser IndexedDB gateway for persistent browser storage.

**Features:**
- Browser-native persistent storage
- Large storage capacity
- Asynchronous operations
- Works offline

**Dependencies:**
- `idb` - IndexedDB wrapper library

**Use Cases:**
- Web applications
- Browser extensions
- Progressive Web Apps (PWAs)

**Storage Location:**
- Browser IndexedDB database
- Persistent across sessions
- Cleared when user clears browser data

## @fireproof/core-gateways-memory

**Location:** `core/gateways/memory/`

**Purpose:** In-memory gateway for testing and ephemeral storage.

**Features:**
- Fast in-memory storage
- No persistence
- Perfect for testing
- No I/O overhead

**Use Cases:**
- Unit testing
- Integration testing
- Temporary/ephemeral data
- Performance testing

**Limitations:**
- Data lost on process exit
- Not suitable for production persistence

## @fireproof/core-gateways-cloud

**Location:** `core/gateways/cloud/`

**Purpose:** Cloud gateway for remote storage synchronization.

**Features:**
- HTTP/HTTPS sync
- WebSocket support for real-time
- Authentication integration
- CAR file upload/download
- Metadata synchronization

**Dependencies:**
- `@fireproof/core-protocols-cloud` - Cloud protocol
- `jose` - JWT handling for authentication

**Protocol Support:**
- HTTP/HTTPS for CAR files
- WebSocket for real-time updates
- Token-based authentication

**Use Cases:**
- Multi-device synchronization
- Cloud backup
- Collaborative applications
- Remote access

