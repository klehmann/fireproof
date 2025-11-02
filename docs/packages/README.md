# Package Documentation

This directory contains detailed documentation for all Fireproof packages organized by category.

## Package Categories

### Core Packages
Core database implementation, storage, and runtime utilities.

- [Core Packages](./core.md)
  - `@fireproof/core-base` - Ledger, CRDT, and database implementation
  - `@fireproof/core-blockstore` - Encrypted block storage
  - `@fireproof/core` - Main entry point
  - `@fireproof/core-runtime` - Runtime utilities and crypto
  - `@fireproof/core-keybag` - Key management
  - `@fireproof/core-types-*` - TypeScript type definitions

### Gateway Packages
Storage backend implementations for different platforms.

- [Gateway Packages](./gateways.md)
  - `@fireproof/core-gateways-base` - Base gateway interface
  - `@fireproof/core-gateways-file` - Platform-agnostic file gateway
  - `@fireproof/core-gateways-file-node` - Node.js file gateway
  - `@fireproof/core-gateways-file-deno` - Deno file gateway
  - `@fireproof/core-gateways-indexeddb` - Browser IndexedDB gateway
  - `@fireproof/core-gateways-memory` - In-memory gateway
  - `@fireproof/core-gateways-cloud` - Cloud sync gateway

### Protocol Packages
Communication protocols for data synchronization.

- [Protocol Packages](./protocols.md)
  - `@fireproof/core-protocols-cloud` - Cloud sync protocol
  - `@fireproof/core-protocols-dashboard` - Dashboard protocol

### User-Facing Packages
Public API and user-facing functionality.

- [User-Facing Packages](./user-facing.md)
  - `use-fireproof` - Main package (React hooks + core API)
  - `@fireproof/dashboard` - Web dashboard application

### Cloud Packages
Backend services and infrastructure.

- [Cloud Packages](./cloud.md)
  - `@fireproof/cloud-base` - Base cloud utilities
  - `@fireproof/cloud-backend-base` - Base backend server
  - `@fireproof/cloud-backend-cf-d1` - Cloudflare Workers backend
  - `@fireproof/cloud-backend-node` - Node.js backend
  - `@fireproof/cloud-todo-app` - Example todo app
  - `@fireproof/cloud-3rd-party` - Third-party integration example

### Infrastructure Packages
Build tools and development utilities.

- [Infrastructure Packages](./infrastructure.md)
  - `@fireproof/core-cli` - Build and development CLI
  - `@fireproof/vendor` - Vendor patches for ESM compatibility

## Package Dependency Graph

```
use-fireproof
├── @fireproof/core-base
│   ├── @fireproof/core-blockstore
│   │   ├── @fireproof/core-gateways-*
│   │   ├── @fireproof/core-keybag
│   │   └── @fireproof/core-runtime
│   └── @fireproof/core-types-*
├── @fireproof/core-gateways-cloud
└── @fireproof/core-protocols-dashboard

@fireproof/dashboard
├── use-fireproof
├── @fireproof/core-protocols-cloud
└── @fireproof/core-protocols-dashboard

@fireproof/cloud-backend-*
├── @fireproof/cloud-backend-base
│   ├── @fireproof/cloud-base
│   ├── @fireproof/core-protocols-cloud
│   └── @fireproof/core-gateways-cloud
└── drizzle-orm, hono, etc.
```

## Package Installation

### Main Package (Recommended)

```bash
npm install use-fireproof
```

This includes:
- React hooks
- Core database API
- File handling
- Cloud connection helpers

### Core Only

```bash
npm install @fireproof/core
```

For environments without React or when you only need the core API.

### Individual Packages

Install specific packages as needed:

```bash
npm install @fireproof/core-blockstore
npm install @fireproof/core-gateways-indexeddb
# etc.
```

## Package Development

All packages use the same build system:

```bash
# Build
pnpm run build

# Build and pack
pnpm run pack

# Publish
pnpm run publish
```

Packages are built using `@fireproof/core-cli` which handles:
- TypeScript compilation
- Package.json generation
- Workspace dependency resolution
- Version management

## Monorepo Structure

Fireproof uses pnpm workspaces. All packages are located in:
- `core/*` - Core packages
- `cloud/*` - Cloud packages
- `use-fireproof/` - Main user package
- `dashboard/` - Dashboard app
- `cli/` - CLI tool
- `vendor/` - Vendor patches

Each package has its own:
- `package.json` - Package metadata
- `tsconfig.json` - TypeScript configuration
- `index.ts` - Main entry point
- `dist/` - Compiled output (generated)

