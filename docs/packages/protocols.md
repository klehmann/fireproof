# Protocol Packages

Protocol packages define communication protocols for syncing data between devices and services.

## @fireproof/core-protocols-cloud

**Location:** `core/protocols/cloud/`

**Purpose:** Cloud sync protocol implementation for HTTP/WebSocket-based synchronization.

**Features:**
- HTTP/HTTPS protocol for CAR file transfer
- WebSocket support for real-time updates
- Token-based authentication
- Metadata synchronization
- WAL (Write-Ahead Log) sync

**Dependencies:**
- `ws` - WebSocket library
- `@fireproof/core-runtime` - Runtime utilities

**Message Types:**
- **Data Messages**: CAR file transfers
- **Metadata Messages**: Index and state synchronization
- **WAL Messages**: Write-ahead log entries
- **Control Messages**: Gateway control operations

**Protocol Flow:**
1. Authentication via token
2. Establish WebSocket connection (optional)
3. Sync metadata to determine state
4. Upload/download CAR files
5. Real-time updates via WebSocket

**Use Cases:**
- Cloud backend services
- Multi-device sync
- Real-time collaboration
- Remote backup

## @fireproof/core-protocols-dashboard

**Location:** `core/protocols/dashboard/`

**Purpose:** Protocol for the Fireproof dashboard application.

**Features:**
- Dashboard-specific message types
- Dashboard state synchronization
- Admin operations

**Dependencies:**
- `@fireproof/core-protocols-cloud` - Uses cloud protocol as base
- `@fireproof/core-runtime` - Runtime utilities

**Use Cases:**
- Dashboard web application
- Admin interfaces
- Monitoring and management

