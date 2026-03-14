# React Wallet Integration Patterns

This document covers the architecture of a production CIP-30 wallet integration
using React hooks in an SSR framework (Astro with ViewTransitions). The patterns
solve real problems: hydration mismatches, cross-component state sync, wallet
extension timing races, and SPA navigation persistence.

## Architecture Overview

The wallet integration uses four key patterns:

1. **Module-level singleton** -- shared mutable state outside React's tree
2. **CustomEvent broadcasting** -- cross-component sync without a shared provider
3. **localStorage persistence** -- instant display data on page navigation
4. **Retry-based auto-reconnect** -- handles late wallet extension injection

## Module-Level Singleton

React components in frameworks like Astro often exist as isolated "islands" -- they
don't share a React context or provider tree. A module-level singleton provides
shared wallet state across all components that import the same module:

```typescript
const isBrowser = typeof window !== 'undefined';

// Initialize as disconnected to match SSR output (avoids hydration mismatch)
let sharedWalletState = {
  isConnected: false,
  walletName: null as string | null,
  stakeAddress: null as string | null,
  balance: null as number | null,
  adaHandle: null as string | null,
  delegatedPool: null as { ticker: string; name: string; poolId: string } | null,
};
```

Critical: initialize as disconnected. The server renders with no wallet state.
If you initialize from localStorage at module load, the server and client HTML
will diverge, causing a React hydration mismatch.

## SSR Guard

Every browser API access must be guarded:

```typescript
const isBrowser = typeof window !== 'undefined';

// WRONG: Module-level localStorage access
const saved = localStorage.getItem('wallet');  // Crashes on server

// RIGHT: Guarded function
function loadCachedState() {
  if (!isBrowser) return null;
  try {
    const cached = localStorage.getItem(WALLET_STATE_CACHE_KEY);
    return cached ? JSON.parse(cached) : null;
  } catch {
    return null;
  }
}
```

Never initialize React state from `localStorage` in a `useState()` initializer.
The SSR pass executes the initializer and produces HTML with that state -- the
client then hydrates with a different value from `localStorage`, causing a mismatch.
Use `useEffect` after mount instead.

## CustomEvent Broadcasting

When one component (e.g., the header wallet button) connects a wallet, other
components (e.g., pool page delegation panel) need to update. Without a shared
React context, use the browser's native `CustomEvent`:

```typescript
const WALLET_STATE_EVENT = 'adavault:wallet-state-change';
const WALLET_STATE_CACHE_KEY = 'adavault:wallet-state-cache';

function broadcastWalletState() {
  if (isBrowser) {
    // Persist for instant display on next page navigation
    try {
      localStorage.setItem(WALLET_STATE_CACHE_KEY, JSON.stringify(sharedWalletState));
    } catch {
      // Ignore storage errors
    }
    window.dispatchEvent(new CustomEvent(WALLET_STATE_EVENT, {
      detail: sharedWalletState,
    }));
  }
}
```

Every component listens:

```typescript
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
```

## localStorage Persistence and Cache

Two separate localStorage keys serve different purposes:

1. **`adavault_connected_wallet`** -- stores the wallet name (`"eternl"`) for auto-reconnect.
2. **`adavault:wallet-state-cache`** -- stores full display state (balance, handle, delegation, stake address) for instant UI.

On page navigation with ViewTransitions, the cache provides immediate display data
while auto-reconnect re-verifies the actual CIP-30 connection in the background:

```typescript
let cacheRestored = false;

useEffect(() => {
  if (!isBrowser || cacheRestored) return;
  cacheRestored = true;

  const cached = loadCachedState();
  if (cached?.walletName) {
    // Restore display data for immediate UI feedback
    // Auto-reconnect will verify actual connection and update isConnected
    sharedWalletState.walletName = cached.walletName;
    sharedWalletState.stakeAddress = cached.stakeAddress;
    sharedWalletState.balance = cached.balance;
    sharedWalletState.adaHandle = cached.adaHandle;
    sharedWalletState.delegatedPool = cached.delegatedPool;
    // Update local React state (but NOT isConnected)
    setEnabledWallet(cached.walletName);
    setStakeAddress(cached.stakeAddress);
    setAccountBalance(cached.balance);
    setAdaHandle(cached.adaHandle);
    setDelegatedPool(cached.delegatedPool);
  }
}, []);
```

Note: `isConnected` is NOT restored from cache. Only auto-reconnect sets it to
`true` after a successful `enable()` call. This prevents showing "connected" when
the wallet extension is no longer available.

## Auto-Reconnect with Retry Backoff

Wallet extensions inject asynchronously. On page reload, the saved wallet name is
in localStorage but `window.cardano.eternl` may not exist yet. The retry logic
handles this timing gap:

```typescript
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
      connectWallet(savedWallet, true).catch(() => {
        // Silent fail -- wallet may have revoked access
      });
      return;
    }

    // Wallet extension not injected yet -- retry with backoff
    attempt++;
    if (attempt < RETRY_DELAYS.length) {
      setTimeout(tryReconnect, RETRY_DELAYS[attempt]);
    } else {
      // Retries exhausted -- clear stale storage
      localStorage.removeItem(WALLET_STORAGE_KEY);
      localStorage.removeItem(WALLET_STATE_CACHE_KEY);
    }
  };

  tryReconnect();
  return () => { cancelled = true; };
}, []);
```

Key design decisions:

- **Delays `[0, 500, 1000, 2000]ms`** -- most extensions inject within 200ms; the
  2-second final attempt catches slow machines and cold browser starts.
- **`isEnabled()` is skipped entirely** -- Eternl returns `false` after reload for
  previously-authorized sites. The `enable()` call inside `connectWallet()` is the
  authoritative CIP-30 handshake and returns silently for authorized dApps.
- **Stale cleanup** -- if all retries fail (extension uninstalled, disabled), clear
  both localStorage keys to prevent infinite reconnect attempts on future loads.
- **Cancellation** -- the `useEffect` cleanup sets `cancelled = true` to prevent
  reconnect attempts after the component unmounts.

## ViewTransitions Persistence

Astro's ViewTransitions provide SPA-like navigation that replaces the DOM. React
components lose their state unless marked with `transition:persist`:

```html
<!-- In Astro layout -->
<div transition:persist="wallet-button">
  <ConnectWalletButton client:load />
</div>
```

Even with `transition:persist`, the module-level singleton and localStorage cache
provide redundancy. If the framework swaps the component, the cache restores
display data instantly while auto-reconnect re-establishes the CIP-30 connection.

For blog layouts that use `astro:page-load` instead of `DOMContentLoaded`, ensure
wallet scripts listen to the correct lifecycle event:

```html
<script>
  document.addEventListener('astro:page-load', () => {
    // ViewTransitions-safe initialization
  });
</script>
```

## Hydration-Safe State Initialization

React state initialization in `useState()` runs during both SSR and client hydration.
The values must match:

```typescript
// CORRECT: Initialize from shared state (disconnected on SSR, matches client)
const [isConnected, setIsConnected] = useState(sharedWalletState.isConnected);
const [enabledWallet, setEnabledWallet] = useState<string | null>(sharedWalletState.walletName);

// WRONG: Initialize from localStorage (different on server vs client)
const [isConnected, setIsConnected] = useState(
  typeof window !== 'undefined' && localStorage.getItem('wallet') !== null
);
```

The `sharedWalletState` singleton initializes as disconnected (matching SSR), then
useEffect populates from cache after hydration completes.

## Disconnect Flow

On disconnect, clear all state and both localStorage keys:

```typescript
const disconnect = useCallback(() => {
  setIsConnected(false);
  setEnabledWallet(null);
  setStakeAddress(null);
  setAccountBalance(null);
  setAdaHandle(null);
  setDelegatedPool(null);
  setError(null);
  localStorage.removeItem(WALLET_STORAGE_KEY);
  localStorage.removeItem(WALLET_STATE_CACHE_KEY);

  // Update shared state and broadcast to other components
  sharedWalletState = {
    isConnected: false, walletName: null, stakeAddress: null,
    balance: null, adaHandle: null, delegatedPool: null,
  };
  broadcastWalletState();
}, []);
```

## See Also

- [CIP-30 Standard](cip30.md) -- full API surface and TypeScript interfaces
- [E2E Testing](testing.md) -- testing auto-reconnect, persistence, ViewTransitions
- [Gotchas](gotchas.md) -- SSR, hydration, wallet quirks
