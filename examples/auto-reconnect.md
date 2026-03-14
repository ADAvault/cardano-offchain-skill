# Example: Auto-Reconnect with Retry Backoff

Automatically reconnect to a previously-connected wallet on page load, handling
late extension injection with retry backoff, and maintaining instant UI via
localStorage cache and cross-component broadcasting.

## Key Concepts

- **localStorage persistence**: Two keys -- wallet name for reconnect, full state for display
- **Retry backoff `[0, 500, 1000, 2000]ms`**: Handles extensions that inject late
- **Skip `isEnabled()`**: Eternl returns false after reload for authorized sites
- **Stale cleanup**: Clear storage if all retries fail (extension uninstalled)
- **Module-level singleton**: Shared state across React islands without context
- **CustomEvent broadcast**: Cross-component sync for disconnected React trees
- **Cache for instant UI**: Display cached balance/handle while reconnect verifies

## Source Code

```typescript
import { useState, useCallback, useEffect } from 'react';

const isBrowser = typeof window !== 'undefined';

// Storage keys
const WALLET_STORAGE_KEY = 'adavault_connected_wallet';       // Wallet name for reconnect
const WALLET_STATE_CACHE_KEY = 'adavault:wallet-state-cache'; // Full display state
const WALLET_STATE_EVENT = 'adavault:wallet-state-change';    // Cross-component event

// Module-level singleton -- shared across all useWallet() instances
// Initialize as disconnected to match SSR (prevents hydration mismatch)
let sharedWalletState = {
  isConnected: false,
  walletName: null as string | null,
  stakeAddress: null as string | null,
  balance: null as number | null,
  adaHandle: null as string | null,
  delegatedPool: null as { ticker: string; name: string; poolId: string } | null,
};

// Track whether cache has been restored (once per page lifecycle)
let cacheRestored = false;

// Load cached display state from localStorage
function loadCachedState() {
  if (!isBrowser) return null;
  try {
    const cached = localStorage.getItem(WALLET_STATE_CACHE_KEY);
    return cached ? JSON.parse(cached) : null;
  } catch {
    return null;
  }
}

// Persist state and notify all components
function broadcastWalletState() {
  if (!isBrowser) return;
  try {
    localStorage.setItem(WALLET_STATE_CACHE_KEY, JSON.stringify(sharedWalletState));
  } catch {}
  window.dispatchEvent(new CustomEvent(WALLET_STATE_EVENT, {
    detail: sharedWalletState,
  }));
}

export function useWallet() {
  const [isConnected, setIsConnected] = useState(sharedWalletState.isConnected);
  const [enabledWallet, setEnabledWallet] = useState<string | null>(sharedWalletState.walletName);
  const [stakeAddress, setStakeAddress] = useState<string | null>(sharedWalletState.stakeAddress);
  const [accountBalance, setAccountBalance] = useState<number | null>(sharedWalletState.balance);
  const [adaHandle, setAdaHandle] = useState<string | null>(sharedWalletState.adaHandle);
  const [delegatedPool, setDelegatedPool] = useState(sharedWalletState.delegatedPool);

  // 1. Restore cached display data after hydration (runs once)
  useEffect(() => {
    if (!isBrowser || cacheRestored) return;
    cacheRestored = true;

    const cached = loadCachedState();
    if (cached?.walletName) {
      // Restore display data for immediate UI feedback
      // Note: isConnected is NOT restored -- auto-reconnect must verify
      sharedWalletState.walletName = cached.walletName;
      sharedWalletState.stakeAddress = cached.stakeAddress;
      sharedWalletState.balance = cached.balance;
      sharedWalletState.adaHandle = cached.adaHandle;
      sharedWalletState.delegatedPool = cached.delegatedPool;

      setEnabledWallet(cached.walletName);
      setStakeAddress(cached.stakeAddress);
      setAccountBalance(cached.balance);
      setAdaHandle(cached.adaHandle);
      setDelegatedPool(cached.delegatedPool);
    }
  }, []);

  // 2. Listen for wallet state changes from other components
  useEffect(() => {
    if (!isBrowser) return;

    const handleStateChange = (e: CustomEvent) => {
      const state = e.detail;
      setIsConnected(state.isConnected);
      setEnabledWallet(state.walletName);
      setStakeAddress(state.stakeAddress);
      setAccountBalance(state.balance);
      setAdaHandle(state.adaHandle);
      setDelegatedPool(state.delegatedPool);
    };

    window.addEventListener(WALLET_STATE_EVENT, handleStateChange as EventListener);
    return () => window.removeEventListener(WALLET_STATE_EVENT, handleStateChange as EventListener);
  }, []);

  // 3. Auto-reconnect with retry backoff
  // Skips isEnabled() -- some wallets (Eternl) return false after reload for
  // previously-authorized sites. enable() is the authoritative CIP-30 handshake.
  useEffect(() => {
    if (!isBrowser) return;

    const savedWallet = localStorage.getItem(WALLET_STORAGE_KEY);
    if (!savedWallet) return;

    const RETRY_DELAYS = [0, 500, 1000, 2000];
    let attempt = 0;
    let cancelled = false;

    const tryReconnect = () => {
      if (cancelled) return;

      const cardano = window.cardano;
      if (cardano?.[savedWallet]) {
        // Extension is available -- connect
        connectWallet(savedWallet, true).catch(() => {
          // Silent fail -- wallet may have revoked access
        });
        return;
      }

      // Extension not injected yet -- retry with backoff
      attempt++;
      if (attempt < RETRY_DELAYS.length) {
        setTimeout(tryReconnect, RETRY_DELAYS[attempt]);
      } else {
        // All retries exhausted -- extension not available
        // Clear stale storage to prevent infinite reconnect attempts
        localStorage.removeItem(WALLET_STORAGE_KEY);
        localStorage.removeItem(WALLET_STATE_CACHE_KEY);
      }
    };

    tryReconnect();

    // Cleanup: cancel retries if component unmounts
    return () => { cancelled = true; };
  }, []);

  // Connect implementation (abbreviated -- see wallet-connect.md for full version)
  const connectWallet = useCallback(async (walletName: string, silent = false) => {
    // ... enable(), getNetworkId(), getBalance(), etc. ...

    // After successful connection, update singleton and broadcast
    sharedWalletState = {
      isConnected: true,
      walletName,
      stakeAddress: stakeAddr,
      balance,
      adaHandle: handle,
      delegatedPool: delegation,
    };
    broadcastWalletState();
  }, []);

  // Disconnect -- clear everything
  const disconnect = useCallback(() => {
    localStorage.removeItem(WALLET_STORAGE_KEY);
    localStorage.removeItem(WALLET_STATE_CACHE_KEY);

    sharedWalletState = {
      isConnected: false,
      walletName: null,
      stakeAddress: null,
      balance: null,
      adaHandle: null,
      delegatedPool: null,
    };
    broadcastWalletState();
  }, []);

  return { isConnected, enabledWallet, stakeAddress, accountBalance, adaHandle,
           delegatedPool, connectWallet, disconnect };
}
```

## Timing Diagram

```
Page Load
  |
  t=0ms   tryReconnect() -- check window.cardano.eternl
  |        -> undefined (extension not injected yet)
  |
  t=100ms  Extension content script injects window.cardano.eternl
  |
  t=500ms  Retry #1 -- check window.cardano.eternl
  |        -> found! Call connectWallet('eternl', true)
  |        -> enable() returns silently (previously authorized)
  |        -> getNetworkId(), getBalance(), etc.
  |        -> Update state, broadcast
  |
  UI shows connected state
```

## Common Mistakes

### 1. Gating on `isEnabled()`

```typescript
// WRONG: Eternl bug -- isEnabled returns false after reload
const isEnabled = await window.cardano.eternl.isEnabled();
if (isEnabled) {
  // This branch never executes after reload on Eternl
}

// RIGHT: Skip isEnabled, call enable() directly
// enable() returns silently for previously-authorized dApps
await window.cardano.eternl.enable();
```

### 2. Not Cleaning Up Stale Storage

```typescript
// WRONG: Retry forever, or leave stale storage
const savedWallet = localStorage.getItem(WALLET_STORAGE_KEY);
if (savedWallet && !window.cardano?.[savedWallet]) {
  // Never cleaned up -- next page load tries again
}

// RIGHT: Clear after exhausting retries
if (attempt >= RETRY_DELAYS.length) {
  localStorage.removeItem(WALLET_STORAGE_KEY);
  localStorage.removeItem(WALLET_STATE_CACHE_KEY);
}
```

### 3. Restoring `isConnected` from Cache

```typescript
// WRONG: User sees "connected" before enable() verifies
setIsConnected(cached.isConnected);  // true from cache

// RIGHT: Only set isConnected after successful enable()
// Restore display data (name, balance) but NOT connection status
setEnabledWallet(cached.walletName);
setAccountBalance(cached.balance);
// isConnected stays false until connectWallet succeeds
```

### 4. Missing Cancellation on Unmount

```typescript
// WRONG: Retry fires after component unmounts
setTimeout(tryReconnect, RETRY_DELAYS[attempt]);

// RIGHT: Track cancellation
let cancelled = false;
const tryReconnect = () => {
  if (cancelled) return;
  // ...
};
return () => { cancelled = true; };
```

## See Also

- [Wallet Connect](wallet-connect.md) -- full connect/disconnect implementation
- [Wallet Tests](wallet-tests.md) -- E2E tests for auto-reconnect
