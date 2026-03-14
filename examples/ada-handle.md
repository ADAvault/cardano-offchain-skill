# Example: ADA Handle Extraction

Extract an ADA Handle ($handle) from CIP-30 UTxO hex data. Supports both legacy
handles and CIP-68 reference tokens.

## Key Concepts

- **ADA Handle policy ID**: `f0ff48bbb7bbe9d59a40f1ce90e9e9d0ff5002ec48f232b49ca0fb9a` (mainnet)
- **CIP-68 reference token prefix**: `000de140` (label 222) -- must be stripped before decoding
- **CBOR byte string parsing**: Asset names are encoded as CBOR byte strings after the policy ID
- **Hex to text**: Convert hex-encoded asset name to UTF-8 string
- **Validation regex**: `^[a-z0-9_.-]+$` -- lowercase alphanumeric, dots, dashes, underscores
- **Display format**: Prefix with `$` (e.g., `$russ`)

## Source Code

```typescript
// ADA Handle policy ID (mainnet)
const ADA_HANDLE_POLICY_ID = 'f0ff48bbb7bbe9d59a40f1ce90e9e9d0ff5002ec48f232b49ca0fb9a';

// CIP-68 reference token prefix (label 222 = 0x000de140)
const CIP68_REF_PREFIX = '000de140';

/**
 * Decode hex string to UTF-8 text.
 */
function hexToText(hex: string): string {
  const bytes = hex.match(/.{1,2}/g)?.map(byte => parseInt(byte, 16)) || [];
  return String.fromCharCode(...bytes);
}

/**
 * Extract ADA Handle from CBOR-encoded UTxOs.
 * Handles both legacy format and CIP-68 reference tokens.
 *
 * @param utxosHex - Array of hex-encoded UTxOs from CIP-30 getUtxos()
 * @returns Handle string with $ prefix (e.g., "$russ") or null
 */
function extractAdaHandle(utxosHex: string[]): string | null {
  try {
    for (const utxoHex of utxosHex) {
      // Search for the Handle policy ID in the UTxO hex
      const policyIndex = utxoHex.indexOf(ADA_HANDLE_POLICY_ID);
      if (policyIndex === -1) continue;

      // Data after the policy ID contains the asset name
      const afterPolicy = utxoHex.substring(policyIndex + ADA_HANDLE_POLICY_ID.length);

      // Parse CBOR structure after policy ID
      // Expected: optional map marker (a1-b7), then byte string with asset name
      const firstByte = parseInt(afterPolicy.substring(0, 2), 16);

      // Skip CBOR map marker if present (a1-b7 = map with 1-23 entries)
      let dataStart = 0;
      if (firstByte >= 0xa1 && firstByte <= 0xb7) {
        dataStart = 2;
      }

      // Read CBOR byte string length
      const lenByte = parseInt(afterPolicy.substring(dataStart, dataStart + 2), 16);
      let assetNameHex: string | null = null;

      // Short byte string: 0x40-0x57 = inline length 0-23
      if (lenByte >= 0x40 && lenByte <= 0x57) {
        const length = lenByte - 0x40;
        assetNameHex = afterPolicy.substring(dataStart + 2, dataStart + 2 + length * 2);
      }
      // Byte string with 1-byte length prefix: 0x58
      else if (lenByte === 0x58) {
        const length = parseInt(afterPolicy.substring(dataStart + 2, dataStart + 4), 16);
        assetNameHex = afterPolicy.substring(dataStart + 4, dataStart + 4 + length * 2);
      }

      if (assetNameHex && assetNameHex.length > 0) {
        // Check for CIP-68 reference token prefix and strip it
        if (assetNameHex.startsWith(CIP68_REF_PREFIX)) {
          assetNameHex = assetNameHex.substring(CIP68_REF_PREFIX.length);
        }

        // Decode hex to text
        const handleName = hexToText(assetNameHex);

        // Validate: lowercase alphanumeric with dots, dashes, underscores
        if (/^[a-z0-9_.-]+$/.test(handleName) && handleName.length >= 1) {
          return '$' + handleName;
        }
      }
    }
  } catch (err) {
    console.error('[Handle] Parsing error:', err);
  }
  return null;
}
```

## How It Works

### Step 1: Find Policy ID in UTxO Hex

CIP-30 `getUtxos()` returns an array of hex-encoded CBOR UTxOs. Each UTxO contains
the transaction output, which includes any native tokens as policy ID + asset name
pairs. We search for the Handle policy ID as a substring:

```
...f0ff48bbb7bbe9d59a40f1ce90e9e9d0ff5002ec48f232b49ca0fb9a...
   ^--- ADA Handle policy ID (56 hex chars = 28 bytes)
```

### Step 2: Parse CBOR After Policy ID

After the policy ID, the CBOR structure contains the asset map. The asset name
appears as a CBOR byte string:

```
f0ff48...ca0fb9a  a1  44  72757373  01
                  |   |   |         |
                  |   |   |         +-- Quantity (1)
                  |   |   +-- "russ" in hex (4 bytes)
                  |   +-- CBOR byte string, length 4 (0x44 = 0x40 + 4)
                  +-- CBOR map, 1 entry (0xa1)
```

### Step 3: Handle CIP-68 Prefix

CIP-68 reference tokens have a 4-byte prefix (label 222 = `000de140`):

```
Asset name hex: 000de14072757373
                |       |
                |       +-- "russ" in hex
                +-- CIP-68 reference token prefix

After stripping: 72757373 -> "russ"
```

### Step 4: Validate and Format

Only accept handles matching the valid pattern (lowercase, alphanumeric, dots,
dashes, underscores). Prefix with `$` for display:

```
"russ"        -> valid   -> "$russ"
"my.handle"   -> valid   -> "$my.handle"
"Russ"        -> invalid -> null (uppercase)
""            -> invalid -> null (empty)
```

## Usage with CIP-30

```typescript
const api = await window.cardano.eternl.enable();
const utxos = await api.getUtxos();

let handle: string | null = null;
if (utxos && utxos.length > 0) {
  handle = extractAdaHandle(utxos);
}

// Display: "$russ" or fallback to truncated address
const displayName = handle || formatAddress(stakeAddress);
```

## Common Mistakes

### 1. Not Handling `getUtxos()` Returning `undefined`

```typescript
// WRONG: Crashes if getUtxos returns undefined
const utxos = await api.getUtxos();
const handle = extractAdaHandle(utxos);

// RIGHT: Guard against undefined
const utxos = await api.getUtxos();
if (utxos && utxos.length > 0) {
  handle = extractAdaHandle(utxos);
}
```

### 2. Not Stripping CIP-68 Prefix

```typescript
// WRONG: Returns "\x00\x0d\xe1\x40russ" instead of "russ"
const name = hexToText(assetNameHex);

// RIGHT: Check and strip CIP-68 prefix first
if (assetNameHex.startsWith(CIP68_REF_PREFIX)) {
  assetNameHex = assetNameHex.substring(CIP68_REF_PREFIX.length);
}
const name = hexToText(assetNameHex);
```

### 3. Case-Sensitive Validation

```typescript
// WRONG: Accepts uppercase handles
if (/^[a-zA-Z0-9_.-]+$/.test(handleName)) { ... }

// RIGHT: ADA Handles are lowercase only
if (/^[a-z0-9_.-]+$/.test(handleName)) { ... }
```

### 4. Missing Error Isolation

```typescript
// WRONG: Handle parsing failure blocks wallet connect
const handle = extractAdaHandle(utxos);

// RIGHT: Wrap in try/catch so connect succeeds even if handle parsing fails
let handle: string | null = null;
try {
  const utxos = await api.getUtxos();
  if (utxos && utxos.length > 0) {
    handle = extractAdaHandle(utxos);
  }
} catch {
  // Handle fetch/parse failed -- continue without handle
}
```

## See Also

- [CBOR Balance](cbor-balance.md) -- related CBOR decoding
- [CBOR Reference](../reference/cbor.md) -- CBOR byte string encoding details
- [Wallet Connect](wallet-connect.md) -- handle extraction in context
