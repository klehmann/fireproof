# Infrastructure Packages

Infrastructure packages provide build tools, development utilities, and vendor patches.

## @fireproof/core-cli

**Location:** `cli/`

**Purpose:** Build and development CLI tool for Fireproof packages.

**Features:**
- TypeScript compilation
- Package building
- Version management
- Package publishing

**Commands:**
- `build` - Build package
- `tsc` - TypeScript compilation
- `pack` - Create npm package
- `publish` - Publish to npm

**Dependencies:**
- `cmd-ts` - CLI argument parsing
- `fs-extra` - File system utilities
- `semver` - Version management
- `zx` - Shell scripting
- `jose` - JWT operations
- `find-up` - File finding utilities

**Usage:**
```bash
# Build a package
core-cli tsc

# Build and pack
core-cli build --doPack

# Build and publish
core-cli build
```

**Scripts:**
- Build TypeScript projects
- Generate package.json exports
- Handle workspace dependencies
- Version management
- Publishing workflow

## @fireproof/vendor

**Location:** `vendor/`

**Purpose:** Vendor patches for ESM compatibility and third-party package fixes.

**Features:**
- ESM-compatible versions of CommonJS packages
- Package merging utilities
- ESM transformation

**Exports:**
- `p-limit` - ESM-compatible version of p-limit

**Dependencies:**
- `yocto-queue` - Queue implementation for p-limit

**Development Tools:**
- `merge-package.ts` - Package merging utility
- `to-esm-transform.ts` - ESM transformation

**Use Cases:**
- Fixing ESM compatibility issues
- Patching third-party packages
- Providing ESM-compatible alternatives

**Scripts:**
```bash
# Merge a package
tsx merge-package.ts --prepare --verbose 'p-limit,https://github.com/mabels/p-limit.git,pnpm'
```

