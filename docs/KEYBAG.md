# Keybag and Encryption Key Management

Fireproof uses a keybag system to manage encryption keys for ledgers. This document explains how keys are generated, stored, and how you can customize keybag storage for different ledgers.

## Overview

The keybag is responsible for:
- **Key Storage**: Persisting encryption keys securely
- **Key Generation**: Automatically generating encryption keys for new stores
- **Key Retrieval**: Loading keys when needed for encryption/decryption
- **Key Naming**: Associating keys with named stores via fingerprint-based lookup

## Key Generation

### Automatic Key Generation

When a new store (CAR, File, Meta, or WAL) is created, Fireproof automatically generates encryption keys if they don't already exist.

**Location**: `core/keybag/key-bag.ts:getNamedKey`

```436:451:core/keybag/key-bag.ts
const rKbfp = await keysByFingerprint.from({
  ...this,
  keysItem: this.keysItem,
  opts: {
    materialStrOrUint8: opts.materialStrOrUint8 ?? this.keybag.rt.crypto.randomBytes(this.keybag.rt.keyLength),
    def: true,
  },
  modified: v2KeysItem.modified,
});
```

**Key Generation Process**:

1. **Random Bytes Generation**: Uses `crypto.randomBytes(keyLength)`
   - Default key length: **16 bytes** (128 bits)
   - Configurable via `KeyBagOpts.keyLength`

2. **Fingerprint Calculation**: SHA-256 hash of the key material
   ```typescript
   const fpr = base58btc.encode(
     new Uint8Array(await keybag.rt.crypto.digestSHA256(material.key))
   );
   ```

3. **Key Material Encoding**: Keys stored as base58-encoded strings
   ```typescript
   keyStr: base58btc.encode(keyMaterial)
   ```
   
   **Note on base58 vs base64**: Fireproof uses base58 (specifically base58btc) instead of base64 for key encoding because:
   - **No visually confusing characters**: Base58 excludes 0, O, I, and l which can be easily mistaken
   - **URL-safe**: Base58 doesn't use + or / characters that require URL encoding or cause issues in filesystems
   - **IPFS compatibility**: Base58btc is the standard encoding used in IPFS/CID systems, maintaining interoperability with IPFS tooling and infrastructure

4. **Storage**: Keys saved in keybag provider (file, IndexedDB, or memory)

### Manual Key Specification

Keys are automatically generated when stores start. To manually specify keys, you need to programmatically set them on the keybag:

```typescript
import { fireproof } from "@fireproof/core";
import { KeyBag } from "@fireproof/core-keybag";

// Create database (keys will be auto-generated)
const db = fireproof("my-app");

// Get the keybag from the ledger
// Option A: Create KeyBag from ledger's keyBag runtime
const kb = new KeyBag(db.ledger.opts.keyBag);

// Option B: Use the loader's keyBag method (returns KeyBagIf)
// const kb = await db.ledger.crdt.blockstore.loader.keyBag();

// Option 1: Provide key material when getting a named key
// Key material can be base58-encoded string or Uint8Array
const keyMaterial = "zBUFMmu5c3VdCa4r2DZTzhR"; // base58-encoded
// or
const keyMaterial = new Uint8Array(16); // 16 bytes = default key length

// Get or create a named key with specific material
// Parameters: (name, failIfNotFound, material)
const namedKey = await kb.getNamedKey("@my-app-data@", false, keyMaterial);
if (namedKey.isOk()) {
  // Key has been created/retrieved with the specified material
}

// Option 2: Add/change key on existing named key using upsert
const existingKey = await kb.getNamedKey("@my-app-data@");
if (existingKey.isOk()) {
  const newKeyMaterial = "zH1fyizirAiYVxoaQ2XZ3Xj"; // different key
  const result = await existingKey.Ok().upsert(newKeyMaterial, true); // true = set as default
  if (result.isOk()) {
    // Key has been updated
  }
}
```

**Note**: Keys are typically managed automatically. Manual key specification is mainly useful for:
- Key sharing between devices (export/import)
- Key rotation
- Testing with specific keys
- Restoring keys from backup

The keybag runtime is accessible via `db.ledger.opts.keyBag` (which is a `KeyBagRuntime`). Create a `KeyBag` instance using `new KeyBag(db.ledger.opts.keyBag)` or use `loader.keyBag()` which returns a `KeyBagIf`.

## Keybag Storage

### Default Keybag Locations

Fireproof automatically selects a keybag storage location based on the runtime:

**Location**: `core/keybag/key-bag.ts:defaultKeyBagOpts`

```605:622:core/keybag/key-bag.ts
export function defaultKeyBagUrl(sthis: SuperThis): URI {
  let bagFnameOrUrl = sthis.env.get("FP_KEYBAG_URL");
  let url: URI;
  if (runtimeFn().isBrowser) {
    url = URI.from(bagFnameOrUrl || "indexeddb://fp-keybag");
  } else {
    if (!bagFnameOrUrl) {
      const home = sthis.env.get("HOME");
      bagFnameOrUrl = `${home}/.fireproof/keybag`;
      url = URI.from(`file://${bagFnameOrUrl}`);
    } else {
      url = URI.from(bagFnameOrUrl);
    }
  }
  const logger = ensureLogger(sthis, "defaultKeyBagUrl");
  logger.Debug().Url(url).Msg("from env");
  return url;
}
```

**Default Locations**:
- **Browser**: `indexeddb://fp-keybag` (IndexedDB storage)
- **Node.js/Deno**: `file://${HOME}/.fireproof/keybag` (file system)

### Environment Variable Override

You can override the default keybag location using the `FP_KEYBAG_URL` environment variable:

```bash
# Browser (via global)
globalThis[Symbol.for("FP_PRESET_ENV")] = {
  FP_KEYBAG_URL: "indexeddb://my-custom-keybag"
};

# Node.js/Deno
export FP_KEYBAG_URL="file:///path/to/custom/keybag"

# Or in code
process.env.FP_KEYBAG_URL = "file:///path/to/custom/keybag";
```

### Per-Ledger Keybag Override

You can specify a different keybag for each ledger:

**Location**: `core/base/ledger.ts:LedgerFactory`

```46:76:core/base/ledger.ts
export function LedgerFactory(name: string, opts?: ConfigOpts): Ledger {
  const sthis = ensureSuperThis(opts);
  const key = keyConfigOpts(sthis, name, opts);
  const item = ledgers.get(key);
  return new LedgerShell(
    item.once((key) => {
      const db = new LedgerImpl(sthis, {
        name,
        meta: opts?.meta,
        keyBag: defaultKeyBagOpts(sthis, opts?.keyBag),
        storeUrls: toStoreURIRuntime(sthis, name, opts?.storeUrls),
        // ... other options
      });
      return db;
    }),
  );
}
```

**Usage Examples**:

```typescript
import { fireproof } from "@fireproof/core";

// Ledger 1: Default keybag
const db1 = fireproof("app1");

// Ledger 2: Custom keybag URL
const db2 = fireproof("app2", {
  keyBag: {
    url: "file:///custom/path/to/keybag2"
  }
});

// Ledger 3: Different keybag with custom key length
const db3 = fireproof("app3", {
  keyBag: {
    url: "indexeddb://app3-keybag",
    keyLength: 32  // 256-bit keys instead of default 128-bit
  }
});

// Ledger 4: Memory keybag (ephemeral, not persisted)
const db4 = fireproof("app4", {
  keyBag: {
    url: "memory://ephemeral-keybag"
  }
});
```

## Keybag Providers

Fireproof supports multiple keybag storage providers:

### 1. IndexedDB Provider (Browser)

**Class**: `KeyBagProviderIndexedDB`

**Location**: `core/gateways/indexeddb/key-bag-indexeddb.ts`

- Stores keys in browser IndexedDB
- Persistent across sessions
- Default for browser environments
- URL format: `indexeddb://keybag-name`

### 2. File Provider (Node.js/Deno)

**Class**: `KeyBagProviderFile`

**Location**: `core/gateways/file/key-bag-file.ts`

- Stores keys as JSON files on filesystem
- One file per keybag
- Default for Node.js/Deno (if no `FP_KEYBAG_URL` set)
- URL format: `file:///path/to/keybag`

**Storage Format**: Each keybag stored as JSON file:
```json
{
  "name": "@my-app-data@",
  "keys": {
    "fingerprint1": {
      "key": "zBUFMmu5c3VdCa4r2DZTzhR",
      "fingerPrint": "fingerprint1",
      "default": true
    }
  }
}
```

### 3. Memory Provider (Testing/Ephemeral)

**Class**: `KeyBagProviderMemory`

**Location**: `core/keybag/key-bag-memory.ts`

- Stores keys in memory (Map)
- Not persisted (lost on process exit)
- Useful for testing
- URL format: `memory://keybag-name`

### 4. Custom Provider

You can register a custom keybag provider:

```typescript
import { registerKeyBagProviderFactory } from "@fireproof/core-keybag";

registerKeyBagProviderFactory({
  protocol: "custom:",
  factory: async (url: URI, sthis: SuperThis) => {
    // Return your custom KeyBagProvider implementation
    return new CustomKeyBagProvider(url, sthis);
  }
});
```

**KeyBagProvider Interface**:
```typescript
interface KeyBagProvider {
  get(id: string): Promise<V1StorageKeyItem | V2KeysItem | undefined>;
  set(item: V2KeysItem): Promise<void>;
  del(id: string): Promise<void>;
}
```

## Store Keys and Key Naming

### Automatic Store Key Assignment

Each store (CAR, File, Meta, WAL) gets a named key from the keybag:

**Location**: `core/blockstore/store.ts:start`

```141:151:core/blockstore/store.ts
// add storekey to url
const kb = await this.loader.keyBag();
const skRes = await kb.ensureKeyFromUrl(this._url, () => {
  const key = this._url.getParam(PARAM.KEY);
  return key as string;
});

if (skRes.isErr()) {
  return skRes as Result<URI>;
}
this._url = skRes.Ok();
```

**Location**: `core/keybag/key-bag.ts:ensureKeyFromUrl`

```488:510:core/keybag/key-bag.ts
async ensureKeyFromUrl(url: URI, keyFactory: () => string): Promise<Result<URI>> {
  // add storekey to url
  const storeKey = url.getParam(PARAM.STORE_KEY);
  if (storeKey === "insecure") {
    return Result.Ok(url);
  }
  if (!storeKey) {
    const keyName = `@${keyFactory()}@`;
    const ret = await this.getNamedKey(keyName);
    if (ret.isErr()) {
      return Result.Err(ret);
    }
    const urb = url.build().setParam(PARAM.STORE_KEY, keyName);
    return Result.Ok(urb.URI());
  }
  if (storeKey.startsWith("@") && storeKey.endsWith("@")) {
    const ret = await this.getNamedKey(storeKey);
    if (ret.isErr()) {
      return Result.Err(ret);
    }
  }
  return Result.Ok(url);
}
```

### Store Key Naming Convention

Store keys follow the pattern: `@{ledger-name}-{store-type}@`

Examples:
- `@my-app-data@` - CAR store (document data)
- `@my-app-file@` - File store (attachments)
- `@my-app-meta@` - Meta store (metadata)
- `@my-app-wal@` - WAL store (write-ahead log)

### Explicit Store Key Specification

You can explicitly specify store keys in the URL:

```typescript
const db = fireproof("my-app", {
  storeUrls: {
    data: {
      car: "indexeddb://my-app?store=car&storekey=@my-custom-key@",
      file: "indexeddb://my-app?store=file&storekey=@my-custom-key@",
      // ...
    }
  }
});
```

### Disable Encryption (Insecure)

To disable encryption for a store, use `storekey=insecure`:

```typescript
const db = fireproof("my-app", {
  storeUrls: {
    data: {
      car: "indexeddb://my-app?store=car&storekey=insecure"
    }
  }
});
```

**Warning**: This stores data unencrypted. Only use for testing or public data.

## Key Retrieval and Usage

### Key Lookup by Name

Keys are retrieved from the keybag by name:

**Location**: `core/keybag/key-bag.ts:getNamedKey`

```540:551:core/keybag/key-bag.ts
async getNamedKey(
  name: string,
  failIfNotFound = false,
  materialStrOrUint8?: string | Uint8Array,
): Promise<Result<KeysByFingerprint>> {
  const kItem = await this._namedKeyItems.get(name).once(async () => {
    const prov = await this.rt.getBagProvider();
    return new KeyBagFingerprintItem(this, prov, name);
  });
  return kItem.getNamedKey({ failIfNotFound, materialStrOrUint8 });
}
```

### Encryption Usage

When a store needs to encrypt/decrypt data, it retrieves the key:

**Location**: `core/runtime/keyed-crypto.ts:keyedCryptoFactory`

```232:255:core/runtime/keyed-crypto.ts
export async function keyedCryptoFactory(url: URI, kb: KeyBagIf, sthis: SuperThis): Promise<CryptoAction> {
  const storekey = url.getParam(PARAM.STORE_KEY);
  if (storekey && storekey !== "insecure") {
    const rkey = await kb.getNamedKey(storekey, false);
    if (rkey.isErr()) {
      throw (
        sthis.logger
          .Error()
          .Str("keybag", kb.rt.id())
          .Str("name", storekey)
          .Msg("getNamedKey failed")
          .AsError()
      );
    }
    return new cryptoAction(url, rkey.Ok(), kb.rt.crypto, sthis);
  }
  return new noCrypto(url, kb.rt.crypto, sthis);
}
```

## Key Structure

### V2 Keys Item Format

Modern keybag format (V2):

```typescript
interface V2KeysItem {
  name: string;  // e.g., "@my-app-data@"
  keys: {
    [fingerprint: string]: {
      key: string;           // base58-encoded key material
      fingerPrint: string;    // SHA-256 fingerprint
      default: boolean;       // Is this the default key?
    };
  };
}
```

### Key Fingerprints

- **Purpose**: Identify keys uniquely
- **Calculation**: SHA-256 hash of key material, base58-encoded
- **Usage**: Lookup keys by fingerprint or use default (`*`)

### Multiple Keys per Store

A single store can have multiple keys:
- Allows key rotation
- One key marked as `default: true`
- Default key used when no fingerprint specified

## Keybag Configuration Options

### KeyBagOpts Interface

**Location**: `core/types/base/types.ts`

```811:819:core/types/base/types.ts
export interface KeyBagOpts {
  // in future you can encrypt the keybag with ?masterkey=xxxxx
  readonly url: CoerceURI;
  // readonly key: string; // key to encrypt the keybag
  readonly crypto: CryptoRuntime;
  readonly keyLength: number; // default: 16
  // readonly logger: Logger;
  readonly keyRuntime: KeyBagRuntime;
}
```

**Options**:
- `url`: Keybag storage location (file://, indexeddb://, memory://)
- `keyLength`: Key size in bytes (default: 16 = 128 bits)
- `crypto`: Custom crypto runtime (optional)
- `keyRuntime`: Full keybag runtime (advanced)

## Advanced: Key Extraction

### Extractable Keys

By default, keys are **not extractable** from the CryptoKey object (for security). You can enable extraction for key export:

```typescript
const db = fireproof("my-app", {
  keyBag: {
    url: "file:///path/to/keybag?extractKey=_deprecated_internal_api"
  }
});
```

**Warning**: This enables key extraction which reduces security. Only use when you need to export keys.

### Key Export Example

```typescript
import { getKeyBag } from "@fireproof/core-keybag";

const kb = await getKeyBag(sthis, {
  url: "file:///path/to/keybag?extractKey=_deprecated_internal_api"
});

const namedKey = await kb.getNamedKey("@my-app-data@");
if (namedKey.isOk()) {
  const keyMaterial = await namedKey.Ok().extract();
  // keyMaterial.key: Uint8Array
  // keyMaterial.keyStr: string (base58-encoded)
}
```

## Keybag Sharing

### Cross-Device Key Sharing

To share data across devices, you need to share the encryption keys:

1. **Export Keys**: Extract keys from source device
2. **Transfer Keys**: Share keys securely (manual copy, encrypted channel, etc.)
3. **Import Keys**: Load keys into keybag on target device

**Example**:

```typescript
// Device 1: Export
const kb1 = await getKeyBag(sthis1);
const dataKey = await kb1.getNamedKey("@my-app-data@");
const keyMaterial = await dataKey.Ok().extract();
const exportedKey = keyMaterial.keyStr; // base58 string

// Device 2: Import
const kb2 = await getKeyBag(sthis2);
await kb2.getNamedKey("@my-app-data@", false, exportedKey);
```

## Implementation Details

### Key Generation Flow

```
1. Store created without storekey parameter
   ↓
2. KeyBag.ensureKeyFromUrl() called
   ↓
3. Generate key name: @{ledger}-{store-type}@
   ↓
4. KeyBag.getNamedKey(name) called
   ↓
5. Check if key exists in provider
   ↓
6. If not exists:
   - Generate random bytes (keyLength bytes)
   - Calculate fingerprint (SHA-256)
   - Create CryptoKey
   - Store in keybag provider
   ↓
7. Return key and add storekey to URL
```

### Key Storage Flow

```
1. Key generated or loaded
   ↓
2. Convert to V2KeysItem format
   ↓
3. Store via KeyBagProvider.set()
   ↓
4. Provider saves to:
   - IndexedDB: objectStore.put(item, name)
   - File: JSON file per keybag
   - Memory: Map.set(key, item)
```

## Best Practices

### 1. Separate Keybags per Ledger

Use different keybags for different ledgers to isolate encryption:

```typescript
const db1 = fireproof("ledger1", {
  keyBag: { url: "indexeddb://ledger1-keybag" }
});

const db2 = fireproof("ledger2", {
  keyBag: { url: "indexeddb://ledger2-keybag" }
});
```

### 2. Backup Keybags

For production applications, backup keybag storage:
- File-based: Copy keybag directory
- IndexedDB: Export database
- Consider encrypted backups

### 3. Key Rotation

To rotate keys:
1. Generate new key material
2. Upsert with new fingerprint
3. Mark as default
4. Old key remains for reading old data

```typescript
const namedKey = await keybag.getNamedKey("@my-app-data@");
await namedKey.Ok().upsert(newKeyMaterial, true); // true = set as default
```

### 4. Secure Key Transmission

When sharing keys:
- Use encrypted channels
- Consider key derivation from user password
- Implement key escrow if needed

## Related Files

- `core/keybag/key-bag.ts` - Main keybag implementation
- `core/keybag/key-bag-memory.ts` - Memory provider
- `core/gateways/indexeddb/key-bag-indexeddb.ts` - IndexedDB provider
- `core/gateways/file/key-bag-file.ts` - File provider
- `core/base/ledger.ts` - Ledger factory with keybag options
- `core/runtime/keyed-crypto.ts` - Crypto action factory
- `core/types/base/types.ts` - Keybag type definitions

