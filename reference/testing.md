# E2E Testing with Playwright

This document covers end-to-end testing of CIP-30 wallet features using Playwright.
The core technique: inject a mock CIP-30 wallet via `addInitScript` before page
navigation, then test wallet connect/disconnect, balance display, error handling,
auto-reconnect, and ViewTransitions persistence.

## Playwright Configuration

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: process.env.CI ? 'github' : 'html',

  use: {
    baseURL: 'http://localhost:4321',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },

  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
  ],

  webServer: {
    command: 'npm run preview',
    url: 'http://localhost:4321',
    reuseExistingServer: !process.env.CI,
    timeout: 30_000,
  },
});
```

Key choices:
- **Single browser (Chromium)**: Wallet extensions are Chromium-only in practice
- **`npm run preview`**: Tests against the production build, not dev server
- **`reuseExistingServer`**: Reuse existing server during local development, start
  fresh in CI
- **Screenshots on failure**: Helps debug CI-only failures

## MockWalletConfig Interface

The mock wallet is fully configurable per-test:

```typescript
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
  /** Delay in ms before wallet appears in window.cardano (simulates late injection) */
  injectionDelay: number;
}
```

Default configuration simulates a mainnet Eternl wallet with 1000 ADA:

```typescript
const DEFAULT_CONFIG: MockWalletConfig = {
  name: 'eternl',
  isEnabled: true,
  enableRejects: false,
  networkId: 1,  // mainnet
  stakeAddress: 'e0a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1',
  balanceCbor: '1a3b9aca00',  // 1,000,000,000 lovelace = 1000 ADA
  utxos: [],
  injectionDelay: 0,
};
```

## Mock Injection with `addInitScript`

The mock must be injected before `page.goto()` because `addInitScript` runs before
any page JavaScript:

```typescript
export async function injectMockWallet(
  page: Page,
  overrides: Partial<MockWalletConfig> = {},
): Promise<MockWalletConfig> {
  const config = { ...DEFAULT_CONFIG, ...overrides };

  await page.addInitScript(
    (cfg: MockWalletConfig) => {
      const inject = () => {
        const api = {
          getNetworkId: async () => cfg.networkId,
          getRewardAddresses: async () => [cfg.stakeAddress],
          getBalance: async () => cfg.balanceCbor,
          getUtxos: async () => cfg.utxos,
          signTx: async () => { throw new Error('Mock: signTx not implemented'); },
          submitTx: async () => { throw new Error('Mock: submitTx not implemented'); },
        };

        const extension = {
          name: cfg.name,
          icon: 'data:image/svg+xml,<svg/>',
          apiVersion: '0.1.0',
          isEnabled: async () => cfg.isEnabled,
          enable: async () => {
            if (cfg.enableRejects) throw new Error('user declined');
            return api;
          },
        };

        if (!(window as any).cardano) (window as any).cardano = {};
        (window as any).cardano[cfg.name] = extension;
      };

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
```

### Late Injection Simulation

The `injectionDelay` parameter simulates real wallet extension timing:

```typescript
// Wallet appears 800ms after page load
await mockWallet.inject({ injectionDelay: 800 });
await mockWallet.setStorage('eternl');
await page.goto('/');

// Retry logic should catch the late injection
await expect(page.locator('.bg-success').first()).toBeVisible({ timeout: 15_000 });
```

### Pre-Setting localStorage

To test auto-reconnect, pre-set the wallet storage key via `addInitScript`:

```typescript
export async function setWalletStorage(page: Page, walletName: string): Promise<void> {
  await page.addInitScript(
    (name: string) => {
      localStorage.setItem('adavault_connected_wallet', name);
    },
    walletName,
  );
}
```

## Playwright Test Fixture

Wrap the mock functions in a Playwright fixture for clean test code:

```typescript
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

Usage in tests:

```typescript
import { test, expect } from './fixtures/wallet-mock';

test('connect wallet', async ({ page, mockWallet }) => {
  await mockWallet.inject();
  await page.goto('/');
  // ...
});
```

## Selector Patterns

### Desktop vs Mobile Navigation

Desktop and mobile navbars are separate DOM elements. To avoid Playwright's strict
mode error when matching duplicates, scope selectors to the desktop nav:

```typescript
// Desktop nav container -- hidden on mobile, visible md+
const desktopNav = '.hidden.md\\:flex';

// Click connect button in desktop nav only
await page.locator(desktopNav).getByText('Connect Wallet').click();

// Check green dot in desktop nav
await expect(page.locator(`${desktopNav} .bg-success`).first()).toBeVisible();
```

Note the escaped backslash: `.hidden.md\\:flex` in Playwright selector syntax.

### Modal Wallet Buttons

The wallet selection modal may contain buttons with the same text as other page
elements. Use `.filter()` and `.last()` to target the modal button specifically:

```typescript
// Click the wallet button in the modal
await page.locator('button').filter({ hasText: 'Eternl' }).last().click();
```

### Connected State Indicator

The connected wallet button contains a green dot element:

```typescript
// Get the connected button (contains green dot)
function connectedButton(page: Page) {
  return page.locator(`${desktopNav} button`).filter({
    has: page.locator('.bg-success'),
  });
}

// Open the connected wallet dropdown
await connectedButton(page).click();
```

## Test Patterns

### Connect/Disconnect Cycle

```typescript
test('connect and disconnect wallet', async ({ page, mockWallet }) => {
  await mockWallet.inject();
  await page.goto('/');

  // Connect
  await page.locator(desktopNav).getByText('Connect Wallet').click();
  await page.locator('button').filter({ hasText: 'Eternl' }).last().click();
  await expect(page.locator(`${desktopNav} .bg-success`).first()).toBeVisible({ timeout: 10_000 });

  // Disconnect
  await connectedButton(page).click();
  await page.getByText('Disconnect').click();
  await expect(page.locator(desktopNav).getByText('Connect Wallet')).toBeVisible({ timeout: 5_000 });
});
```

### Wrong Network

```typescript
test('wrong network shows error', async ({ page, mockWallet }) => {
  await mockWallet.inject({ networkId: 0 });  // testnet
  await page.goto('/');
  await page.locator(desktopNav).getByText('Connect Wallet').click();
  await page.locator('button').filter({ hasText: 'Eternl' }).last().click();
  await expect(page.getByText(/mainnet/i)).toBeVisible({ timeout: 10_000 });
});
```

### Auto-Reconnect with Eternl isEnabled Bug

```typescript
test('auto-reconnect when isEnabled returns false', async ({ page, mockWallet }) => {
  // First visit: connect normally
  await mockWallet.inject({ isEnabled: true });
  await page.goto('/');
  // ... connect ...

  // Simulate Eternl bug: isEnabled false, but enable() still works
  await mockWallet.inject({ isEnabled: false });
  await page.reload();

  // Should still auto-reconnect (we skip isEnabled)
  await expect(page.locator(`${desktopNav} .bg-success`).first()).toBeVisible({ timeout: 10_000 });
});
```

### ViewTransitions Persistence

```typescript
test('wallet persists across page navigation', async ({ page, mockWallet }) => {
  await mockWallet.inject();
  await page.goto('/');

  // Connect
  await page.locator(desktopNav).getByText('Connect Wallet').click();
  await page.locator('button').filter({ hasText: 'Eternl' }).last().click();
  await expect(page.locator(`${desktopNav} .bg-success`).first()).toBeVisible({ timeout: 10_000 });

  // Navigate to a different page
  await page.locator(`${desktopNav} a[href="/pools/"]`).click();
  await expect(page).toHaveURL(/\/pools\//);

  // Wallet should still be connected
  await expect(page.locator(`${desktopNav} .bg-success`).first()).toBeVisible({ timeout: 10_000 });
});
```

## CI Considerations

- **E2E tests require a built `dist/`**: Run `npm run build` before `npm run test:e2e`
- **Chromium system deps**: CI environments may need `--with-deps` for Playwright install
- **Retries in CI**: Set `retries: 2` to handle flaky network/timing issues
- **Single worker in CI**: `workers: 1` avoids resource contention
- **HTML reporter locally, GitHub reporter in CI**: Different output formats for
  different consumption contexts

## See Also

- [CIP-30 Standard](cip30.md) -- the API being mocked
- [Wallet Integration](wallet-integration.md) -- the patterns being tested
- [Gotchas](gotchas.md) -- testing-specific issues
