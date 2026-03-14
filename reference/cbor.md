# CBOR Encoding and Decoding

CBOR (Concise Binary Object Representation, RFC 8949) is the wire format for all
Cardano on-chain data. CIP-30 wallet APIs return balances, UTxOs, and transaction
bodies as hex-encoded CBOR. This document covers the minimal decoding needed for
dApp frontend work.

## CBOR Major Types

Each CBOR data item starts with a byte where the high 3 bits encode the major type
and the low 5 bits encode additional information (length or value):

| Major Type | Bits 7-5 | Meaning | Additional Info |
|-----------|----------|---------|-----------------|
| 0 | `000` | Unsigned integer | Value or length prefix |
| 1 | `001` | Negative integer | -1 - value |
| 2 | `010` | Byte string | Length of bytes |
| 3 | `011` | Text string | Length of text |
| 4 | `100` | Array | Number of items |
| 5 | `101` | Map | Number of pairs |
| 6 | `110` | Tag | Tag number |
| 7 | `111` | Float/simple | Special values |

### Additional Information Encoding

The low 5 bits (0-31) encode the value or a length indicator:

| Additional | Meaning |
|-----------|---------|
| 0-23 | Immediate value |
| 24 | 1-byte value follows |
| 25 | 2-byte value follows |
| 26 | 4-byte value follows |
| 27 | 8-byte value follows |
| 31 | Indefinite length |

## Balance Decoding

CIP-30 `getBalance()` returns CBOR-encoded values in two formats:

### Pure ADA (Lovelace Only)

A single unsigned integer:

```
1a3b9aca00  =  CBOR unsigned int, 4-byte value, 1000000000 (1000 ADA)
 |   |
 |   +-- 0x3B9ACA00 = 1,000,000,000
 +-- 0x1A = major type 0 (unsigned int), additional 26 (4-byte value follows)
```

### Multi-Asset Balance

When the wallet holds native tokens, the balance is a CBOR array:

```
[lovelace, { policyId: { assetName: quantity } }]
```

Encoded as:
```
82              -- Array of 2 items (major type 4, additional 2)
  1a3b9aca00    -- First item: 1,000,000,000 lovelace
  a1            -- Second item: Map of 1 pair
    ...policy/asset data...
```

### Minimal Decoder

This decoder handles both formats by reading the first unsigned integer. If it
encounters an array (major type 4), it reads the first element:

```typescript
function decodeCborBalance(hex: string): number | null {
  try {
    const bytes = new Uint8Array(
      hex.match(/.{1,2}/g)!.map(byte => parseInt(byte, 16))
    );
    let offset = 0;

    const readUint = (): number => {
      const first = bytes[offset++];
      const majorType = first >> 5;
      const additional = first & 0x1f;

      if (majorType !== 0 && majorType !== 4) {
        throw new Error('Unsupported CBOR type');
      }

      // Array: read first element (lovelace)
      if (majorType === 4) {
        return readUint();
      }

      // Unsigned integer
      if (additional <= 23) return additional;
      if (additional === 24) return bytes[offset++];
      if (additional === 25) {
        const val = (bytes[offset] << 8) | bytes[offset + 1];
        offset += 2;
        return val;
      }
      if (additional === 26) {
        const val = (bytes[offset] << 24) | (bytes[offset + 1] << 16) |
                    (bytes[offset + 2] << 8) | bytes[offset + 3];
        offset += 4;
        return val >>> 0;  // Unsigned right shift to avoid sign bit
      }
      if (additional === 27) {
        let val = BigInt(0);
        for (let i = 0; i < 8; i++) {
          val = (val << BigInt(8)) | BigInt(bytes[offset++]);
        }
        return Number(val);  // Safe up to 2^53 (9 quadrillion lovelace)
      }
      throw new Error('Invalid CBOR additional info');
    };

    return readUint();
  } catch {
    return null;
  }
}
```

### Key Implementation Notes

- **`>>> 0` for 4-byte values**: JavaScript bitwise operators work on signed 32-bit
  integers. Values above 2^31 (2.147 billion lovelace = 2147 ADA) would be negative
  without the unsigned right shift.

- **`BigInt` for 8-byte values**: JavaScript `Number` is 64-bit float, safe to
  2^53 (9,007,199,254,740,992 lovelace = ~9 billion ADA). This exceeds the total
  ADA supply, so `Number(bigint)` is safe.

- **Always return `null` on failure**: Malformed CBOR from broken wallets or empty
  balances should not crash the UI.

## ADA Handle Extraction from UTxO CBOR

ADA Handles are NFTs minted under the Handle policy ID. To find a user's handle,
search their UTxOs for the policy ID and decode the asset name:

### ADA Handle Constants

```typescript
// ADA Handle policy ID (mainnet)
const ADA_HANDLE_POLICY_ID = 'f0ff48bbb7bbe9d59a40f1ce90e9e9d0ff5002ec48f232b49ca0fb9a';

// CIP-68 reference token prefix (label 222 = 0x000de140)
const CIP68_REF_PREFIX = '000de140';
```

### CIP-68 Reference Tokens

CIP-68 changed how token metadata works. Instead of a single token, there are:
- A **reference token** (label 222, prefix `000de140`) that holds on-chain metadata
- A **user token** (label 222, prefix `000643b0`) held in the user's wallet

Both carry the handle name after the prefix. The extraction function must strip
the CIP-68 prefix before decoding the name.

### Hex to Text Conversion

```typescript
function hexToText(hex: string): string {
  const bytes = hex.match(/.{1,2}/g)?.map(byte => parseInt(byte, 16)) || [];
  return String.fromCharCode(...bytes);
}
```

### CBOR Byte String Parsing

After finding the policy ID in the UTxO hex, the asset name follows as a CBOR
byte string:

```
a1                          -- Map of 1 entry
  4X <asset_name_hex> 01    -- Byte string of length X, then quantity
```

- `0x40-0x57` = byte string with inline length (0-23 bytes)
- `0x58` = byte string with 1-byte length prefix

### Handle Validation

Valid handles contain only lowercase alphanumeric characters, dots, dashes, and
underscores:

```typescript
// Valid: "russ", "my.handle", "test-handle_1"
// Invalid: "Russ" (uppercase), "" (empty), "@handle" (special chars)
if (/^[a-z0-9_.-]+$/.test(handleName) && handleName.length >= 1) {
  return '$' + handleName;  // Prefix with $ for display
}
```

## Common CBOR Values

Quick reference for values you'll encounter:

| Hex | Decoded | Meaning |
|-----|---------|---------|
| `00` | 0 | Zero |
| `01` | 1 | One |
| `18 64` | 100 | 100 (1-byte additional) |
| `19 03e8` | 1000 | 1000 (2-byte additional) |
| `1a 3b9aca00` | 1000000000 | 1000 ADA in lovelace |
| `1b 00000002540be400` | 10000000000 | 10000 ADA in lovelace |
| `82` | array(2) | Array of 2 items |
| `a1` | map(1) | Map of 1 key-value pair |
| `44` | bytes(4) | Byte string of 4 bytes |
| `58 20` | bytes(32) | Byte string of 32 bytes (1-byte length) |

## See Also

- [CIP-30 Standard](cip30.md) -- API methods that return CBOR
- [Gotchas](gotchas.md) -- CBOR-related pitfalls
- [ADA Handle Example](../examples/ada-handle.md) -- complete extraction code
- [CBOR Balance Example](../examples/cbor-balance.md) -- complete decoder code
