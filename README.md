# Cardano Off-Chain & dApp Frontend Skill

A [Claude Code skill](https://code.claude.com/docs/en/skills) for building Cardano dApp frontends with CIP-30 wallet integration, MeshJS transaction building, and Playwright E2E testing.

Following the [Agent Skills](https://agentskills.io) open standard -- works with Claude Code, Cursor, Gemini CLI, VS Code Copilot, and 30+ other AI coding tools.

> **Important:** This skill captures production-tested patterns for off-chain Cardano development but does not replace thorough security review. Wallet integrations handling real funds should be audited for XSS vectors, CBOR parsing edge cases, and transaction validation before mainnet deployment.

## Summary

| Metric | Count |
|--------|-------|
| Reference docs | 8 |
| Working examples | 8 |
| Decision trees | 3 |
| Wallet quirks documented | 11 |
| CIP-30 API methods covered | 11 |
| Production-validated wallets | 7 (Eternl, Lace, Vespr, Flint, Yoroi, Typhon, Gero) |

All patterns extracted from a production Cardano dApp ([adavault.com](https://adavault.com)) with live wallet integration, delegation transactions, ADA Handle resolution, and 17 Playwright E2E tests using CIP-30 mock wallets.

## Features

### Reference Documentation (8 files)

- **CIP-30** -- full wallet lifecycle from extension detection to auto-reconnect
- **Wallet Integration** -- React hooks, SSR safety, state management, auto-reconnect
- **MeshJS Transactions** -- BrowserWallet, KoiosProvider, delegation, build/sign/submit
- **CBOR** -- balance decoding, UTxO parsing, multi-asset value handling
- **API Resilience** -- failover with 5s timeout, AbortController, DR architecture
- **Infrastructure** -- Ogmios, Kupo, Docker deployment, HAProxy, SSH tunnels
- **Testing** -- Playwright mock wallet fixture, CIP-30 simulation, E2E patterns
- **Gotchas** -- wallet quirks, CORS workarounds, MeshJS pitfalls, Kupo operational issues

### Working Examples (8 files)

| Example | Pattern |
|---------|---------|
| Wallet Connect | `useWallet()` -- connect, disconnect, auto-reconnect, state broadcast |
| Auto-Reconnect | Retry with backoff, localStorage persistence, stale state cleanup |
| Delegation | MeshJS Transaction + BrowserWallet + KoiosProvider |
| API Failover | `apiFetch()` with 5s timeout, AbortController, DR endpoint |
| Mock Wallet | Playwright CIP-30 mock with configurable behaviour and late injection |
| Wallet Tests | E2E suite -- connect, disconnect, wrong network, persistence |
| ADA Handle | UTxO hex scanning with CIP-68 reference token support |
| CBOR Balance | Minimal decoder for CIP-30 `getBalance()` (pure lovelace + multi-asset) |

## Installation

### Project-Level (Recommended)

```bash
mkdir -p .claude/skills
git clone https://github.com/ADAvault/cardano-offchain-skill.git .claude/skills/cardano-offchain
```

### Personal (All Projects)

```bash
git clone https://github.com/ADAvault/cardano-offchain-skill.git ~/.claude/skills/cardano-offchain
```

## Usage

The skill activates automatically when your conversation involves CIP-30, wallets, BrowserWallet, MeshJS, delegation, UTxO queries, ADA Handle, CBOR decoding, Ogmios, Kupo, or dApp frontend patterns.

Or invoke explicitly: `/cardano-offchain`

## Structure

```
cardano-offchain-skill/
├── SKILL.md                           # Core skill -- decision trees, inline patterns
├── README.md                          # This file
├── LICENSE                            # MIT
├── reference/
│   ├── cip30.md                      # CIP-30 wallet standard API
│   ├── wallet-integration.md         # React wallet patterns (SSR, state, persistence)
│   ├── transactions.md               # MeshJS browser transaction building
│   ├── api-resilience.md             # Failover, timeout, CORS proxy, data pipeline
│   ├── cbor.md                       # CBOR encoding/decoding for CIP-30
│   ├── infrastructure.md             # Ogmios, Kupo, Docker, HAProxy
│   ├── testing.md                    # Playwright mock wallet and E2E patterns
│   └── gotchas.md                    # Hard-won learnings from production
└── examples/
    ├── wallet-connect.md             # Full useWallet() React hook
    ├── auto-reconnect.md             # Retry with backoff, persistence
    ├── delegation.md                 # Stake delegation transaction
    ├── api-failover.md               # apiFetch() with DR failover
    ├── mock-wallet.md                # Playwright CIP-30 fixture
    ├── wallet-tests.md               # E2E test suite
    ├── ada-handle.md                 # ADA Handle extraction (CIP-68)
    └── cbor-balance.md               # CBOR balance decoding
```

## Key Findings from Validation

Building and testing against 7 real wallet extensions and a production dApp uncovered patterns not documented elsewhere:

- **`isEnabled()` is unreliable** -- Eternl returns false after reload for authorized sites. Auto-reconnect must skip `isEnabled()` and call `enable()` directly.
- **Late injection is common** -- wallet extensions inject into `window.cardano` asynchronously. A single check on page load misses wallets that arrive 200-500ms later. Retry with backoff.
- **CBOR multi-asset gotcha** -- `getBalance()` returns a bare unsigned int for pure-ADA wallets but a CBOR array for wallets holding native tokens. Your decoder must handle both.
- **CIP-68 changed handle encoding** -- ADA Handle migrated to CIP-68 reference tokens. The asset name now has a `000de140` prefix that must be stripped before decoding.
- **ViewTransitions destroy React state** -- Astro's ViewTransitions unmount and remount components on navigation. Without `transition:persist`, wallet state resets silently.
- **Koios CORS requires a registered account** -- browser-to-Koios requests fail without CORS headers for unregistered API accounts. Proxy through your own API.

## Sources

Built from:

- **CIP-30 specification** -- Cardano wallet-web bridge standard
- **MeshJS SDK** -- off-chain transaction building (v1.x)
- **Production deployment** -- [adavault.com](https://adavault.com) with 7 wallet integrations since R2.0
- **Playwright E2E suite** -- 17 tests with CIP-30 mock wallet fixture
- **cardano-notary** -- 61 E2E operations with Ogmios/Kupo on preview testnet
- **programmable-tokens** -- CIP-113 lifecycle with manual exUnits and multi-validator patterns
- **adavault-web commit history** -- months of wallet bug fixes, ViewTransitions work, API resilience

## Companion Skills

- [cardano-skill](https://github.com/ADAvault/cardano-skill) -- Aiken smart contract development (on-chain validators, testing, auditing)
- [midnight-skill](https://github.com/ADAvault/midnight-skill) -- Compact smart contracts on Midnight (privacy, ZK proofs)

## Contributing

PRs welcome -- especially from dApp developers, wallet teams, and Cardano builders. Areas of interest:

- Additional wallet quirks and workarounds
- MeshJS v2 migration patterns
- Alternative off-chain frameworks (Lucid, cardano-js-sdk)
- Ogmios/Kupo operational patterns
- Mobile wallet integration (WalletConnect, deep links)

## License

MIT -- see [LICENSE](LICENSE).

## Credits

Built by [ADAvault](https://adavault.com) -- Cardano stake pool operators since epoch 208.
