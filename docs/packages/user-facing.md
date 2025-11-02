# User-Facing Packages

Packages that provide the public API and user-facing functionality.

## use-fireproof

**Location:** `use-fireproof/`

**Purpose:** Main user-facing package providing both React hooks and core JavaScript API.

**Package Name:** `use-fireproof` (published to npm)

**Key Features:**
- React hooks for reactive UI updates
- Core JavaScript API
- File handling utilities
- Cloud connection helpers
- Authentication integration

**Exports:**

### React Hooks

```typescript
import { useFireproof } from "use-fireproof";

function App() {
  const { database, useLiveQuery, useDocument } = useFireproof("my-app");
  // ...
}
```

**Hooks:**
- `useFireproof()` - Main hook, returns database and other hooks
- `useLiveQuery()` - Reactive query hook
- `useDocument()` - Single document hook
- `useAllDocs()` - All documents hook

### Core API

```typescript
import { fireproof } from "use-fireproof";
// or
import { fireproof } from "@fireproof/core";

const db = fireproof("my-app");
```

**Cloud Integration:**

```typescript
import { toCloud } from "use-fireproof";

const attach = toCloud({
  dashboardURI: "https://dashboard.fireproof.storage",
  tokenApiURI: "https://api.fireproof.storage",
  urls: { base: "https://storage.fireproof.storage" }
});

const { database } = useFireproof("my-app", { attach });
```

**File Handling:**
- `ImgFile` component for displaying images
- File upload/download utilities
- File encoding/decoding

**Dependencies:**
- `@fireproof/core-base` - Core database
- `@fireproof/core-gateways-cloud` - Cloud sync
- `@fireproof/core-protocols-dashboard` - Dashboard protocol
- `dompurify` - HTML sanitization for file display
- `jose` - JWT handling

**Peer Dependencies:**
- `react >= 18.0.0`

**Usage Examples:**

**React Hook Usage:**
```typescript
function TodoApp() {
  const { database, useLiveQuery, useDocument } = useFireproof("todos");
  
  const { docs } = useLiveQuery("createdAt", { 
    descending: true, 
    limit: 100 
  });
  
  const { doc, merge, save } = useDocument({ 
    text: "", 
    completed: false 
  });
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    await save();
  };
  
  return (
    <div>
      <form onSubmit={handleSubmit}>
        <input 
          value={doc.text} 
          onChange={(e) => merge({ text: e.target.value })} 
        />
        <button type="submit">Add</button>
      </form>
      <ul>
        {docs.map(todo => (
          <li key={todo._id}>{todo.text}</li>
        ))}
      </ul>
    </div>
  );
}
```

**Core API Usage:**
```typescript
import { fireproof } from "use-fireproof";

const db = fireproof("my-app");

// Put a document
await db.put({ _id: "doc1", name: "Example" });

// Get a document
const doc = await db.get("doc1");

// Query documents
const result = await db.query("name", { 
  range: ["a", "z"] 
});

// Subscribe to changes
db.subscribe(() => {
  console.log("Database updated!");
});
```

## @fireproof/dashboard

**Location:** `dashboard/`

**Purpose:** Web dashboard application for managing Fireproof databases.

**Type:** Application (not published as npm package)

**Features:**
- Database management UI
- Query interface
- Document editor
- Cloud sync management
- Authentication via Clerk

**Technology Stack:**
- **React 19** - UI framework
- **Vite** - Build tool
- **Tailwind CSS** - Styling
- **React Router** - Routing
- **TanStack Query** - Data fetching
- **Monaco Editor** - Code editing
- **Clerk** - Authentication

**Backend Support:**
- Cloudflare D1 (primary)
- LibSQL (local/remote)
- Node.js server

**Key Features:**
- Database browser
- Document viewer/editor
- Query builder
- Cloud sync status
- User authentication
- Real-time updates

**Development:**
```bash
pnpm run dev              # Start dev server
pnpm run backend:d1       # Start D1 backend
pnpm run backend:deno     # Start Deno backend
pnpm run build            # Build for production
```

**Deployment:**
- Cloudflare Pages (primary)
- Can deploy to any static host

