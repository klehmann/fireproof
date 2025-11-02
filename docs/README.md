# Fireproof Codebase Documentation

Welcome to the Fireproof codebase documentation. This directory contains comprehensive documentation about the Fireproof project structure, architecture, and individual packages.

## Table of Contents

- [Quick Start Guide](./QUICK_START.md) - Get started quickly
- [Project Overview](#project-overview)
- [Architecture](./ARCHITECTURE.md) - Deep dive into system architecture
- [CRDT and Conflict Resolution](./CRDT.md) - CRDT implementation and automatic conflict resolution
- [Network Topology](./NETWORK_TOPOLOGY.md) - Client-server architecture and server synchronization
- [Metadata and Synchronization](./METADATA_SYNC.md) - Role of metadata during sync and multiple heads
- [Changes API and Advanced Indexing](./CHANGES_API.md) - Change tracking, query API, and powerful indexing patterns
- [Advanced Indexing Projects](./ADVANCED_INDEXING.md) - Vector search, full-text indexing, and faceted search integration ideas
- [Time-Travel Queries](./TIME_TRAVEL.md) - Query historical states, time-travel vector search, and temporal analysis
- [File Attachments](./ATTACHMENTS.md) - How file attachments are stored and synchronized
- [Keybag and Encryption](./KEYBAG.md) - Encryption key generation and keybag management
- [Write-Ahead Log (WAL)](./WAL.md) - WAL queue and asynchronous synchronization
- [Complete Package List](./PACKAGES.md) - All packages at a glance
- [Packages](./packages/) - Detailed package documentation
  - [Package Overview](./packages/README.md)
  - [Core Packages](./packages/core.md)
  - [Gateway Packages](./packages/gateways.md)
  - [Protocol Packages](./packages/protocols.md)
  - [Cloud Packages](./packages/cloud.md)
  - [User-Facing Packages](./packages/user-facing.md)
  - [Infrastructure Packages](./packages/infrastructure.md)

## Project Overview

Fireproof is a lightweight embedded document database with encrypted live sync, designed to make browser apps easy. It provides a unified API that works in any JavaScript environment (Node.js, Deno, Bun, and browsers) with React hooks support.

### Key Features

- **Local-First**: Works offline with automatic persistence
- **Real-Time Sync**: Live queries and subscriptions for collaborative editing
- **Encrypted**: End-to-end encryption with content-addressed storage
- **CRDT-Based**: Conflict-free replicated data types for multi-writer scenarios
- **IPFS-Compatible**: Uses IPLD/CAR format for content-addressed storage
- **Universal**: Works in browsers, Node.js, Deno, and Bun

### Technology Stack

- **TypeScript**: Full type safety across the codebase
- **IPLD/CAR**: Content-addressed archives for immutable data
- **Merkle Trees**: Hash-based verification (based on Alan Shaw's Pail)
- **Prolly Trees**: Efficient indexing (based on Mikeal Rogers' implementation)
- **React**: Hooks for reactive UI updates
- **pnpm**: Monorepo package management

## Project Structure

```
fireproof/
├── core/                    # Core database implementation
│   ├── base/               # Base ledger and CRDT logic
│   ├── blockstore/         # Encrypted blockstore implementation
│   ├── core/               # Main entry point (@fireproof/core)
│   ├── runtime/            # Runtime utilities and crypto
│   ├── keybag/             # Key management for encryption
│   ├── gateways/           # Storage gateway implementations
│   ├── protocols/          # Sync protocols (cloud, dashboard)
│   └── types/              # TypeScript type definitions
├── cloud/                  # Cloud backend services
│   ├── base/               # Base cloud utilities
│   ├── backend/            # Backend implementations
│   │   ├── base/           # Base backend server
│   │   ├── cf-d1/          # Cloudflare D1 backend
│   │   └── node/           # Node.js backend
│   └── todo-app/           # Example todo application
├── use-fireproof/          # Main user-facing package (React hooks + core API)
├── dashboard/              # Web dashboard application
├── cli/                     # Build and development CLI tools
├── vendor/                  # Vendor patches for ESM compatibility
├── scripts/                 # Utility scripts
├── smoke/                   # Smoke tests for different environments
└── examples/                # Example applications
```

## Getting Started

### Installation

```bash
# Install main package (includes React hooks)
npm install use-fireproof

# Or install just the core
npm install @fireproof/core
```

### Basic Usage

```javascript
import { fireproof } from "@fireproof/core";

const db = fireproof("my-app");

// Put a document
await db.put({ _id: "doc1", name: "Example", value: 42 });

// Get a document
const doc = await db.get("doc1");

// Query documents
const result = await db.query("value", { range: [0, 100] });

// Subscribe to changes
db.subscribe(() => {
  console.log("Database updated!");
});
```

### React Usage

```javascript
import { useFireproof } from "use-fireproof";

function App() {
  const { database, useLiveQuery, useDocument } = useFireproof("my-app");
  
  const { docs } = useLiveQuery("timestamp", { descending: true });
  const { doc, merge, save } = useDocument({ text: "" });
  
  // ... use in your UI
}
```

## Development

### Prerequisites

- Node.js >= 22
- pnpm

### Setup

```bash
pnpm install
```

### Build

```bash
pnpm run build
```

### Test

```bash
pnpm run test
```

### Development Commands

- `pnpm run dev` - Start development servers
- `pnpm run build:all` - Build all packages
- `pnpm run smoke` - Run smoke tests
- `pnpm run lint` - Lint codebase
- `pnpm run format` - Format codebase

## Contributing

Fireproof is an open-source project. Contributions are welcome! Please join the [Discord](https://discord.gg/cCryrNHePH) to discuss contributions.

## License

Dual-licensed under [MIT or Apache 2.0](./LICENSE.md)

