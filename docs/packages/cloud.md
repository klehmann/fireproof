# Cloud Packages

Cloud packages provide backend services and infrastructure for Fireproof.

## @fireproof/cloud-base

**Location:** `cloud/base/`

**Purpose:** Base utilities and shared code for cloud backend services.

**Features:**
- AWS S3 integration utilities
- Common cloud utilities
- Shared configuration

**Dependencies:**
- `aws4fetch` - AWS signature v4 for fetch API
- `jose` - JWT operations
- `cmd-ts` - CLI argument parsing

**Use Cases:**
- Shared code for cloud backends
- AWS S3 operations
- Common utilities

## @fireproof/cloud-backend-base

**Location:** `cloud/backend/base/`

**Purpose:** Base backend server implementation using Hono framework.

**Features:**
- Hono server setup
- Fireproof protocol endpoints
- Database abstraction (Drizzle ORM)
- Authentication integration
- CAR file handling
- Metadata management

**Technology Stack:**
- **Hono** - Web framework
- **Drizzle ORM** - Database abstraction
- **LibSQL** - SQLite-compatible database

**Dependencies:**
- `hono` - Web framework
- `drizzle-orm` - Database ORM
- `@libsql/client` - Database client
- `@fireproof/core-protocols-cloud` - Cloud protocol

**Key Components:**
- Server setup and routing
- CAR file upload/download endpoints
- Metadata endpoints
- Authentication middleware

**Use Cases:**
- Base for custom backend implementations
- Shared backend functionality
- Protocol implementation

## @fireproof/cloud-backend-cf-d1

**Location:** `cloud/backend/cf-d1/`

**Purpose:** Cloudflare Workers backend using D1 database.

**Features:**
- Cloudflare Workers runtime
- D1 database for metadata
- R2 storage for CAR files (via gateway)
- Wrangler deployment

**Technology Stack:**
- **Cloudflare Workers** - Serverless runtime
- **D1** - SQLite-compatible database
- **Drizzle ORM** - Database abstraction

**Dependencies:**
- `@fireproof/cloud-backend-base` - Base backend
- `drizzle-orm` - Database ORM
- `wrangler` - Cloudflare deployment tool

**Deployment:**
```bash
pnpm run wrangler:deploy    # Deploy to Cloudflare
```

**Configuration:**
- `wrangler.toml` - Cloudflare Workers configuration
- Drizzle configs for D1 (local and remote)

**Use Cases:**
- Production cloud backend
- Edge deployment
- Serverless architecture

## @fireproof/cloud-backend-node

**Location:** `cloud/backend/node/`

**Purpose:** Node.js backend server implementation.

**Features:**
- Node.js HTTP server
- Hono with Node.js adapter
- LibSQL database
- WebSocket support
- File system storage

**Technology Stack:**
- **Hono** - Web framework
- **@hono/node-server** - Node.js adapter
- **@hono/node-ws** - WebSocket support
- **LibSQL** - SQLite database

**Dependencies:**
- `@fireproof/cloud-backend-base` - Base backend
- `@hono/node-server` - Node.js server adapter
- `@hono/node-ws` - WebSocket support
- `drizzle-orm` - Database ORM

**Use Cases:**
- Local development server
- Self-hosted backend
- Testing and development

**Running:**
```bash
# Development
pnpm run dev:cf-d1

# Production
node dist/server.js
```

## @fireproof/cloud-todo-app

**Location:** `cloud/todo-app/`

**Purpose:** Example todo application demonstrating Fireproof usage.

**Type:** Example application

**Features:**
- Todo item CRUD operations
- Real-time synchronization
- File attachments
- React-based UI

**Dependencies:**
- `use-fireproof` - Main Fireproof package
- `react-dom` - React rendering

**Use Cases:**
- Learning example
- Testing Fireproof features
- Reference implementation

## @fireproof/cloud-3rd-party

**Location:** `cloud/3rd-party/`

**Purpose:** Third-party integration example application.

**Type:** Example application

**Features:**
- Third-party service integration
- React UI
- Vite-based development

**Dependencies:**
- `use-fireproof` - Main Fireproof package
- `react-dom` - React rendering

**Technology Stack:**
- **Vite** - Build tool
- **React** - UI framework

**Use Cases:**
- Integration examples
- Third-party service demos

