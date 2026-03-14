# CIP-30 Wallet Standard

The CIP-30 dApp Connector standard defines how browser-based Cardano wallets expose
their API to web applications. Every compliant wallet injects an object into
`window.cardano` under a wallet-specific key.

## Extension Injection Model

Wallet browser extensions inject themselves into the page's `window.cardano` namespace.
Each wallet registers under its own key:

```
window.cardano.eternl    // Eternl wallet
window.cardano.lace      // Lace wallet
window.cardano.vespr     // Vespr wallet
window.cardano.yoroi     // Yoroi wallet
window.cardano.flint     // Flint wallet
```

The injection happens asynchronously during page load. Extensions may not be present
when your JavaScript first executes -- this is by design, not a bug. Extensions
are injected by browser extension content scripts, which may race with your application
code.

### Injection Timing

Extensions typically inject within 0-200ms of page load, but can take up to 2000ms
on slow machines, browser cold starts, or when the extension was recently installed.
Always account for late injection with retry logic.

```typescript
// Detect installed wallets -- scan twice with delay
const detectWallets = () => {
  const cardano = window.cardano;
  if (!cardano) return [];

  return SUPPORTED_WALLETS
    .filter((wallet) => cardano[wallet.name] !== undefined)
    .map((wallet) => wallet.name);
};

// Initial scan
detectWallets();
// Re-scan after 500ms for late injection
setTimeout(detectWallets, 500);
```

## Extension Object Interface

Each extension exposes a fixed set of properties and methods before `enable()`:

```typescript
interface CardanoExtension {
  name: string;          // Extension identifier (e.g., "eternl")
  icon: string;          // Base64 or data URI of wallet icon
  apiVersion: string;    // CIP-30 API version (e.g., "0.1.0")
  enable(): Promise<CardanoWalletApi>;    // Request access to wallet API
  isEnabled(): Promise<boolean>;          // Check if dApp is already authorized
}
```

## Wallet API Interface

After calling `enable()`, you receive the full CIP-30 API handle:

```typescript
interface CardanoWalletApi {
  getNetworkId(): Promise<number>;              // 1 = mainnet, 0 = testnet
  getBalance(): Promise<string>;                // CBOR hex-encoded
  getUsedAddresses(): Promise<string[]>;        // Hex-encoded payment addresses
  getUnusedAddresses(): Promise<string[]>;      // Hex-encoded payment addresses
  getRewardAddresses(): Promise<string[]>;      // Hex-encoded stake addresses
  getChangeAddress(): Promise<string>;          // Hex-encoded change address
  getUtxos(): Promise<string[] | undefined>;    // CBOR hex-encoded UTxOs
  signTx(tx: string, partialSign?: boolean): Promise<string>;  // Returns signed CBOR
  signData(addr: string, payload: string): Promise<{ signature: string; key: string }>;
  submitTx(tx: string): Promise<string>;        // Returns transaction hash
}
```

## TypeScript Type Definitions

For type-safe wallet integration, define these interfaces in your project:

```typescript
// src/types/wallet.ts

export interface WalletState {
  isConnected: boolean;
  isConnecting: boolean;
  walletName: string | null;
  stakeAddress: string | null;
  balance: number | null;   // in lovelace
  error: string | null;
}

export interface WalletInfo {
  name: string;        // CIP-30 wallet identifier (window.cardano key)
  displayName: string; // Human-readable name for UI
  icon: string;        // Path to wallet icon
}

export interface CardanoWindow {
  [walletName: string]: {
    name: string;
    icon: string;
    apiVersion: string;
    enable: () => Promise<CardanoWalletApi>;
    isEnabled: () => Promise<boolean>;
  };
}
```

## Wallet Configuration

Define supported wallets as a typed configuration array:

```typescript
// src/config/wallets.ts
import type { WalletInfo } from '../types/wallet';

export const SUPPORTED_WALLETS: WalletInfo[] = [
  { name: 'eternl',       displayName: 'Eternl',     icon: '/wallets/eternl.svg' },
  { name: 'lace',         displayName: 'Lace',       icon: '/wallets/lace.svg' },
  { name: 'flint',        displayName: 'Flint',      icon: '/wallets/flint.svg' },
  { name: 'yoroi',        displayName: 'Yoroi',      icon: '/wallets/yoroi.svg' },
  { name: 'typhoncip30',  displayName: 'Typhon',     icon: '/wallets/typhon.svg' },
  { name: 'gerowallet',   displayName: 'GeroWallet', icon: '/wallets/gero.svg' },
  { name: 'vespr',        displayName: 'Vespr',      icon: '/wallets/vespr.svg' },
];

export const WALLET_STORAGE_KEY = 'adavault_connected_wallet';

export function getWalletInfo(name: string): WalletInfo | undefined {
  return SUPPORTED_WALLETS.find((w) => w.name === name);
}
```

Note: `name` must match the key used in `window.cardano`. Some wallets use
non-obvious keys: Typhon uses `typhoncip30`, not `typhon`.

## Network ID

`getNetworkId()` returns:
- `1` = Cardano mainnet
- `0` = any testnet (preview, preprod, or custom)

Always check the network ID immediately after `enable()` and before any other API
calls. Users on the wrong network should see a clear error, not silent failures or
undefined behavior.

```typescript
const api = await window.cardano.eternl.enable();
const networkId = await api.getNetworkId();
if (networkId !== 1) {
  throw new Error('Please switch to Cardano mainnet');
}
```

## Address Encoding

All addresses from CIP-30 are hex-encoded. Convert to human-readable bech32 format:

```typescript
import { bech32 } from 'bech32';

function hexToBech32(hex: string, prefix: string): string {
  const cleanHex = hex.startsWith('0x') ? hex.slice(2) : hex;
  const bytes = Buffer.from(cleanHex, 'hex');
  const words = bech32.toWords(bytes);
  return bech32.encode(prefix, words, 200);
}

// Stake address: e0... -> stake1...
hexToBech32(stakeAddrHex, 'stake');

// Pool ID: 3116c8... -> pool1...
hexToBech32(poolIdHex, 'pool');
```

## Common Mistakes

### 1. Trusting `isEnabled()`

`isEnabled()` is supposed to return `true` if the dApp has previously been authorized.
In practice, Eternl returns `false` after page reload even for authorized sites. Never
gate auto-reconnect on `isEnabled()` -- call `enable()` directly, which returns
silently for previously-authorized dApps.

### 2. Assuming Synchronous Injection

Wallet extensions inject asynchronously. Code that runs immediately on page load
will often not find `window.cardano` or its sub-keys. Always use retry logic with
backoff when checking for wallet availability.

### 3. Treating `getBalance()` as a Number

`getBalance()` returns a CBOR hex string, not a number. For pure-ADA wallets it
encodes a single unsigned integer (lovelace). For wallets holding native tokens,
it encodes a CBOR array `[lovelace, {policy: {asset: qty}}]`. You must decode
the CBOR.

### 4. Ignoring `getUtxos()` Returning `undefined`

`getUtxos()` can return `undefined` instead of an empty array. Some wallets return
`undefined` when UTxO pagination is unsupported or when the wallet is not fully
synced. Always guard: `if (utxos && utxos.length > 0)`.

### 5. Hex vs Bech32 Confusion

CIP-30 returns hex addresses. APIs typically expect bech32 (`stake1...`, `pool1...`,
`addr1...`). Bech32 includes a human-readable prefix and checksum. Always convert
before passing to external APIs or displaying to users.

### 6. Not Handling User Rejection

`enable()` rejects if the user clicks "Cancel" in the wallet popup. The error message
varies by wallet ("user declined", "User cancelled", etc.). Catch it and display a
friendly message rather than letting it propagate as an unhandled promise rejection.

## See Also

- [Wallet Integration Patterns](wallet-integration.md) -- React hooks, state management
- [CBOR Decoding](cbor.md) -- balance and UTxO decoding
- [E2E Testing](testing.md) -- mock wallet for Playwright
