# Complete Package List

This document provides a complete list of all packages in the Fireproof monorepo with brief descriptions.

## Core Packages

| Package | Location | Description |
|---------|----------|-------------|
| `@fireproof/core-base` | `core/base/` | Core ledger, CRDT, database, and indexer implementation |
| `@fireproof/core-blockstore` | `core/blockstore/` | Encrypted block storage with CAR file support |
| `@fireproof/core` | `core/core/` | Main entry point package |
| `@fireproof/core-runtime` | `core/runtime/` | Runtime utilities, crypto, and platform abstractions |
| `@fireproof/core-keybag` | `core/keybag/` | Encryption key management |
| `@fireproof/core-types-base` | `core/types/base/` | Core TypeScript type definitions |
| `@fireproof/core-types-blockstore` | `core/types/blockstore/` | Blockstore type definitions |
| `@fireproof/core-types-runtime` | `core/types/runtime/` | Runtime type definitions |
| `@fireproof/core-types-protocols-cloud` | `core/types/protocols/cloud/` | Cloud protocol type definitions |

## Gateway Packages

| Package | Location | Description |
|---------|----------|-------------|
| `@fireproof/core-gateways-base` | `core/gateways/base/` | Base gateway interface and utilities |
| `@fireproof/core-gateways-file` | `core/gateways/file/` | Platform-agnostic file gateway |
| `@fireproof/core-gateways-file-node` | `core/gateways/file-node/` | Node.js file system gateway |
| `@fireproof/core-gateways-file-deno` | `core/gateways/file-deno/` | Deno file system gateway |
| `@fireproof/core-gateways-indexeddb` | `core/gateways/indexeddb/` | Browser IndexedDB gateway |
| `@fireproof/core-gateways-memory` | `core/gateways/memory/` | In-memory gateway (for testing) |
| `@fireproof/core-gateways-cloud` | `core/gateways/cloud/` | Cloud sync gateway |

## Protocol Packages

| Package | Location | Description |
|---------|----------|-------------|
| `@fireproof/core-protocols-cloud` | `core/protocols/cloud/` | Cloud sync protocol (HTTP/WebSocket) |
| `@fireproof/core-protocols-dashboard` | `core/protocols/dashboard/` | Dashboard application protocol |

## User-Facing Packages

| Package | Location | Description |
|---------|----------|-------------|
| `use-fireproof` | `use-fireproof/` | **Main package** - React hooks and core API (published to npm) |
| `@fireproof/dashboard` | `dashboard/` | Web dashboard application (not published) |

## Cloud Packages

| Package | Location | Description |
|---------|----------|-------------|
| `@fireproof/cloud-base` | `cloud/base/` | Base cloud utilities and AWS integration |
| `@fireproof/cloud-backend-base` | `cloud/backend/base/` | Base backend server (Hono) |
| `@fireproof/cloud-backend-cf-d1` | `cloud/backend/cf-d1/` | Cloudflare Workers backend with D1 |
| `@fireproof/cloud-backend-node` | `cloud/backend/node/` | Node.js backend server |
| `@fireproof/cloud-todo-app` | `cloud/todo-app/` | Example todo application |
| `@fireproof/cloud-3rd-party` | `cloud/3rd-party/` | Third-party integration example |

## Infrastructure Packages

| Package | Location | Description |
|---------|----------|-------------|
| `@fireproof/core-cli` | `cli/` | Build and development CLI tool |
| `@fireproof/vendor` | `vendor/` | Vendor patches for ESM compatibility |

## Published Packages

The following packages are published to npm:

- `use-fireproof` - Main user-facing package
- `@fireproof/core` - Core database API
- All `@fireproof/core-*` packages (for advanced use cases)

## Package Dependencies

### Core Dependency Chain

```
use-fireproof
  └── @fireproof/core-base
      ├── @fireproof/core-blockstore
      │   ├── @fireproof/core-gateways-*
      │   ├── @fireproof/core-keybag
      │   └── @fireproof/core-runtime
      └── @fireproof/core-types-*
```

### Cloud Backend Chain

```
@fireproof/cloud-backend-*
  └── @fireproof/cloud-backend-base
      ├── @fireproof/cloud-base
      ├── @fireproof/core-protocols-cloud
      └── @fireproof/core-gateways-cloud
```

## Package Statistics

- **Total Packages**: 26
- **Core Packages**: 9
- **Gateway Packages**: 7
- **Protocol Packages**: 2
- **User-Facing Packages**: 2
- **Cloud Packages**: 6
- **Infrastructure Packages**: 2

## Package Locations in Workspace

All packages follow pnpm workspace conventions:
- `core/*` - Core packages
- `core/gateways/*` - Gateway packages
- `core/protocols/*` - Protocol packages
- `core/types/*` - Type packages
- `cloud/*` - Cloud packages
- `cloud/backend/*` - Backend packages
- `use-fireproof/` - Main user package
- `dashboard/` - Dashboard app
- `cli/` - CLI tool
- `vendor/` - Vendor patches

## Building Packages

All packages use the same build commands:

```bash
# Build TypeScript
pnpm run build

# Build and create npm package
pnpm run pack

# Build and publish
pnpm run publish
```

Builds are handled by `@fireproof/core-cli` which:
- Compiles TypeScript
- Generates package.json exports
- Resolves workspace dependencies
- Manages versions

