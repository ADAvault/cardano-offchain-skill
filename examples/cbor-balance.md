# Example: CBOR Balance Decoder

Minimal CBOR decoder for CIP-30 `getBalance()` responses. Handles pure ADA
(unsigned integer) and multi-asset (array with lovelace as first element).

## Key Concepts

- **CBOR major type 0**: Unsigned integer -- pure ADA balance in lovelace
- **CBOR major type 4**: Array -- multi-asset balance, first element is lovelace
- **Additional info encoding**: Low 5 bits encode the value or length prefix
- **`>>> 0` for 4-byte values**: Unsigned right shift prevents negative numbers from sign bit
- **`BigInt` for 8-byte values**: JavaScript `Number` is safe to 2^53 (~9 billion ADA)
- **Always return `null` on failure**: Malformed CBOR must not crash the UI

## Source Code

```typescript
/**
 * Minimal CBOR decoder for CIP-30 balance responses.
 *
 * CIP-30 getBalance() returns hex-encoded CBOR:
 * - Pure ADA: unsigned integer (major type 0) in lovelace
 * - Multi-asset: array [lovelace, {policy: {asset: qty}}] (major type 4)
 *
 * This decoder reads the first unsigned integer, handling both formats.
 *
 * @param hex - Hex-encoded CBOR string from getBalance()
 * @returns Balance in lovelace, or null if decoding fails
 */
function decodeCborBalance(hex: string): number | null {
  try {
    // Convert hex string to byte array
    const bytes = new Uint8Array(
      hex.match(/.{1,2}/g)!.map(byte => parseInt(byte, 16))
    );
    let offset = 0;

    const readUint = (): number => {
      const first = bytes[offset++];
      const majorType = first >> 5;       // High 3 bits
      const additional = first & 0x1f;     // Low 5 bits

      // Only handle unsigned int (type 0) and array (type 4)
      if (majorType !== 0 && majorType !== 4) {
        throw new Error('Unsupported CBOR type');
      }

      // Array: read the first element (lovelace amount)
      if (majorType === 4) {
        return readUint();
      }

      // Unsigned integer decoding based on additional info
      if (additional <= 23) {
        // Immediate value (0-23)
        return additional;
      }
      if (additional === 24) {
        // 1-byte value follows
        return bytes[offset++];
      }
      if (additional === 25) {
        // 2-byte value follows (big-endian)
        const val = (bytes[offset] << 8) | bytes[offset + 1];
        offset += 2;
        return val;
      }
      if (additional === 26) {
        // 4-byte value follows (big-endian)
        // Use >>> 0 to convert to unsigned (JS bitwise ops use signed 32-bit)
        const val = (bytes[offset] << 24) | (bytes[offset + 1] << 16) |
                    (bytes[offset + 2] << 8) | bytes[offset + 3];
        offset += 4;
        return val >>> 0;
      }
      if (additional === 27) {
        // 8-byte value follows -- use BigInt for precision
        let val = BigInt(0);
        for (let i = 0; i < 8; i++) {
          val = (val << BigInt(8)) | BigInt(bytes[offset++]);
        }
        // Safe: max ADA supply is ~45B ADA = 45e15 lovelace << 2^53
        return Number(val);
      }

      throw new Error('Invalid CBOR additional info');
    };

    return readUint();
  } catch {
    return null;
  }
}
```

## Example Decoding

### Pure ADA: 1000 ADA (1,000,000,000 lovelace)

```
Input hex: "1a3b9aca00"

Byte 0: 0x1A = 0b00011010
  Major type: 0b000 = 0 (unsigned integer)
  Additional:  0b11010 = 26 (4-byte value follows)

Bytes 1-4: 0x3B9ACA00 = 1,000,000,000

Result: 1000000000
Display: (1000000000 / 1000000).toLocaleString() = "1,000.00" ADA
```

### Small Value: 5 lovelace

```
Input hex: "05"

Byte 0: 0x05 = 0b00000101
  Major type: 0b000 = 0 (unsigned integer)
  Additional:  0b00101 = 5 (immediate value)

Result: 5
```

### Multi-Asset: 500 ADA + tokens

```
Input hex: "821a1dcd6500a1..."

Byte 0: 0x82 = 0b10000010
  Major type: 0b100 = 4 (array)
  Additional:  0b00010 = 2 (2 items)

Recurse to read first item:
  Byte 1: 0x1A = unsigned int, 4-byte value
  Bytes 2-5: 0x1DCD6500 = 500,000,000

Result: 500000000 (500 ADA)
Note: Token data (second array element) is ignored.
```

### Large Balance: 10,000 ADA (10,000,000,000 lovelace)

```
Input hex: "1b00000002540be400"

Byte 0: 0x1B = 0b00011011
  Major type: 0b000 = 0 (unsigned integer)
  Additional:  0b11011 = 27 (8-byte value follows)

Bytes 1-8: 0x00000002540BE400 = 10,000,000,000

Uses BigInt path. Number(10000000000n) = 10000000000
Result: 10000000000
```

## Formatting for Display

```typescript
function formatBalance(lovelace: number | null): string {
  if (lovelace === null) return '---';
  return (lovelace / 1_000_000).toLocaleString(undefined, {
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
  });
}

// Examples:
formatBalance(1_000_000_000)  // "1,000.00"
formatBalance(500_000_000)    // "500.00"
formatBalance(1_500_123)      // "1.50"
formatBalance(0)              // "0.00"
formatBalance(null)           // "---"
```

## Usage with CIP-30

```typescript
const api = await window.cardano.eternl.enable();

let balance: number | null = null;
try {
  const balanceCbor = await api.getBalance();
  balance = decodeCborBalance(balanceCbor);
} catch {
  // Balance fetch failed -- show placeholder
}

// Display
const display = formatBalance(balance);  // "1,000.00"
```

## Common Mistakes

### 1. Treating CBOR Hex as a Number

```typescript
// WRONG: Parses the hex string as if it were the number
const balance = parseInt(balanceCbor, 16);
// "1a3b9aca00" -> 113305743360 (wrong!)

// RIGHT: Decode the CBOR structure
const balance = decodeCborBalance(balanceCbor);
// "1a3b9aca00" -> 1000000000 (correct)
```

### 2. Missing `>>> 0` on 4-Byte Values

```typescript
// WRONG: Values above 2^31 become negative
const val = (bytes[0] << 24) | (bytes[1] << 16) | (bytes[2] << 8) | bytes[3];
// 0x80000000 = -2147483648

// RIGHT: Unsigned right shift converts to unsigned
const val = ((bytes[0] << 24) | (bytes[1] << 16) | (bytes[2] << 8) | bytes[3]) >>> 0;
// 0x80000000 = 2147483648
```

### 3. Not Handling Multi-Asset Format

```typescript
// WRONG: Assumes pure integer, fails on multi-asset CBOR
const first = bytes[0];
if ((first >> 5) !== 0) throw new Error('Not an integer');

// RIGHT: Handle array (major type 4) by reading first element
if (majorType === 4) return readUint();  // Recurse into first array item
```

### 4. Crashing UI on Decode Failure

```typescript
// WRONG: Exception propagates, crashes component
const balance = decodeCborBalance(balanceCbor)!;

// RIGHT: Handle null gracefully
const balance = decodeCborBalance(balanceCbor);
const display = balance !== null ? formatBalance(balance) : '---';
```

### 5. Precision Loss with Large Values

```typescript
// WRONG: Direct number parsing for 8-byte values loses precision beyond 2^53
const val = Number('0x' + hex.substring(2));

// RIGHT: Use BigInt for 8-byte decoding, then convert
let val = BigInt(0);
for (let i = 0; i < 8; i++) {
  val = (val << 8n) | BigInt(bytes[offset++]);
}
return Number(val);  // Safe for ADA amounts (max ~45B ADA = 45e15 < 2^53)
```

## See Also

- [ADA Handle](ada-handle.md) -- related CBOR parsing for UTxOs
- [CBOR Reference](../reference/cbor.md) -- full CBOR major type table
- [Wallet Connect](wallet-connect.md) -- balance decoding in context
