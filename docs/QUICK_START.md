# Quick Start Guide

This guide helps you quickly understand and get started with the Fireproof codebase.

## What is Fireproof?

Fireproof is a lightweight embedded document database with:
- **Local-first** architecture (works offline)
- **Real-time sync** across devices
- **Encrypted** end-to-end
- **CRDT-based** for conflict-free replication
- **IPFS-compatible** content-addressed storage

## Key Concepts

### Ledger
The core data structure that stores documents and maintains version history using Merkle trees.

### Blockstore
Encrypted, content-addressed storage that persists data as CAR (Content-Addressed Archive) files.

### Gateways
Storage backends that abstract different storage mechanisms (file system, IndexedDB, cloud, etc.).

### CRDT
Conflict-free Replicated Data Types that enable multi-writer scenarios without conflicts.

## Codebase Structure

```
fireproof/
├── core/              # Core database implementation
│   ├── base/         # Ledger, CRDT, database
│   ├── blockstore/   # Encrypted storage
│   ├── gateways/     # Storage backends
│   └── runtime/      # Utilities and crypto
├── use-fireproof/    # Main user package
├── dashboard/        # Web dashboard app
└── cloud/            # Backend services
```

## Common Tasks

### Adding a New Gateway

1. Create new package in `core/gateways/your-gateway/`
2. Implement `Gateway` interface from `@fireproof/core-gateways-base`
3. Register protocol in `core/blockstore/register-store-protocol.ts`
4. Add to workspace in `pnpm-workspace.yaml`

### Adding a New Feature

1. Identify appropriate package (core/base, blockstore, etc.)
2. Add feature following existing patterns
3. Update type definitions in `core/types/`
4. Add tests in `core/tests/`
5. Update documentation

### Debugging

Set debug level:
```bash
FP_DEBUG='*' node your-script.js
```

Or in code:
```typescript
import { logger } from "@fireproof/core-runtime";
logger.setDebug("*");
```

### Testing

```bash
# Run all tests
pnpm run test

# Run specific test
pnpm run test -t "test name"

# Run with debug
FP_DEBUG='Loader,CRDTClock' pnpm run test -t 'codec implicit iv'
```

## Key Files to Understand

### Core Database

- `core/base/ledger.ts` - Main ledger implementation
- `core/base/database.ts` - Database API wrapper
- `core/base/crdt.ts` - CRDT implementation
- `core/base/indexer.ts` - Query indexing

### Storage

- `core/blockstore/store.ts` - Blockstore implementation
- `core/blockstore/transaction.ts` - Transaction handling
- `core/blockstore/loader.ts` - Block loading

### Gateways

- `core/gateways/base/index.ts` - Gateway interface
- `core/gateways/indexeddb/index.ts` - IndexedDB implementation
- `core/gateways/cloud/index.ts` - Cloud sync

### User API

- `use-fireproof/index.ts` - Main export
- `use-fireproof/react/use-fireproof.ts` - React hook
- `use-fireproof/react/use-live-query.ts` - Query hook

## Development Workflow

1. **Make changes** in appropriate package
2. **Build** with `pnpm run build` in package directory
3. **Test** with `pnpm run test`
4. **Check types** - TypeScript will catch errors
5. **Run smoke tests** - `pnpm run smoke` to test across environments

## Key Dependencies

- **@web3-storage/pail** - Merkle tree implementation
- **prolly-trees** - Index data structure
- **@ipld/car** - CAR file format
- **@ipld/dag-cbor** - IPLD encoding
- **multiformats** - CID handling

## Understanding Data Flow

1. **Write**: `db.put(doc)` → WriteQueue → CRDT → Blockstore → Gateway → Storage
2. **Read**: `db.get(id)` → Cache → Blockstore → Gateway → Storage → Decrypt
3. **Query**: `db.query(index)` → Indexer → Prolly Tree → Results
4. **Sync**: Gateway → Download CAR → Decrypt → Merge CRDT → Update Ledger

## Common Patterns

### Creating a Ledger

```typescript
import { LedgerFactory } from "@fireproof/core-base";

const ledger = LedgerFactory("my-app", {
  storeUrls: { data: "indexeddb://my-app" }
});
```

### Using a Gateway

```typescript
import { registerStoreProtocol } from "@fireproof/core-blockstore";

registerStoreProtocol({
  protocol: "custom",
  defaultURI: () => "custom://storage",
  serdegateway: async (sthis) => new CustomGateway(sthis)
});
```

### React Hook Usage

```typescript
import { useFireproof } from "use-fireproof";

function App() {
  const { database, useLiveQuery } = useFireproof("my-app");
  const { docs } = useLiveQuery("field", { limit: 100 });
  // ...
}
```

## Getting Help

- **Discord**: [Join the community](https://discord.gg/cCryrNHePH)
- **Documentation**: See other docs in this directory
- **Examples**: Check `examples/` and `cloud/todo-app/`
- **Tests**: See `core/tests/` for usage examples

## Next Steps

- Read [ARCHITECTURE.md](./ARCHITECTURE.md) for deep dive
- Check [PACKAGES.md](./PACKAGES.md) for package details
- Browse [packages/](./packages/) for specific package docs

