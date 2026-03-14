---
name: cardano-offchain
description: >
  Build Cardano dApp frontends with CIP-30 wallet integration, MeshJS transactions,
  and Playwright E2E testing. Use when working with browser wallets, delegation,
  UTxO queries, CBOR decoding, ADA Handle resolution, or dApp frontend architecture.
  Triggers on: CIP-30, wallet, BrowserWallet, MeshJS, Ogmios, Kupo, delegation,
  UTxO, ADA Handle, dApp frontend, ConnectWallet, stake address, lovelace, CBOR,
  transaction building, wallet mock, Playwright wallet, Cardano off-chain.
user-invocable: true
---

# Cardano Off-Chain & dApp Frontend Development

You are an expert Cardano dApp frontend developer. You build browser-based wallet
integrations using CIP-30, construct transactions with MeshJS, query chain state
via Ogmios/Kupo, and test everything with Playwright mock wallets.

## Core Principles

1. **SSR safety first** -- never access `window`, `localStorage`, or browser APIs at module load. Guard with `typeof window !== 'undefined'` and use `useEffect` for hydration.
2. **CIP-30 is the contract** -- all wallet interaction goes through the standardized CIP-30 API. Never assume wallet-specific behaviour.
3. **CBOR is the wire format** -- balances, UTxOs, and transaction bodies arrive as hex-encoded CBOR. Decode explicitly, handle edge cases.
4. **Fail gracefully with fallback** -- API calls should timeout (5s) and failover to a DR endpoint. Wallet connections should retry with backoff.
5. **Test with mock wallets** -- inject a CIP-30-compliant mock via Playwright's `addInitScript` before `page.goto()`. Never depend on real browser extensions in CI.
6. **Persist display, verify connection** -- cache wallet display data (balance, handle, delegation) in localStorage for instant UI, but always re-verify via `enable()` on page load.

## Decision Tree: Wallet Integration

When building or modifying wallet connectivity:

1. **Define supported wallets** -- create a typed config array with CIP-30 `name`, display name, and icon path
2. **Detect installed extensions** -- scan `window.cardano` for matching keys; re-scan after 500ms for late injection
3. **Connect via `enable()`** -- call `window.cardano[name].enable()` to get the API handle; this is the CIP-30 handshake
4. **Verify network** -- call `api.getNetworkId()`; reject if not the expected network (1 = mainnet, 0 = testnet)
5. **Fetch stake address** -- call `api.getRewardAddresses()`; convert hex to bech32 with `stake` prefix
6. **Decode balance** -- call `api.getBalance()`; decode the CBOR response (see CBOR section below)
7. **Extract ADA Handle** -- call `api.getUtxos()`; search for the ADA Handle policy ID in UTxO hex
8. **Fetch delegation status** -- call your API with the bech32 stake address; display pool ticker
9. **Persist wallet name** -- save to `localStorage`; broadcast state via `CustomEvent` for cross-component sync
10. **Auto-reconnect on reload** -- read saved wallet from storage; retry with backoff `[0, 500, 1000, 2000]ms`; skip `isEnabled()` (unreliable on Eternl); call `enable()` directly

## Decision Tree: Transaction Building

When constructing and submitting transactions:

1. **Dynamic import MeshJS** -- use `await import('@meshsdk/core')` to avoid SSR bundling issues
2. **Initialize provider** -- `new KoiosProvider('api')` for mainnet; or custom provider for testnet
3. **Connect wallet** -- `BrowserWallet.enable(walletName)` returns a MeshJS wallet handle
4. **Get reward address** -- `wallet.getRewardAddresses()` for delegation; `wallet.getUsedAddresses()` for payment
5. **Build transaction** -- `new Transaction({ initiator: wallet, fetcher: provider, submitter: provider })`
6. **Add operations** -- `tx.registerStake()`, `tx.delegateStake()`, `tx.sendLovelace()`, etc.
7. **Sign and submit** -- `tx.build()` then `wallet.signTx()` then `wallet.submitTx()`
8. **Handle errors** -- catch user rejection ("user declined"), insufficient funds, network errors separately
9. **Refresh state** -- after successful submission, re-fetch delegation status / balance

## Decision Tree: E2E Testing

When writing Playwright tests for wallet features:

1. **Create a mock wallet fixture** -- implement the CIP-30 interface returning configurable test data
2. **Inject before navigation** -- use `page.addInitScript()` to inject the mock before `page.goto()`
3. **Support late injection** -- add configurable `injectionDelay` via `setTimeout` to test retry logic
4. **Set localStorage** -- pre-set the wallet storage key via `addInitScript` to trigger auto-reconnect
5. **Test the happy path** -- connect, verify balance/address display, disconnect
6. **Test error paths** -- wrong network, enable rejection, missing extension
7. **Test persistence** -- connect, navigate (ViewTransitions), verify state survives
8. **Use strict selectors** -- desktop nav uses `.hidden.md\\:flex`; modal buttons use `page.locator('button').filter({ hasText: name }).last()`

## CIP-30 API Quick Reference

```typescript
// Extension detection
window.cardano                          // Global namespace
window.cardano[walletName]              // Specific wallet extension
window.cardano[walletName].enable()     // -> Promise<WalletApi>
window.cardano[walletName].isEnabled()  // -> Promise<boolean> (unreliable)

// Wallet API (returned by enable())
api.getNetworkId()          // -> Promise<number>       1=mainnet, 0=testnet
api.getBalance()            // -> Promise<string>       CBOR hex
api.getRewardAddresses()    // -> Promise<string[]>     hex stake addresses
api.getUsedAddresses()      // -> Promise<string[]>     hex payment addresses
api.getUnusedAddresses()    // -> Promise<string[]>     hex addresses
api.getChangeAddress()      // -> Promise<string>       hex address
api.getUtxos()              // -> Promise<string[]|undefined>  CBOR hex UTxOs
api.signTx(tx, partial?)    // -> Promise<string>       signed tx CBOR
api.signData(addr, payload) // -> Promise<{signature, key}>
api.submitTx(tx)            // -> Promise<string>       tx hash
```

## CBOR Balance Decoding

CIP-30 `getBalance()` returns CBOR-encoded values. Minimal decoder for lovelace:

```typescript
function decodeCborBalance(hex: string): number | null {
  try {
    const bytes = new Uint8Array(
      hex.match(/.{1,2}/g)!.map(b => parseInt(b, 16))
    );
    let offset = 0;

    const readUint = (): number => {
      const first = bytes[offset++];
      const majorType = first >> 5;
      const additional = first & 0x1f;

      // Major type 0 = unsigned int, type 4 = array (multi-asset, read first element)
      if (majorType === 4) return readUint();
      if (majorType !== 0) throw new Error('Unsupported CBOR type');

      if (additional <= 23) return additional;
      if (additional === 24) return bytes[offset++];
      if (additional === 25) {
        const v = (bytes[offset] << 8) | bytes[offset + 1];
        offset += 2; return v;
      }
      if (additional === 26) {
        const v = (bytes[offset] << 24) | (bytes[offset+1] << 16) |
                  (bytes[offset+2] << 8) | bytes[offset+3];
        offset += 4; return v >>> 0;
      }
      if (additional === 27) {
        let v = BigInt(0);
        for (let i = 0; i < 8; i++) v = (v << 8n) | BigInt(bytes[offset++]);
        return Number(v);
      }
      throw new Error('Invalid CBOR');
    };

    return readUint();
  } catch { return null; }
}
```

Key points:
- Pure lovelace returns a CBOR unsigned integer directly
- Multi-asset balance wraps in a CBOR array `[lovelace, {policy: {asset: qty}}]` -- the decoder reads the first element
- Always handle `null` return (malformed CBOR from broken wallets)

## ADA Handle Extraction

Search UTxO hex for the ADA Handle policy ID, then decode the asset name:

```typescript
const ADA_HANDLE_POLICY_ID = 'f0ff48bbb7bbe9d59a40f1ce90e9e9d0ff5002ec48f232b49ca0fb9a';
const CIP68_REF_PREFIX = '000de140'; // CIP-68 reference token label 222

function extractAdaHandle(utxosHex: string[]): string | null {
  for (const utxo of utxosHex) {
    const idx = utxo.indexOf(ADA_HANDLE_POLICY_ID);
    if (idx === -1) continue;

    const after = utxo.substring(idx + ADA_HANDLE_POLICY_ID.length);
    // Skip CBOR map marker if present (a1-b7)
    const first = parseInt(after.substring(0, 2), 16);
    let start = (first >= 0xa1 && first <= 0xb7) ? 2 : 0;

    // Read CBOR byte string length
    const lenByte = parseInt(after.substring(start, start + 2), 16);
    let nameHex: string | null = null;
    if (lenByte >= 0x40 && lenByte <= 0x57) {
      nameHex = after.substring(start + 2, start + 2 + (lenByte - 0x40) * 2);
    } else if (lenByte === 0x58) {
      const len = parseInt(after.substring(start + 2, start + 4), 16);
      nameHex = after.substring(start + 4, start + 4 + len * 2);
    }

    if (nameHex) {
      // Strip CIP-68 reference token prefix
      if (nameHex.startsWith(CIP68_REF_PREFIX)) {
        nameHex = nameHex.substring(CIP68_REF_PREFIX.length);
      }
      const name = nameHex.match(/.{1,2}/g)?.map(b =>
        String.fromCharCode(parseInt(b, 16))
      ).join('') || '';
      if (/^[a-z0-9_.-]+$/.test(name) && name.length >= 1) {
        return '$' + name;
      }
    }
  }
  return null;
}
```

## API Failover Pattern

Resilient fetch with timeout and automatic DR failover:

```typescript
const API_PRIMARY = 'https://api.example.com/api/v1';
const API_FALLBACK = 'https://api2.example.com/api/v1';

async function apiFetch(path: string, options?: RequestInit): Promise<Response> {
  const norm = path.startsWith('/') ? path : `/${path}`;
  try {
    const ctrl = new AbortController();
    const tid = setTimeout(() => ctrl.abort(), 5000);
    const res = await fetch(`${API_PRIMARY}${norm}`, {
      ...options, signal: ctrl.signal,
    });
    clearTimeout(tid);
    if (res.ok || res.status < 500) return res;
    throw new Error(`Primary error: ${res.status}`);
  } catch (err) {
    console.warn('Primary API failed, trying fallback:', err);
    const ctrl = new AbortController();
    const tid = setTimeout(() => ctrl.abort(), 5000);
    try {
      return await fetch(`${API_FALLBACK}${norm}`, { ...options, signal: ctrl.signal });
    } finally { clearTimeout(tid); }
  }
}
```

Pattern notes:
- 5-second timeout via `AbortController` on both primary and fallback
- 4xx responses returned to caller (client errors, not server availability issues)
- Only 5xx and network errors trigger failover
- Dev mode should skip fallback (DR unavailable locally)

## Bech32 Address Encoding

Convert hex addresses to human-readable bech32:

```typescript
import { bech32 } from 'bech32';

function hexToBech32(hex: string, prefix: string): string {
  const bytes = Buffer.from(hex, 'hex');
  const words = bech32.toWords(bytes);
  return bech32.encode(prefix, words, 200);
}

// Usage
hexToBech32(stakeAddrHex, 'stake');  // -> stake1u...
hexToBech32(poolIdHex, 'pool');       // -> pool1...
```

## Key Wallet Quirks

Hard-won learnings from production wallet integration:

- **Eternl first-load quirk** -- first browser session after install may need Eternl open in a separate tab before it injects. Subsequent cold starts work fine. Their side.
- **`isEnabled()` is unreliable** -- Eternl returns false after page reload for previously-authorized sites. Skip `isEnabled()` entirely; call `enable()` directly for auto-reconnect.
- **Late injection** -- wallet extensions may not be in `window.cardano` when your JS runs. Retry with backoff: `[0, 500, 1000, 2000]ms`. Clear stale storage if all retries fail.
- **SSR + localStorage** -- never initialize React state from `localStorage` at module load. Causes hydration mismatch. Use `useEffect` after mount.
- **ViewTransitions break React state** -- use `transition:persist` on wallet components (Astro) or the wallet state resets on SPA navigation.
- **Cross-component sync** -- use `CustomEvent` dispatch + `window.addEventListener` for wallet state sharing across React islands that don't share a provider tree.
- **Module-level singleton** -- shared wallet state at module scope avoids duplicate connections when multiple components mount. Initialize as disconnected (matches SSR).
- **Network check before balance** -- always verify `getNetworkId()` before any other API call. Users on testnet should see a clear error, not silent failures.
- **Dynamic MeshJS import** -- `await import('@meshsdk/core')` avoids bundling MeshJS (which has Node-only deps) into SSR output.
- **Koios CORS for unregistered accounts** -- browser-to-Koios requests fail without CORS headers unless you have a registered API account. Proxy through your own API to avoid this. See [koios-artifacts #397](https://github.com/cardano-community/koios-artifacts/issues/397).
- **MeshJS transitive vulnerabilities** -- `npm audit` flags undici/bn.js/devalue via MeshJS. Use `--audit-level=critical` and document the accepted risk.

## Reference Material

- [CIP-30 Wallet Standard](reference/cip30.md) -- full API surface, extension detection, TypeScript interfaces
- [Wallet Integration Patterns](reference/wallet-integration.md) -- React hooks, SSR safety, state management, auto-reconnect
- [MeshJS Transactions](reference/transactions.md) -- BrowserWallet, KoiosProvider, delegation, build/sign/submit
- [API Resilience](reference/api-resilience.md) -- failover, timeout, CORS proxy, data pipeline
- [CBOR Decoding](reference/cbor.md) -- balance, UTxO parsing, multi-asset values, ADA Handle
- [Infrastructure](reference/infrastructure.md) -- Ogmios, Kupo, Docker, HAProxy, SSH tunnels
- [E2E Testing](reference/testing.md) -- Playwright mock wallet, CIP-30 fixture, ViewTransitions
- [Gotchas](reference/gotchas.md) -- wallet quirks, CORS, MeshJS pitfalls, infrastructure issues

## Examples

- [Wallet Connect](examples/wallet-connect.md) -- full `useWallet()` React hook
- [Auto-Reconnect](examples/auto-reconnect.md) -- retry with backoff, localStorage persistence
- [Delegation](examples/delegation.md) -- MeshJS Transaction + BrowserWallet + KoiosProvider
- [API Failover](examples/api-failover.md) -- `apiFetch()` with timeout and DR endpoint
- [Mock Wallet](examples/mock-wallet.md) -- Playwright CIP-30 fixture with configurable behaviour
- [Wallet Tests](examples/wallet-tests.md) -- E2E test suite (connect, disconnect, persistence)
- [ADA Handle](examples/ada-handle.md) -- UTxO scanning with CIP-68 support
- [CBOR Balance](examples/cbor-balance.md) -- minimal decoder for CIP-30 getBalance()
