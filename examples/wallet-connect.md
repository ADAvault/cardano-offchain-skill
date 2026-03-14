# Example: Wallet Connect and Disconnect

Connect to a CIP-30 wallet, detect installed extensions, verify network, fetch
stake address, decode CBOR balance, extract ADA Handle, and disconnect cleanly.

## Key Concepts

- **Extension detection**: Scan `window.cardano` for supported wallet keys
- **`enable()` handshake**: The CIP-30 authorization call that returns the API handle
- **Network gate**: Always verify `getNetworkId()` before other calls
- **CBOR balance**: `getBalance()` returns hex CBOR, not a number
- **ADA Handle**: Search UTxOs for the Handle policy ID, decode asset name
- **Error isolation**: Each API call can fail independently; handle each

## Source Code

```typescript
import { useState, useCallback, useEffect } from 'react';
import { bech32 } from 'bech32';
import { SUPPORTED_WALLETS, WALLET_STORAGE_KEY, getWalletInfo } from '../config/wallets';
import type { WalletInfo } from '../types/wallet';

// SSR guard
const isBrowser = typeof window !== 'undefined';

// ADA Handle policy ID (mainnet)
const ADA_HANDLE_POLICY_ID = 'f0ff48bbb7bbe9d59a40f1ce90e9e9d0ff5002ec48f232b49ca0fb9a';

// -- Helpers (see cbor-balance.md and ada-handle.md for full implementations) --
function decodeCborBalance(hex: string): number | null { /* ... */ }
function extractAdaHandle(utxosHex: string[]): string | null { /* ... */ }
function hexToBech32StakeAddress(hexAddr: string): string {
  const cleanHex = hexAddr.startsWith('0x') ? hexAddr.slice(2) : hexAddr;
  const bytes = Buffer.from(cleanHex, 'hex');
  const words = bech32.toWords(bytes);
  return bech32.encode('stake', words, 200);
}

export function useWallet() {
  const [isConnected, setIsConnected] = useState(false);
  const [isConnecting, setIsConnecting] = useState(false);
  const [enabledWallet, setEnabledWallet] = useState<string | null>(null);
  const [stakeAddress, setStakeAddress] = useState<string | null>(null);
  const [accountBalance, setAccountBalance] = useState<number | null>(null);
  const [adaHandle, setAdaHandle] = useState<string | null>(null);
  const [installedExtensions, setInstalledExtensions] = useState<string[]>([]);
  const [error, setError] = useState<string | null>(null);

  // Detect installed wallets on mount
  useEffect(() => {
    if (!isBrowser) return;

    const detectWallets = () => {
      const cardano = window.cardano;
      if (!cardano) {
        setInstalledExtensions([]);
        return;
      }
      const installed = SUPPORTED_WALLETS
        .filter((wallet) => cardano[wallet.name] !== undefined)
        .map((wallet) => wallet.name);
      setInstalledExtensions(installed);
    };

    detectWallets();
    // Re-scan after 500ms for late injection
    const timer = setTimeout(detectWallets, 500);
    return () => clearTimeout(timer);
  }, []);

  // Connect to wallet
  const connectWallet = useCallback(async (walletName: string, silent = false) => {
    if (!isBrowser) return;
    setIsConnecting(true);
    setError(null);

    try {
      const cardano = window.cardano;
      if (!cardano?.[walletName]) {
        throw new Error(`${walletName} wallet not found`);
      }

      // Step 1: CIP-30 handshake
      const api = await cardano[walletName].enable();

      // Step 2: Verify network
      const networkId = await api.getNetworkId();
      if (networkId !== 1) {
        throw new Error('Please switch to Cardano mainnet');
      }

      // Step 3: Get stake address
      const rewardAddresses = await api.getRewardAddresses();
      const stakeAddr = rewardAddresses?.[0] || null;

      // Step 4: Decode CBOR balance (independent -- don't let failure block connect)
      let balance: number | null = null;
      try {
        const balanceCbor = await api.getBalance();
        balance = decodeCborBalance(balanceCbor);
      } catch {
        // Balance fetch failed -- continue
      }

      // Step 5: Extract ADA Handle (independent)
      let handle: string | null = null;
      try {
        const utxos = await api.getUtxos();
        if (utxos && utxos.length > 0) {
          handle = extractAdaHandle(utxos);
        }
      } catch {
        // Handle fetch failed -- continue
      }

      // Update state
      setIsConnected(true);
      setEnabledWallet(walletName);
      setStakeAddress(stakeAddr);
      setAccountBalance(balance);
      setAdaHandle(handle);
      localStorage.setItem(WALLET_STORAGE_KEY, walletName);

    } catch (err) {
      const message = err instanceof Error ? err.message : 'Connection failed';
      if (!silent) setError(message);
      throw err;
    } finally {
      setIsConnecting(false);
    }
  }, []);

  // Disconnect wallet
  const disconnect = useCallback(() => {
    setIsConnected(false);
    setEnabledWallet(null);
    setStakeAddress(null);
    setAccountBalance(null);
    setAdaHandle(null);
    setError(null);
    localStorage.removeItem(WALLET_STORAGE_KEY);
  }, []);

  // Get available wallets (installed AND supported)
  const getAvailableWallets = useCallback((): WalletInfo[] => {
    return SUPPORTED_WALLETS.filter((wallet) =>
      installedExtensions.includes(wallet.name)
    );
  }, [installedExtensions]);

  // Format balance for display (lovelace to ADA)
  const formatBalance = useCallback((lovelace: number | null): string => {
    if (lovelace === null) return '---';
    return (lovelace / 1_000_000).toLocaleString(undefined, {
      minimumFractionDigits: 2,
      maximumFractionDigits: 2,
    });
  }, []);

  // Format address for display (truncate)
  const formatAddress = useCallback((addr: string | null): string => {
    if (!addr) return '';
    return `${addr.slice(0, 12)}...${addr.slice(-6)}`;
  }, []);

  const walletInfo = enabledWallet ? getWalletInfo(enabledWallet) : null;

  return {
    isConnected,
    isConnecting,
    walletName: enabledWallet,
    walletDisplayName: walletInfo?.displayName || enabledWallet,
    walletIcon: walletInfo?.icon,
    stakeAddress,
    adaHandle,
    balance: accountBalance,
    error,
    connect: connectWallet,
    disconnect,
    getAvailableWallets,
    clearError: () => setError(null),
    formatAddress,
    formatBalance,
  };
}
```

## Common Mistakes

### 1. Checking `isEnabled()` Before `enable()`

```typescript
// WRONG: isEnabled() is unreliable (see gotchas G-W1)
if (await window.cardano.eternl.isEnabled()) {
  const api = await window.cardano.eternl.enable();
}

// RIGHT: Call enable() directly
const api = await window.cardano.eternl.enable();
```

### 2. Not Handling Each CIP-30 Call Independently

```typescript
// WRONG: One failure aborts everything
const balance = decodeCborBalance(await api.getBalance());
const utxos = await api.getUtxos();
const handle = extractAdaHandle(utxos);

// RIGHT: Isolate failures
let balance: number | null = null;
try { balance = decodeCborBalance(await api.getBalance()); } catch {}

let handle: string | null = null;
try {
  const utxos = await api.getUtxos();
  if (utxos && utxos.length > 0) handle = extractAdaHandle(utxos);
} catch {}
```

### 3. Initializing State from localStorage in `useState()`

```typescript
// WRONG: Causes hydration mismatch
const [wallet, setWallet] = useState(localStorage.getItem('wallet'));

// RIGHT: Initialize as null, populate in useEffect
const [wallet, setWallet] = useState<string | null>(null);
```

### 4. Not Re-Scanning for Extensions

```typescript
// WRONG: Single scan misses late injection
const installed = detectWallets();

// RIGHT: Scan twice with delay
detectWallets();
setTimeout(detectWallets, 500);
```

## See Also

- [Auto-Reconnect](auto-reconnect.md) -- persistence and retry logic
- [CBOR Balance](cbor-balance.md) -- full decoder implementation
- [ADA Handle](ada-handle.md) -- full extraction implementation
