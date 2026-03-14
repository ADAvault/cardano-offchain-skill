# Example: Delegation Transaction

Build and submit a stake delegation transaction using MeshJS from the browser.
Covers dynamic import for SSR safety, BrowserWallet, KoiosProvider, stake
registration, delegation, and the build/sign/submit flow.

## Key Concepts

- **Dynamic import**: `await import('@meshsdk/core')` avoids SSR bundling issues
- **BrowserWallet**: MeshJS wrapper around CIP-30 `enable()`
- **KoiosProvider**: Connects to Koios API for UTxO fetching and tx submission
- **Transaction class**: High-level builder with `registerStake()` and `delegateStake()`
- **Pool ID bech32**: Delegation requires `pool1...` format, not hex
- **Stake registration**: First-time delegation requires 2 ADA deposit

## Source Code

```typescript
import { useState, useCallback } from 'react';
import { bech32 } from 'bech32';

const isBrowser = typeof window !== 'undefined';

// Pool IDs in hex format (stored as hex, converted to bech32 for delegation)
const POOL_IDS_HEX: Record<string, string> = {
  ADV:  '3116c834a09b0060aef7284f63d3275456364e3309b3c19ec328af60',
  ADV2: '3b3327c0a885ba7c1ebeec8b44158aab79c32148d45b4c701344cd97',
  ADV3: 'edfa208d441511f9595ba80e8f3a7b07b6a80cbc9dda9d8e9d1dc039',
  ADV4: '2d8a81b805c360e9d6de6185b763f0556528d75a4561530e8ce6337c',
};

// Convert hex pool ID to bech32 format (pool1...)
function hexToBech32PoolId(hexId: string): string {
  const bytes = Buffer.from(hexId, 'hex');
  const words = bech32.toWords(bytes);
  return bech32.encode('pool', words);
}

// Pre-compute bech32 pool IDs
export const POOL_IDS: Record<string, string> = Object.fromEntries(
  Object.entries(POOL_IDS_HEX).map(([ticker, hex]) => [ticker, hexToBech32PoolId(hex)])
);

interface DelegationState {
  isProcessing: boolean;
  error: string | null;
  txHash: string | null;
}

interface UseDelegationReturn extends DelegationState {
  delegate: (walletName: string, poolTicker: string) => Promise<string>;
  reset: () => void;
}

export function useDelegation(): UseDelegationReturn {
  const [isProcessing, setIsProcessing] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [txHash, setTxHash] = useState<string | null>(null);

  const reset = useCallback(() => {
    setError(null);
    setTxHash(null);
    setIsProcessing(false);
  }, []);

  const delegate = useCallback(async (
    walletName: string,
    poolTicker: string,
  ): Promise<string> => {
    if (!isBrowser) {
      throw new Error('Delegation only available in browser');
    }

    const poolId = POOL_IDS[poolTicker.toUpperCase()];
    if (!poolId) {
      throw new Error(`Unknown pool: ${poolTicker}`);
    }

    setIsProcessing(true);
    setError(null);
    setTxHash(null);

    try {
      // Dynamic import -- avoids SSR issues with MeshJS Node-only dependencies
      const { Transaction, KoiosProvider, BrowserWallet } = await import('@meshsdk/core');

      // Initialize Koios provider (mainnet)
      const provider = new KoiosProvider('api');

      // Connect to the wallet using MeshJS BrowserWallet
      // This wraps the CIP-30 enable() call with MeshJS convenience methods
      const wallet = await BrowserWallet.enable(walletName);

      // Get the reward address (stake address) from the wallet
      const rewardAddresses = await wallet.getRewardAddresses();
      if (!rewardAddresses || rewardAddresses.length === 0) {
        throw new Error('No reward address found in wallet');
      }
      const rewardAddress = rewardAddresses[0];

      // Build the delegation transaction
      // provider is used for UTxO fetching (fee calculation) and tx submission
      const tx = new Transaction({
        initiator: wallet,
        fetcher: provider,
        submitter: provider,
      });

      // Register stake address (2 ADA refundable deposit for first-time delegation)
      // TODO: Query registration status first to avoid StakeKeyAlreadyRegistered error
      tx.registerStake(rewardAddress);

      // Delegate to the selected pool
      tx.delegateStake(rewardAddress, poolId);

      // Build: selects UTxOs, calculates fees, balances tx, returns unsigned CBOR
      const unsignedTx = await tx.build();

      // Sign: prompts user via wallet extension popup
      const signedTx = await wallet.signTx(unsignedTx);

      // Submit: sends signed tx to the network via the wallet's connected node
      const txHashResult = await wallet.submitTx(signedTx);

      setTxHash(txHashResult);
      setIsProcessing(false);
      return txHashResult;
    } catch (err) {
      const message = err instanceof Error ? err.message : 'Delegation failed';
      setError(message);
      setIsProcessing(false);
      throw err;
    }
  }, []);

  return {
    isProcessing,
    error,
    txHash,
    delegate,
    reset,
  };
}
```

## Usage in a Component

```tsx
import { useDelegation } from '../hooks/useDelegation';
import { useWallet } from '../hooks/useWallet';

function DelegateButton({ poolTicker }: { poolTicker: string }) {
  const { walletName } = useWallet();
  const { isProcessing, error, txHash, delegate, reset } = useDelegation();

  const handleDelegate = async () => {
    if (!walletName) return;
    try {
      await delegate(walletName, poolTicker);
      // Optionally refresh delegation status after success
    } catch {
      // Error already captured in hook state
    }
  };

  if (txHash) {
    return (
      <div>
        <p>Delegation submitted!</p>
        <p>Tx: {txHash.slice(0, 16)}...</p>
        <button onClick={reset}>Done</button>
      </div>
    );
  }

  return (
    <div>
      <button onClick={handleDelegate} disabled={isProcessing || !walletName}>
        {isProcessing ? 'Delegating...' : `Delegate to ${poolTicker}`}
      </button>
      {error && <p className="text-red-500">{error}</p>}
    </div>
  );
}
```

## Common Mistakes

### 1. Top-Level MeshJS Import

```typescript
// WRONG: Breaks SSR build
import { Transaction, BrowserWallet } from '@meshsdk/core';

// RIGHT: Dynamic import at call site
const { Transaction, BrowserWallet } = await import('@meshsdk/core');
```

### 2. Using Hex Pool ID for Delegation

```typescript
// WRONG: delegateStake expects bech32
tx.delegateStake(rewardAddress, '3116c834a09b0060aef7284f63d3275456364e3309b3c19ec328af60');

// RIGHT: Convert to pool1... format
tx.delegateStake(rewardAddress, hexToBech32PoolId(hexPoolId));
```

### 3. Not Handling `StakeKeyAlreadyRegistered`

```typescript
// WRONG: Always register -- fails for already-registered wallets
tx.registerStake(rewardAddress);
tx.delegateStake(rewardAddress, poolId);

// BETTER: Check registration first (via API), or catch and retry without register
try {
  tx.registerStake(rewardAddress);
  tx.delegateStake(rewardAddress, poolId);
  const unsigned = await tx.build();
  // ...
} catch (err) {
  if (err.message?.includes('StakeKeyAlreadyRegistered')) {
    // Retry without registerStake
    const tx2 = new Transaction({ initiator: wallet, fetcher: provider, submitter: provider });
    tx2.delegateStake(rewardAddress, poolId);
    const unsigned = await tx2.build();
    // ...
  }
}
```

### 4. Confusing `signTx` + `submitTx` With `wallet.signTx` + `provider.submitTx`

Both wallet and provider can submit. The wallet submits via its connected node;
the provider submits via Koios. Either works. Using the wallet is more reliable
since it uses the same node the wallet is synced with.

## See Also

- [Wallet Connect](wallet-connect.md) -- getting the walletName for delegation
- [API Failover](api-failover.md) -- refreshing delegation status after submission
