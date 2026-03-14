# Example: Mock Wallet for Playwright

A complete CIP-30 mock wallet injected via Playwright's `addInitScript` for E2E
testing. Supports configurable behavior per-test: network ID, balance, UTxOs,
enable rejection, and late injection timing.

## Key Concepts

- **`addInitScript` runs before page JS**: Inject the mock before `page.goto()`
- **`MockWalletConfig`**: Full configuration interface for per-test behavior
- **Late injection via `setTimeout`**: Simulates real extension timing for testing retry logic
- **`setWalletStorage`**: Pre-set localStorage to trigger auto-reconnect on page load
- **Playwright fixture pattern**: Clean `test.extend()` integration

## Source Code

### Test Data Constants

```typescript
// e2e/fixtures/test-data.ts

// Synthetic mainnet stake address (hex, e0 prefix = mainnet reward address)
export const TEST_STAKE_ADDRESS_HEX =
  'e0a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1';

// CBOR-encoded balance: 1,000,000,000 lovelace (1000 ADA)
// 0x1A = unsigned int, 4-byte value; 0x3B9ACA00 = 1000000000
export const TEST_BALANCE_CBOR = '1a3b9aca00';

export const TEST_BALANCE_LOVELACE = 1_000_000_000;
export const TEST_BALANCE_DISPLAY = '1,000.00';

export const ROUTES = {
  HOME: '/',
  POOLS: '/pools/',
  POOL_ADV: '/pools/adv/',
  ABOUT: '/about/',
  BLOG: '/blog/',
} as const;

export const NETWORK = {
  MAINNET: 1,
  TESTNET: 0,
} as const;
```

### Mock Wallet Implementation

```typescript
// e2e/fixtures/wallet-mock.ts
import { test as base, type Page } from '@playwright/test';
import { TEST_STAKE_ADDRESS_HEX, TEST_BALANCE_CBOR, NETWORK } from './test-data';

export interface MockWalletConfig {
  /** Wallet extension name (injected as window.cardano[name]) */
  name: string;
  /** What isEnabled() returns */
  isEnabled: boolean;
  /** If true, enable() rejects with "user declined" */
  enableRejects: boolean;
  /** Network ID: 1 = mainnet, 0 = testnet */
  networkId: number;
  /** Hex-encoded stake/reward address */
  stakeAddress: string;
  /** CBOR-encoded balance */
  balanceCbor: string;
  /** UTXO hex strings (empty = no handles) */
  utxos: string[];
  /** Delay in ms before wallet appears in window.cardano */
  injectionDelay: number;
}

const DEFAULT_CONFIG: MockWalletConfig = {
  name: 'eternl',
  isEnabled: true,
  enableRejects: false,
  networkId: NETWORK.MAINNET,
  stakeAddress: TEST_STAKE_ADDRESS_HEX,
  balanceCbor: TEST_BALANCE_CBOR,
  utxos: [],
  injectionDelay: 0,
};

/**
 * Inject a CIP-30 compliant mock wallet into the page.
 * Must be called before page.goto() -- uses addInitScript.
 */
export async function injectMockWallet(
  page: Page,
  overrides: Partial<MockWalletConfig> = {},
): Promise<MockWalletConfig> {
  const config = { ...DEFAULT_CONFIG, ...overrides };

  await page.addInitScript(
    (cfg: MockWalletConfig) => {
      const inject = () => {
        // CIP-30 Wallet API methods
        const api = {
          getNetworkId: async () => cfg.networkId,
          getRewardAddresses: async () => [cfg.stakeAddress],
          getBalance: async () => cfg.balanceCbor,
          getUtxos: async () => cfg.utxos,
          signTx: async () => {
            throw new Error('Mock: signTx not implemented');
          },
          submitTx: async () => {
            throw new Error('Mock: submitTx not implemented');
          },
        };

        // CIP-30 extension object (pre-enable)
        const extension = {
          name: cfg.name,
          icon: 'data:image/svg+xml,<svg/>',
          apiVersion: '0.1.0',
          isEnabled: async () => cfg.isEnabled,
          enable: async () => {
            if (cfg.enableRejects) {
              throw new Error('user declined');
            }
            return api;
          },
        };

        // Inject into window.cardano namespace
        if (!(window as any).cardano) {
          (window as any).cardano = {};
        }
        (window as any).cardano[cfg.name] = extension;
      };

      // Support delayed injection to test retry logic
      if (cfg.injectionDelay > 0) {
        setTimeout(inject, cfg.injectionDelay);
      } else {
        inject();
      }
    },
    config,
  );

  return config;
}

/**
 * Pre-set localStorage so auto-reconnect fires on page load.
 * Must be called before page.goto().
 */
export async function setWalletStorage(
  page: Page,
  walletName: string,
): Promise<void> {
  await page.addInitScript(
    (name: string) => {
      localStorage.setItem('adavault_connected_wallet', name);
    },
    walletName,
  );
}

// Playwright test fixture
type WalletFixtures = {
  mockWallet: {
    inject: (overrides?: Partial<MockWalletConfig>) => Promise<MockWalletConfig>;
    setStorage: (walletName: string) => Promise<void>;
  };
};

export const test = base.extend<WalletFixtures>({
  mockWallet: async ({ page }, use) => {
    await use({
      inject: (overrides) => injectMockWallet(page, overrides),
      setStorage: (name) => setWalletStorage(page, name),
    });
  },
});

export { expect } from '@playwright/test';
```

## Per-Test Configuration Examples

```typescript
// Default: mainnet Eternl with 1000 ADA, immediate injection
await mockWallet.inject();

// Wrong network (testnet)
await mockWallet.inject({ networkId: NETWORK.TESTNET });

// User rejects connection
await mockWallet.inject({ enableRejects: true });

// Late injection (800ms delay -- tests retry logic)
await mockWallet.inject({ injectionDelay: 800 });
await mockWallet.setStorage('eternl');  // Pre-set storage for auto-reconnect

// Eternl isEnabled bug (returns false but enable() still works)
await mockWallet.inject({ isEnabled: false });

// Different wallet
await mockWallet.inject({ name: 'lace' });

// Custom balance (10 ADA = 10,000,000 lovelace)
// CBOR: 0x1A 0x00989680
await mockWallet.inject({ balanceCbor: '1a00989680' });
```

## Common Mistakes

### 1. Injecting After `page.goto()`

```typescript
// WRONG: Mock arrives too late, page JS already ran
await page.goto('/');
await mockWallet.inject();

// RIGHT: Inject before navigation
await mockWallet.inject();
await page.goto('/');
```

### 2. Forgetting `setStorage` for Auto-Reconnect Tests

```typescript
// WRONG: Auto-reconnect has nothing in localStorage to trigger on
await mockWallet.inject({ injectionDelay: 800 });
await page.goto('/');  // No auto-reconnect attempt

// RIGHT: Pre-set the storage key
await mockWallet.inject({ injectionDelay: 800 });
await mockWallet.setStorage('eternl');
await page.goto('/');  // Auto-reconnect triggers, retries until extension injects
```

### 3. Not Implementing All Required CIP-30 Methods

If your app calls `getUsedAddresses()` or `getChangeAddress()`, the mock must
implement them. Missing methods cause runtime errors that are hard to trace:

```typescript
// Add methods as needed
const api = {
  getNetworkId: async () => cfg.networkId,
  getRewardAddresses: async () => [cfg.stakeAddress],
  getBalance: async () => cfg.balanceCbor,
  getUtxos: async () => cfg.utxos,
  getUsedAddresses: async () => [cfg.paymentAddress],  // Add if needed
  getChangeAddress: async () => cfg.changeAddress,      // Add if needed
  signTx: async () => { throw new Error('Mock: not implemented'); },
  submitTx: async () => { throw new Error('Mock: not implemented'); },
};
```

### 4. Using `page.evaluate` Instead of `addInitScript`

```typescript
// WRONG: evaluate runs after page JS, too late for wallet detection
await page.goto('/');
await page.evaluate(() => {
  window.cardano = { eternl: { ... } };
});

// RIGHT: addInitScript runs before any page JS
await page.addInitScript((cfg) => { ... }, config);
await page.goto('/');
```

## See Also

- [Wallet Tests](wallet-tests.md) -- E2E tests using this mock
- [Testing Reference](../reference/testing.md) -- Playwright configuration and patterns
