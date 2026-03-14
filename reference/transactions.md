# MeshJS Transaction Building (Browser)

MeshJS provides a high-level transaction builder for browser-based Cardano dApps.
This document covers the `BrowserWallet`, `KoiosProvider`, and `Transaction` classes
used for delegation and other on-chain operations from the client side.

For server-side transaction building with `AppWallet` and `MeshTxBuilder`, see the
`cardano-skill/reference/offchain.md` companion document.

## Dynamic Import Pattern

MeshJS has Node-only dependencies that break SSR bundling in frameworks like Astro
or Next.js. Always use dynamic import:

```typescript
// WRONG: Top-level import breaks SSR
import { Transaction, KoiosProvider, BrowserWallet } from '@meshsdk/core';

// RIGHT: Dynamic import at call site
const { Transaction, KoiosProvider, BrowserWallet } = await import('@meshsdk/core');
```

This ensures MeshJS is only loaded in the browser context, avoiding server-side
import failures and keeping SSR output clean.

## BrowserWallet

`BrowserWallet.enable()` connects to a CIP-30 wallet extension and returns a MeshJS
wallet handle. This wraps the native CIP-30 `enable()` call with additional
convenience methods:

```typescript
// Connect to the user's wallet
const wallet = await BrowserWallet.enable('eternl');

// Get reward/stake addresses
const rewardAddresses = await wallet.getRewardAddresses();
const rewardAddress = rewardAddresses[0];

// Get used payment addresses
const usedAddresses = await wallet.getUsedAddresses();
```

`BrowserWallet.enable()` is the MeshJS equivalent of `window.cardano.eternl.enable()`.
Both trigger the same CIP-30 handshake -- use `BrowserWallet` when you need MeshJS
transaction building, use the raw CIP-30 API for lightweight wallet state queries.

## KoiosProvider

`KoiosProvider` connects to the Koios API for UTxO fetching, transaction submission,
and protocol parameters:

```typescript
// Mainnet -- uses Koios public API
const provider = new KoiosProvider('api');

// Testnet (preview)
const provider = new KoiosProvider('preview');

// Custom endpoint
const provider = new KoiosProvider('https://your-koios-instance.com/api/v1');
```

The provider is passed to the `Transaction` constructor as both `fetcher` and
`submitter`:

```typescript
const tx = new Transaction({
  initiator: wallet,    // BrowserWallet -- signs and provides UTxOs
  fetcher: provider,    // Fetches UTxOs and protocol params
  submitter: provider,  // Submits signed transactions
});
```

## Building a Delegation Transaction

The complete flow for delegating to a stake pool:

```typescript
const isBrowser = typeof window !== 'undefined';

// Pool IDs in hex format
const POOL_IDS_HEX: Record<string, string> = {
  ADV:  '3116c834a09b0060aef7284f63d3275456364e3309b3c19ec328af60',
  ADV2: '3b3327c0a885ba7c1ebeec8b44158aab79c32148d45b4c701344cd97',
  ADV3: 'edfa208d441511f9595ba80e8f3a7b07b6a80cbc9dda9d8e9d1dc039',
  ADV4: '2d8a81b805c360e9d6de6185b763f0556528d75a4561530e8ce6337c',
};

async function delegate(walletName: string, poolTicker: string): Promise<string> {
  if (!isBrowser) throw new Error('Delegation only available in browser');

  // Dynamic import to avoid SSR issues
  const { Transaction, KoiosProvider, BrowserWallet } = await import('@meshsdk/core');

  // Initialize provider
  const provider = new KoiosProvider('api');

  // Connect wallet
  const wallet = await BrowserWallet.enable(walletName);

  // Get reward address
  const rewardAddresses = await wallet.getRewardAddresses();
  if (!rewardAddresses?.length) throw new Error('No reward address found');
  const rewardAddress = rewardAddresses[0];

  // Convert pool ID hex to bech32
  const poolId = hexToBech32PoolId(POOL_IDS_HEX[poolTicker.toUpperCase()]);

  // Build transaction
  const tx = new Transaction({
    initiator: wallet,
    fetcher: provider,
    submitter: provider,
  });

  // Register stake address (2 ADA deposit for first-time delegation)
  tx.registerStake(rewardAddress);
  // Delegate to pool
  tx.delegateStake(rewardAddress, poolId);

  // Build, sign, and submit
  const unsignedTx = await tx.build();
  const signedTx = await wallet.signTx(unsignedTx);
  const txHash = await wallet.submitTx(signedTx);

  return txHash;
}
```

### Hex to Bech32 Pool ID

Pool IDs are stored as hex internally but the delegation API expects bech32:

```typescript
import { bech32 } from 'bech32';

function hexToBech32PoolId(hexId: string): string {
  const bytes = Buffer.from(hexId, 'hex');
  const words = bech32.toWords(bytes);
  return bech32.encode('pool', words);
}

// Usage
const poolId = hexToBech32PoolId('3116c834a09b0060aef7284f63d3275456364e3309b3c19ec328af60');
// -> pool1xytdmzh5pfc6z27y3nkzqatya6dw3nfqw9nrg0k2rjf77rc0r30
```

## Transaction Lifecycle

### 1. Build

`tx.build()` serializes the transaction into CBOR, automatically:
- Selects UTxOs from the wallet to cover fees and outputs
- Calculates the minimum fee based on protocol parameters
- Balances the transaction (change back to wallet)
- Returns unsigned CBOR hex string

### 2. Sign

`wallet.signTx(unsignedTx)` prompts the user via their wallet extension to review
and sign the transaction. The wallet adds the cryptographic witness. Returns signed
CBOR hex.

For multi-sig transactions, pass `partialSign: true`:
```typescript
const signedTx = await wallet.signTx(unsignedTx, true);
```

### 3. Submit

`wallet.submitTx(signedTx)` sends the signed transaction to the network via the
wallet's connected node. Returns the transaction hash on success.

Alternatively, submit via the provider:
```typescript
const txHash = await provider.submitTx(signedTx);
```

## Stake Registration

First-time delegation requires a stake key registration certificate, which costs
a 2 ADA refundable deposit. For wallets that have already registered:

```typescript
// TODO: Check registration status before adding registerStake
// If already registered, registerStake will fail with "StakeKeyAlreadyRegistered"
tx.registerStake(rewardAddress);
tx.delegateStake(rewardAddress, poolId);
```

In practice, you should query the stake address registration status (via Koios or
your own API) before including `registerStake()`. If already registered, skip it.

## React Hook Pattern

Wrap the delegation logic in a React hook for clean component integration:

```typescript
interface UseDelegationReturn {
  isProcessing: boolean;
  error: string | null;
  txHash: string | null;
  delegate: (walletName: string, poolTicker: string) => Promise<string>;
  reset: () => void;
}

export function useDelegation(): UseDelegationReturn {
  const [isProcessing, setIsProcessing] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [txHash, setTxHash] = useState<string | null>(null);

  const delegate = useCallback(async (walletName: string, poolTicker: string) => {
    setIsProcessing(true);
    setError(null);
    try {
      // ... transaction building logic ...
      setTxHash(txHashResult);
      return txHashResult;
    } catch (err) {
      const message = err instanceof Error ? err.message : 'Delegation failed';
      setError(message);
      throw err;
    } finally {
      setIsProcessing(false);
    }
  }, []);

  return { isProcessing, error, txHash, delegate, reset };
}
```

## Error Handling

Common errors during transaction building:

| Error | Cause | Recovery |
|-------|-------|----------|
| `user declined` | User rejected in wallet popup | Show "Transaction cancelled" |
| `Insufficient funds` | Not enough ADA for delegation + fee | Show balance requirement |
| `StakeKeyAlreadyRegistered` | Called `registerStake` on already-registered key | Skip registration |
| Network timeout | Koios or node unreachable | Retry or show network error |
| `Unknown pool` | Invalid pool ticker/ID | Validate before building |

## Cross-Reference: Server-Side Patterns

For server-side transaction building (used in automated scripts, backend services,
or E2E test harnesses), see `cardano-skill/reference/offchain.md` which covers:

- `AppWallet` with spending/staking keys
- `MeshTxBuilder` for low-level control
- Manual `exUnits` for script transactions
- `selectUtxosFrom()` for UTxO selection control
- Ogmios evaluator integration

## See Also

- [CIP-30 Standard](cip30.md) -- raw wallet API used by BrowserWallet internally
- [Wallet Integration](wallet-integration.md) -- React state management around wallet
- [API Resilience](api-resilience.md) -- fetching delegation status from your API
