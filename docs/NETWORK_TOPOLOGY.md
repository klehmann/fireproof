# Network Topology and Server Architecture

This document explains Fireproof's network topology, how clients and servers interact, and the implications for encryption and data synchronization.

## Standard Topology: Client-Server (Hub-and-Spoke)

Fireproof uses a **client-server (hub-and-spoke) architecture**:

```
        ┌─────────┐
        │ Server 1│
        └────┬────┘
             │
      ┌──────┼──────┐
      │      │      │
   ┌──▼─┐ ┌─▼──┐ ┌─▼──┐
   │C1  │ │ C2 │ │ C3 │  (Clients)
   └────┘ └────┘ └────┘
```

**Characteristics**:
- **Many clients, one or more servers**: Multiple clients connect to the same server
- **Clients are peers**: All clients have equal rights to read/write
- **Server as dumb storage**: Server stores encrypted data without decryption keys
- **No client-to-client**: Clients don't directly sync with each other (they sync through server)

### Client Role

- **Encrypts/Decrypts**: All encryption/decryption happens on clients
- **Manages keys**: Encryption keys stored in client keybag (never sent to server)
- **Writes locally first**: Fast local writes, async sync to server
- **Reads from server**: Downloads encrypted blocks from server when needed

### Server Role

- **Storage only**: Stores encrypted CAR files and metadata
- **No decryption**: Server never has encryption keys
- **Content-addressed**: Identifies blocks by CID (hash)
- **Protocol handler**: Responds to Fireproof protocol messages

## Multiple Servers (Multiple Hubs)

Fireproof supports attaching to **multiple servers**:

```typescript
const db = fireproof("my-app", {
  attach: [
    toCloud({ urls: { base: "https://server1.example.com" } }),
    toCloud({ urls: { base: "https://server2.example.com" } }),
  ]
});
```

**Topology**:
```
     ┌─────────┐     ┌─────────┐
     │ Server 1│     │ Server 2│
     └────┬────┘     └────┬────┘
          │               │
          │           ┌───┼───┐
          │           │   │   │
       ┌──▼─┐      ┌─▼──┐ ┌─▼──┐
       │ C1 │      │ C2 │ │ C3 │
       └────┘      └────┘ └────┘
```

**How It Works**:
- Client uploads to **all attached servers** (via `forRemotes()`)
- Client downloads from **any attached server** (loads missing blocks)
- **Servers don't sync with each other** - they're independent
- All servers have same encrypted data (replicated by client uploads)

## Server-to-Server Sync: Current State

### ❌ Not Currently Supported

Fireproof **does not currently support** direct server-to-server synchronization.

**What's Missing**:
- No gateway that acts as both client and server
- No server-side Fireproof instance that can attach to other servers
- No automatic replication between servers without client involvement

**Current Behavior**:
- Servers are **independent storage endpoints**
- Data replication happens only when clients upload to multiple servers
- If Server A has data and Server B doesn't, they won't sync without a client

### ✅ Theoretical Possibility

Server-to-server sync **is architecturally possible** because:

1. **Servers store encrypted blocks**: All data is content-addressed (CID-based)
2. **No keys needed for storage**: Servers can store encrypted blocks without decryption
3. **Content-addressed storage**: Blocks identified by hash, can be replicated blindly

**Why It Could Work**:
- Encrypted blocks can be copied between servers without decryption
- CIDs allow verification without decryption
- Metadata (CRDT clock head) is also encrypted and can be replicated
- Server acts as "client" to sync from another server

### Potential Implementation

A server-to-server sync could work like this:

```typescript
// Hypothetical server-to-server sync
class ServerToServerSync {
  async sync(fromServer: string, toServer: string) {
    // 1. Create database and attach to BOTH servers
    // Note: This would require the server to have encryption keys
    // to decrypt metadata and determine which blocks to sync
    const db = fireproof("sync-db", {
      // Attach to source server (download)
      attach: toCloud({ urls: { base: fromServer } })
    });
    
    // 2. Also attach to destination server (upload)
    await db.attach(toCloud({ urls: { base: toServer } }));
    
    // 3. Download metadata from source server
    const sourceMeta = await db.ledger.crdt.blockstore.loader.attachedStores
      .remotes()
      .find(store => store.active.meta.url().toString().includes(fromServer))
      ?.active.meta.load();
    
    if (!sourceMeta) {
      throw new Error("Failed to load metadata from source server");
    }
    
    // 4. Download all CAR files referenced in metadata
    for (const carCid of sourceMeta.cars) {
      const car = await db.ledger.crdt.blockstore.loader.attachedStores
        .remotes()
        .find(store => store.active.car.url().toString().includes(fromServer))
        ?.active.car.load(carCid);
      
      if (car) {
        // 5. Upload to destination server only
        // Find the specific remote store for toServer and upload directly
        const destStore = db.ledger.crdt.blockstore.loader.attachedStores
          .remotes()
          .find(store => store.active.car.url().toString().includes(toServer));
        
        if (destStore) {
          await destStore.active.car.save(car);
        }
      }
    }
    
    // 6. Upload metadata to destination server
    await db.ledger.crdt.blockstore.loader.attachedStores
      .remotes()
      .find(store => store.active.meta.url().toString().includes(toServer))
      ?.active.meta.save(sourceMeta);
  }
}
```

**Note**: This example shows the concept, but has limitations:
- `forRemotes()` uploads to **all** remotes, not a specific one
- To target a specific server, you'd need to filter `remotes()` array or use a different approach
- Server would need encryption keys to decrypt metadata and understand which blocks to sync
- Without keys, server can only do blind replication (copy all encrypted blocks)

**Requirements**:
- Server would need encryption keys (to decrypt metadata, find blocks)
- OR: Server syncs **only encrypted blocks** without understanding content
- Server would need Fireproof client library running on server

## How Servers Handle Encrypted Data

### Server Perspective: Encrypted Opaque Data

**What Servers See**:
- **Encrypted CAR files**: Binary data, cannot be decrypted
- **Encrypted metadata**: JSON with encrypted fields
- **CIDs**: Content identifiers (hashes) used to reference blocks
- **Store types**: `car`, `file`, `meta`, `wal` (but all encrypted)

**What Servers Don't See**:
- ❌ Decrypted document content
- ❌ Encryption keys
- ❌ Document structure or field names (encrypted)
- ❌ File content (encrypted)

### Server Storage Structure

**Location**: `fireproof-test/src/server/fireproof-backend.ts`

```
data/
├── {tenant}/
│   ├── {ledger}/
│   │   ├── car-files/          # Encrypted CAR files
│   │   ├── file-store/         # Encrypted file attachments
│   │   └── meta-store.json     # Encrypted metadata
```

**Server Operations**:
1. **Receive encrypted block**: Client uploads encrypted CAR/file via `reqPutData`
2. **Store by CID**: Save encrypted data identified by CID
3. **Serve encrypted block**: Return encrypted data when client requests by CID
4. **No decryption**: Server never decrypts, just stores and serves

### Protocol: Encrypted Data Transfer

**Location**: `core/protocols/cloud/index.ts`

The Fireproof protocol transfers encrypted data:

**Messages**:
- `reqPutData`: Upload encrypted block (CAR, file, meta, wal)
- `reqGetData`: Download encrypted block by CID
- `reqGetMeta`: Download metadata (also encrypted)
- `reqPutMeta`: Upload metadata (encrypted)

**Server Implementation**:
```typescript
// Server receives encrypted data
if (msg.type === 'reqPutData') {
  const { cid, store } = msg.methodParam;
  // Store encrypted block (no decryption)
  storage.save(store, cid, msg.payload); // payload is encrypted
}
```

## Encryption Keys and Servers

### Key Management: Client-Side Only

**Key Storage**:
- **Clients**: Keys stored in keybag (IndexedDB, file system, or memory)
- **Servers**: **Never store encryption keys**

**Key Transfer**:
- Keys **never sent to server** in normal operation
- Keys can be **exported/imported between clients** (manual key sharing)
- Server stores encrypted data that clients can decrypt if they have keys

### Why Servers Don't Need Keys

**Content-Addressed Storage**:
- Blocks identified by CID (hash of encrypted content)
- Server can store/retrieve blocks without understanding content
- CID is same whether data is encrypted or not

**Encrypted Metadata**:
- Metadata is also encrypted before upload
- Server stores encrypted metadata
- Client decrypts metadata on download

**Example Flow**:
```
Client:
  1. Encrypt document → encrypted block (CID: abc123)
  2. Upload to server: PUT /fp?car=abc123 { encrypted bytes }

Server:
  1. Receives encrypted bytes
  2. Stores: data/{tenant}/{ledger}/car-files/abc123.car
  3. Returns: { status: 'ok' }

Another Client:
  1. Requests: GET /fp?car=abc123
  2. Server returns: encrypted bytes (doesn't decrypt)
  3. Client decrypts: document content
```

## Multi-Server Scenarios

### Scenario 1: Redundant Storage

**Use Case**: Backup/redundancy across multiple servers

```
Client → Uploads to Server A, Server B, Server C
         (Same encrypted data to all)
```

**Benefits**:
- Redundancy: If one server fails, data on others
- Geographic distribution: Servers in different regions
- Load balancing: Clients can download from nearest server

**Limitation**:
- No automatic replication if one server gets new data
- Client must upload to all servers

### Scenario 2: Primary and Backup

**Use Case**: Primary server for writes, backup for disaster recovery

```
Client → Writes to Primary Server
         → Periodically syncs to Backup Server
```

**How to Implement** (requires custom code):
- Client attached to both servers
- WAL processes uploads to primary
- Background job syncs primary → backup (as client)

### Scenario 3: Federation

**Use Case**: Multiple independent servers, clients choose which to use

```
Client A → Server 1 (region A)
Client B → Server 2 (region B)
Client C → Both servers (multi-region)
```

**Current Support**: ✅ Clients can attach to multiple servers

**Not Supported**: ❌ Servers syncing with each other automatically

## Client-Side Multi-Server Sync

Clients can sync to multiple servers simultaneously:

**Location**: `core/blockstore/store.ts:WAL processing`

```572:590:core/blockstore/store.ts
// Process fileOperations
await pMap(
  fileOperations,
  async ({ cid: fileCid }) => {
    await retryableUpload(async () => {
      const fileBlock = await this.loader.attachedStores.local().active.file.load(fileCid);
      if (!fileBlock) {
        throw this.logger.Error().Ref("cid", fileCid).Msg("missing file block").AsError();
      }
      // Upload to ALL remote stores
      await this.loader.attachedStores.forRemotes((x) => x.active.file.save(fileBlock, { public: false }));
    }, `fileOperation with cid=${fileCid.toString()}`);
  },
  { concurrency: concurrencyLimit },
);
```

**How It Works**:
- `forRemotes()` iterates through all attached remote stores
- Uploads same encrypted block to each server
- Each server stores independently
- All servers end up with same encrypted data

## Protocol Implementation

### HTTP/HTTPS Gateway

**Client → Server**: HTTP requests with encrypted payloads

```typescript
// Client uploads encrypted block
PUT /fp?car={cid}&tenant={tenant}&ledger={ledger}
Content-Type: application/octet-stream
Body: { encrypted CAR bytes }
```

### WebSocket Gateway

**Client → Server**: Real-time WebSocket connection

**Location**: `fireproof-test/src/server/fireproof-backend.ts`

```916:996:fireproof-test/src/server/fireproof-backend.ts
// WebSocket endpoint for real-time communication
this.app.get('/ws', async (c: Context) => {
  // Handle WebSocket upgrade
  // Messages: reqPutData, reqGetData, etc.
  // All payloads are encrypted
});
```

**Benefits**:
- Real-time updates
- Bidirectional communication
- Lower latency than HTTP polling

## Security Model

### End-to-End Encryption

**Encryption Chain**:
```
Client (encrypts) → Network → Server (stores encrypted) → Network → Client (decrypts)
```

**What's Encrypted**:
- ✅ Document content (CAR files)
- ✅ File attachments
- ✅ Metadata (CRDT clock head, etc.)
- ✅ WAL state

**What's Not Encrypted** (but could be):
- HTTP headers (can use HTTPS)
- Query parameters (CIDs, store types)
- Request/response structure

### Server Security

**Server Capabilities**:
- ✅ Can serve encrypted data
- ✅ Can store encrypted data
- ✅ Can verify CIDs (content integrity)
- ❌ Cannot read document content
- ❌ Cannot modify document content (would break CID)
- ❌ Cannot decrypt data

**Attack Surface**:
- Server compromise: Attacker gets encrypted blocks (cannot decrypt)
- Network interception: HTTPS protects transmission
- CID spoofing: Prevented by content addressing (hash verification)

## Future: Server-to-Server Sync

### Possible Implementation Approaches

**Approach 1: Server as Client**
- Server runs Fireproof client library
- Attaches to other servers as remote stores
- Syncs encrypted blocks without decryption

**Approach 2: Block Replication**
- Direct block replication between servers
- Server A → Server B: Copy encrypted blocks by CID
- No Fireproof client needed, just block storage

**Approach 3: Metadata-Driven Sync**
- Servers exchange metadata (encrypted CRDT heads)
- Determine which blocks need replication
- Replicate missing encrypted blocks

### Requirements for Server-to-Server Sync

**Without Keys** (theoretical):
- ✅ Sync encrypted blocks by CID
- ✅ Sync encrypted metadata
- ❌ Cannot verify content (only hash)
- ❌ Cannot determine if block is latest version
- ❌ Cannot merge CRDTs (requires decryption)

**With Keys** (practical):
- ✅ Full CRDT merge on server
- ✅ Content verification
- ✅ Latest version detection
- ❌ Server can decrypt data (security tradeoff)
- ❌ Requires key management on server

## Summary

### Current Topology

**Standard**: Many clients → One server (hub-and-spoke)

**Multi-Server**: Many clients → Multiple servers (clients upload to all)

**Server Role**: Dumb storage (encrypted blocks, no keys, no decryption)

### Server-to-Server Sync

**Status**: ❌ Not currently supported

**Possibility**: ✅ Architecturally possible (servers can sync encrypted blocks)

**Why Not Implemented**: 
- Current architecture focuses on client-server
- Server-to-server would require server-side Fireproof client
- Security tradeoffs (keys on server vs. blind replication)

### Encryption and Servers

**Key Principle**: Servers **never have encryption keys**

**Implications**:
- Servers store encrypted data as opaque blobs
- Content-addressed storage (CID-based) enables blind replication
- Server-to-server sync possible without keys (but limited functionality)

### Recommendations

**For Redundancy**: Use multi-server attachment from clients

**For Server-to-Server**: Currently not supported - clients must upload to all servers

**For Federation**: Multiple independent servers, clients choose which to use

## Related Files

- `core/blockstore/store.ts` - Remote store synchronization
- `core/protocols/cloud/index.ts` - Cloud protocol implementation
- `core/gateways/cloud/gateway.ts` - Cloud gateway for server communication
- `fireproof-test/src/server/fireproof-backend.ts` - Example server implementation

