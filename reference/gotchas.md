# Gotchas: Hard-Won Learnings

Organized by category. Each entry documents a real issue encountered in production
or development, with the root cause and fix.

---

## CIP-30 Wallet

### G-W1: Eternl `isEnabled()` Returns False After Reload

**Symptom**: Auto-reconnect fails because `isEnabled()` returns `false` for a
previously-authorized dApp after page reload.

**Root cause**: Eternl's `isEnabled()` implementation does not persist authorization
state correctly across page loads. This is a wallet-side bug.

**Fix**: Skip `isEnabled()` entirely. Call `enable()` directly for auto-reconnect.
For previously-authorized dApps, `enable()` returns silently without showing a
popup.

```typescript
// WRONG: Gate on isEnabled
if (await window.cardano.eternl.isEnabled()) {
  api = await window.cardano.eternl.enable();
}

// RIGHT: Call enable() directly
api = await window.cardano.eternl.enable();
```

### G-W2: Eternl First-Load Quirk

**Symptom**: First browser session after installing Eternl, the extension does not
inject into `window.cardano` on other tabs.

**Root cause**: Eternl's content script injection requires the extension's own tab
to be open at least once after install.

**Fix**: Open Eternl in its own tab first. Subsequent cold starts work without this.
Document for users. This is their side, not yours.

### G-W3: Extension Injection Timing (0-2000ms)

**Symptom**: `window.cardano.eternl` is `undefined` when your JavaScript runs.

**Root cause**: Browser extension content scripts inject asynchronously. On cold
browser starts or slow machines, injection can take up to 2 seconds.

**Fix**: Retry with backoff: `[0, 500, 1000, 2000]ms`. Clear stale localStorage
if all retries fail (extension was uninstalled).

### G-W4: `getBalance()` Returns CBOR, Not a Number

**Symptom**: Code treats balance as a number and gets `NaN` or nonsensical values.

**Root cause**: CIP-30 `getBalance()` returns a hex-encoded CBOR string. Pure ADA
is a CBOR unsigned integer; multi-asset is a CBOR array.

**Fix**: Decode the CBOR. See `reference/cbor.md` for the minimal decoder.

### G-W5: `getUtxos()` Can Return `undefined`

**Symptom**: `utxos.length` throws `Cannot read property 'length' of undefined`.

**Root cause**: Some wallets return `undefined` instead of an empty array when
UTxO pagination is unsupported or the wallet isn't synced.

**Fix**: Always guard: `if (utxos && utxos.length > 0)`.

### G-W6: User Rejection Error Messages Vary

**Symptom**: Error handling catches "user declined" but misses Lace's "User cancelled"
or Yoroi's different message.

**Root cause**: CIP-30 does not standardize the rejection error message.

**Fix**: Catch all errors from `enable()` and display a generic "Connection cancelled"
message. Don't match on specific strings for user rejections.

---

## SSR / Hydration

### G-S1: `localStorage` at Module Load

**Symptom**: React hydration mismatch warning. Server renders one state, client
renders another.

**Root cause**: `localStorage.getItem()` in a `useState()` initializer runs during
SSR (returns `undefined`) but returns a value on the client.

**Fix**: Initialize state as a fixed default (e.g., disconnected). Use `useEffect`
to read localStorage after mount:

```typescript
// WRONG
const [wallet, setWallet] = useState(localStorage.getItem('wallet'));

// RIGHT
const [wallet, setWallet] = useState<string | null>(null);
useEffect(() => {
  setWallet(localStorage.getItem('wallet'));
}, []);
```

### G-S2: `typeof window` Guard Required

**Symptom**: `ReferenceError: window is not defined` during build.

**Root cause**: Astro/Next.js pre-renders pages on the server where `window` does
not exist.

**Fix**: Guard all browser API access:

```typescript
const isBrowser = typeof window !== 'undefined';
```

### G-S3: Cloudflare Rocket Loader Breaks React Hydration

**Symptom**: React components fail to hydrate. Console shows hydration mismatch
errors. Interactive elements don't work.

**Root cause**: Cloudflare's Rocket Loader modifies script loading order, breaking
React's expectation of synchronous hydration.

**Fix**: Disable Rocket Loader in Cloudflare dashboard (Speed > Optimization).

### G-S4: ViewTransitions Require `astro:page-load`

**Symptom**: Scripts in blog layout stop working after ViewTransitions navigation.

**Root cause**: `DOMContentLoaded` only fires on full page load, not ViewTransitions
soft navigations.

**Fix**: Use `astro:page-load` event instead:

```html
<script>
  document.addEventListener('astro:page-load', () => {
    // Runs on both full load and ViewTransitions navigation
  });
</script>
```

---

## MeshJS (Browser)

### G-M1: Dynamic Import Required for SSR

**Symptom**: Build fails with Node-only module errors, or SSR output includes
broken MeshJS code.

**Root cause**: MeshJS depends on Node-built-in modules (`crypto`, `buffer`, etc.)
that aren't available during SSR.

**Fix**: Always use dynamic import at the call site:

```typescript
const { Transaction, BrowserWallet } = await import('@meshsdk/core');
```

### G-M2: Transitive Vulnerabilities -- `audit-level=critical`

**Symptom**: `npm audit` fails CI with high-severity warnings from undici, bn.js,
devalue.

**Root cause**: MeshJS pulls in transitive dependencies with known high-severity
advisories that cannot be resolved by the consuming project.

**Fix**: Use `npm audit --audit-level=critical` in CI. Document the accepted risk.
These are transitive dependencies you cannot control.

### G-M3: CSP `unsafe-eval` Required for Polyfills

**Symptom**: Wallet operations fail with CSP violations in the browser console.

**Root cause**: MeshJS and its crypto polyfills use `eval()` or `Function()` for
WebAssembly and buffer polyfills.

**Fix**: Add `unsafe-eval` to `script-src` in your CSP header. Flag for future
removal when MeshJS updates its polyfill strategy.

```
Content-Security-Policy: script-src 'self' 'unsafe-eval';
```

---

## API / Network

### G-A1: Koios CORS (Unregistered Accounts)

**Symptom**: Browser fetch to `api.koios.rest` fails with CORS error.

**Root cause**: Koios public API does not send CORS headers for unregistered accounts.
Registering for a Koios API account resolves this, but proxying is more resilient.
See [koios-artifacts #397](https://github.com/cardano-community/koios-artifacts/issues/397).

**Fix**: Either register for a Koios API account, or proxy requests through your
own API server which adds proper CORS headers.

### G-A2: Express Owns CORS, Not nginx

**Symptom**: Browser rejects API responses despite CORS being configured.

**Root cause**: Both nginx and Express add CORS headers. Duplicate
`Access-Control-Allow-Origin` headers cause browsers to reject the response.

**Fix**: Remove CORS headers from nginx. Let Express handle all CORS logic. nginx
should pass through without modification.

### G-A3: `AbortController` 5-Second Timeout

**Symptom**: Slow API responses never trigger failover.

**Root cause**: Missing timeout on the primary fetch. Without `AbortController`,
the browser waits 30-60 seconds before timing out.

**Fix**: Create an `AbortController`, set a 5-second `setTimeout` to abort it,
pass the signal to `fetch()`. Clear the timeout on successful response.

### G-A4: 4xx Should Not Trigger Failover

**Symptom**: "Not Found" errors on primary trigger unnecessary failover to DR,
which returns the same 404.

**Root cause**: Failover logic treats all non-ok responses as server failures.

**Fix**: Only failover on 5xx and network errors. Return 4xx responses directly:

```typescript
if (res.ok || res.status < 500) return res;
throw new Error(`Server error: ${res.status}`);
```

---

## Infrastructure

### G-I1: PM2 Requires nvm on All Servers

**Symptom**: `pm2: command not found` when SSHing to API servers.

**Root cause**: Node.js is installed via nvm, which requires sourcing before use.

**Fix**: Always source nvm first:

```bash
export NVM_DIR="$HOME/.nvm" && [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
pm2 list
```

### G-I2: Mixed OS in Infrastructure

**Symptom**: Commands fail with "command not found" on some servers.

**Root cause**: If your infrastructure includes FreeBSD alongside Linux, commands differ. Different package manager, service manager, and utility commands.

**Fix**: Use OS-appropriate equivalents:
- `fetch` instead of `curl` (FreeBSD)
- `service nginx reload` instead of `systemctl reload nginx`
- `pfctl` instead of `ufw`
- Check SSH user per host

### G-I3: Never Edit Production Directly

**Symptom**: Production outage after direct file edit on API server.

**Root cause**: Manual edits bypass version control, can't be rolled back, and
create drift between servers.

**Fix**: All changes via git. Deploy with `git pull` on each server.

### G-I4: 523 Error = Upstream Network Issue

**Symptom**: Cloudflare returns 523 but nginx shows no errors and no access log
entries for the period.

**Root cause**: Traffic never reached the origin server. The issue is between
Cloudflare and your infrastructure (ISP, firewall, DNS).

**Fix**: Check DNS, ISP status, and Cloudflare origin pulls. nginx access log
gaps (no requests) confirm traffic didn't arrive.

---

## Kupo

### G-K1: ConflictingOptionsException on Restart

**Symptom**: Kupo container won't start after dynamic pattern additions via PUT API.

**Root cause**: DB patterns diverge from Docker `--match` flags. Kupo validates
on startup and refuses to run with mismatched configuration.

**Fix**: Stop container, remove DB volume, recreate with ALL patterns in `--match`
flags. Fresh sync needed.

### G-K2: PUT Rollback Must Be Within 36 Hours

**Symptom**: PUT request to add pattern fails or hangs.

**Root cause**: Kupo's safe rollback zone is 36 hours. Older rollback points require
explicit override.

**Fix**: Use `"limit": "unsafe_allow_beyond_safe_zone"` in the PUT request body.
Or use a recent checkpoint from the checkpoint API.

### G-K3: PUT Can Stall Block Sync

**Symptom**: Kupo's `most_recent_checkpoint` stops advancing. Queries return stale
UTxOs. Transactions fail with error 3117 ("unknown UTxO references").

**Root cause**: A long-running PUT re-index monopolizes Kupo's processing, preventing
it from consuming new blocks from Ogmios.

**Fix**: Kill the stale `curl` process. Restart the Kupo container. Monitor
`most_recent_checkpoint` to confirm sync resumes.

### G-K4: Targeted Indexing Saves Massive Disk

**Symptom**: Kupo uses 50+ GB of disk and takes hours to sync.

**Root cause**: Running with `--match "*"` indexes everything.

**Fix**: Use targeted `--match` patterns with specific credentials. A single-dApp
deployment uses ~50 MB instead of ~50 GB.

### G-K5: MeshTxBuilder Fee Calculation Underestimates by ~2K Lovelace

**Symptom**: Transaction builds successfully but Ogmios rejects with error 3122
"Insufficient fee" on submission. The gap is typically 1,000-3,000 lovelace.

**Root cause**: This is a **known Cardano ecosystem issue**, not MeshJS-specific.
Fee calculation has a circular dependency: fee depends on tx size, tx size depends
on fee (fee is embedded in the tx body), and change output depends on fee.
MeshTxBuilder uses iterative resolution but doesn't perfectly converge. The same
issue is documented in `cardano-serialization-lib` (#496, #441) and `cardano-node`
(#1529).

**Fix — Pattern A (simplest, production-proven)**: Set an explicit fee via `complete()`:

```typescript
const txBuilder = new MeshTxBuilder({
  fetcher: kupo,
  submitter: ogmios,
  // No evaluator — manual exUnits
});

await txBuilder
  .mintPlutusScriptV3()
  .mint('1', policyId, tokenName)
  .mintingScript(scriptCbor)
  .mintRedeemerValue(redeemer, 'JSON', { mem: 2_000_000, steps: 1_000_000_000 })
  // ... rest of tx ...
  .complete({ fee: '500000' });  // Explicit 0.5 ADA — excess goes to change

txBuilder.completeSigning();
const txHash = await ogmios.submitTx(txBuilder.txHex);
```

The explicit fee bypasses `calculateFee()` entirely. The Cardano protocol only
charges the actual execution cost, not the budgeted amount. The excess fee above
the minimum is returned as change output.

**Fix — Pattern B (defensive retry)**: Build normally, catch 3122, bump fee:

```typescript
try {
  const txHash = await ogmios.submitTx(txBuilder.txHex);
  return txHash;
} catch (err) {
  if (err.message.includes('3122')) {
    const currentFee = BigInt(txBuilder.meshTxBuilderBody.fee);
    await txBuilder.complete({ fee: (currentFee + 10000n).toString() });
    txBuilder.completeSigning();
    return await ogmios.submitTx(txBuilder.txHex);
  }
  throw err;
}
```

**Why generous manual exUnits help**: The fee formula includes
`priceStep * totalSteps + priceMem * totalMem`. Generous exUnits inflate
the calculated fee, naturally providing padding. This is why the CIP-113
programmable-tokens tests (15M total mem budget) never hit the issue.

### G-K6: Free-Text On-Chain Data Is an Abuse Vector

**Symptom**: Users can submit arbitrary text that gets permanently stored in
transaction metadata or inline datums on the blockchain.

**Root cause**: Accepting free-text fields (description, comments) in on-chain
data allows storage of illegal content, PII, profanity, or XSS payloads that
appear in blockchain explorers.

**Fix**: Replace free-text with generated references. Use a deterministic format
like `[TICKER]-[SERVICE]-[YYMMDD]-[SHORT_HASH]` (e.g. `ADV-N-260316-a7b3c`).
Store the human-readable description off-chain in your application database,
linked by the reference. Benefits: predictable datum size, predictable fees,
no abuse vector, still provides a correlation key for off-chain records.

### G-K7: API Key Caching in localStorage Creates Stale Auth State

**Symptom**: Notarization fails with "Key revoked" or "Unauthorized" even though
the user just connected their wallet. Works after clearing localStorage.

**Root cause**: Browser-cached API keys go stale when the backend revokes them
(key rotation, force re-provision, server restart). The frontend keeps using
the cached key, unaware it's been invalidated. This is especially problematic
when the key provisioning endpoint revokes old keys as a side effect of creating
new ones.

**Fix**: API keys cached in `localStorage` are a temporary pattern for MVP.
The proper solution is wallet-based authentication (CIP-30 challenge-sign),
where the wallet IS the identity and there's no key to cache or revoke.

For MVPs that must use API keys:
- Return existing active keys on re-request instead of revoking and recreating
- Support a `force: true` parameter for explicit re-provisioning
- On auth failure (401/403), clear the cached key and retry once with a fresh key
- Never revoke keys as a side effect of another operation

### G-K8: Response Headers Cannot Be Set in onFinished Callbacks

**Symptom**: Custom response headers (e.g. `X-Credits-Remaining`) are never
received by the client despite being set in the Express middleware.

**Root cause**: Express/Node.js `on-finished` callbacks fire after the response
body has been flushed to the network. Calling `res.setHeader()` at that point
is a no-op — the headers have already been sent. This affects any pattern where
you want to include post-processing data (credits deducted, audit trail) in
response headers.

**Fix**: Include the data in the response body instead of headers. For credit
deductions, calculate the remaining balance in the route handler before
`res.json()`, not in the middleware's `onFinished` callback:

```typescript
// In the route handler, BEFORE sending the response:
let creditsRemaining;
if (req.credits) {
  const balance = engine.getBalance(req.credits.keyId);
  creditsRemaining = Math.max(0, balance.available - req.credits.cost);
}

res.status(202).json({
  data: { ...result, credits_remaining: creditsRemaining },
  meta: { timestamp: new Date().toISOString() },
});

// The middleware's onFinished handles the actual deduction
// (which happens after the response is sent — that's fine for writes)
```

---

## See Also

- [CIP-30 Standard](cip30.md) -- wallet API details
- [Infrastructure](infrastructure.md) -- Ogmios, Kupo, Docker
- [Testing](testing.md) -- mock wallet patterns
- [API Resilience](api-resilience.md) -- failover implementation
