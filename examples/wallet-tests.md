# Example: Wallet E2E Tests

Complete Playwright test suite for CIP-30 wallet integration: connect, disconnect,
balance display, wrong network, user rejection, auto-reconnect with late injection,
Eternl isEnabled bug, and ViewTransitions persistence.

## Key Concepts

- **Desktop nav selector**: `.hidden.md\\:flex` scopes to desktop nav (avoids mobile duplicates)
- **Modal button targeting**: `page.locator('button').filter({ hasText: name }).last()`
- **Connected indicator**: Green dot (`.bg-success`) in the wallet button
- **ViewTransitions persistence**: Wallet state survives SPA navigation
- **Auto-reconnect with isEnabled false**: Tests the Eternl bug workaround

## Source Code

### Wallet Integration Tests

```typescript
// e2e/wallet.spec.ts
import { test, expect } from './fixtures/wallet-mock';
import { ROUTES, TEST_BALANCE_DISPLAY, NETWORK } from './fixtures/test-data';

// Desktop nav container -- hidden on mobile, visible md+
const desktopNav = '.hidden.md\\:flex';

// Helper: click wallet button in the selection modal
async function clickWalletInModal(page: import('@playwright/test').Page, walletName: string) {
  await page.locator('button').filter({ hasText: walletName }).last().click();
}

// Helper: get the connected button in desktop nav (contains green dot)
function connectedButton(page: import('@playwright/test').Page) {
  return page.locator(`${desktopNav} button`).filter({ has: page.locator('.bg-success') });
}

test.describe('Wallet integration', () => {
  test('connect wallet via modal shows connected state', async ({ page, mockWallet }) => {
    await mockWallet.inject();
    await page.goto(ROUTES.HOME);

    // Click connect button (desktop nav)
    await page.locator(desktopNav).getByText('Connect Wallet').click();

    // Modal should appear -- click Eternl
    await clickWalletInModal(page, 'Eternl');

    // Should show connected state: green dot visible in desktop nav
    await expect(page.locator(`${desktopNav} .bg-success`).first()).toBeVisible({
      timeout: 10_000,
    });
  });

  test('disconnect returns to Connect Wallet button', async ({ page, mockWallet }) => {
    await mockWallet.inject();
    await page.goto(ROUTES.HOME);

    // Connect first
    await page.locator(desktopNav).getByText('Connect Wallet').click();
    await clickWalletInModal(page, 'Eternl');
    await expect(page.locator(`${desktopNav} .bg-success`).first()).toBeVisible({
      timeout: 10_000,
    });

    // Click the connected button to open dropdown
    await connectedButton(page).click();
    await page.getByText('Disconnect').click();

    // Should be back to disconnected
    await expect(page.locator(desktopNav).getByText('Connect Wallet')).toBeVisible({
      timeout: 5_000,
    });
  });

  test('balance displays correctly in dropdown', async ({ page, mockWallet }) => {
    await mockWallet.inject();
    await page.goto(ROUTES.HOME);

    // Connect
    await page.locator(desktopNav).getByText('Connect Wallet').click();
    await clickWalletInModal(page, 'Eternl');
    await expect(page.locator(`${desktopNav} .bg-success`).first()).toBeVisible({
      timeout: 10_000,
    });

    // Open dropdown
    await connectedButton(page).click();

    // Check balance display
    await expect(page.getByText(TEST_BALANCE_DISPLAY)).toBeVisible();
    await expect(page.getByText('ADA', { exact: true })).toBeVisible();
  });

  test('wrong network shows error', async ({ page, mockWallet }) => {
    await mockWallet.inject({ networkId: NETWORK.TESTNET });
    await page.goto(ROUTES.HOME);

    await page.locator(desktopNav).getByText('Connect Wallet').click();
    await clickWalletInModal(page, 'Eternl');

    // Should show mainnet error
    await expect(page.getByText(/mainnet/i)).toBeVisible({ timeout: 10_000 });
  });

  test('wallet rejection handled gracefully', async ({ page, mockWallet }) => {
    await mockWallet.inject({ enableRejects: true });
    await page.goto(ROUTES.HOME);

    await page.locator(desktopNav).getByText('Connect Wallet').click();
    await clickWalletInModal(page, 'Eternl');

    // Should show error, not crash
    await expect(page.getByText(/declined/i)).toBeVisible({ timeout: 10_000 });
  });

  test('auto-reconnect after page reload', async ({ page, mockWallet }) => {
    await mockWallet.inject();
    await page.goto(ROUTES.HOME);

    // Connect wallet
    await page.locator(desktopNav).getByText('Connect Wallet').click();
    await clickWalletInModal(page, 'Eternl');
    await expect(page.locator(`${desktopNav} .bg-success`).first()).toBeVisible({
      timeout: 10_000,
    });

    // Reload page (mock wallet re-injected via addInitScript)
    await page.reload();

    // Should auto-reconnect -- green dot visible again
    await expect(page.locator(`${desktopNav} .bg-success`).first()).toBeVisible({
      timeout: 10_000,
    });
  });

  test('auto-reconnect when isEnabled returns false (Eternl bug)', async ({
    page,
    mockWallet,
  }) => {
    // First visit: connect normally
    await mockWallet.inject({ isEnabled: true });
    await page.goto(ROUTES.HOME);

    await page.locator(desktopNav).getByText('Connect Wallet').click();
    await clickWalletInModal(page, 'Eternl');
    await expect(page.locator(`${desktopNav} .bg-success`).first()).toBeVisible({
      timeout: 10_000,
    });

    // Simulate Eternl bug: isEnabled returns false after reload
    await mockWallet.inject({ isEnabled: false });
    await page.reload();

    // Auto-reconnect should still work because we skip isEnabled()
    await expect(page.locator(`${desktopNav} .bg-success`).first()).toBeVisible({
      timeout: 10_000,
    });
  });

  test('auto-reconnect with late wallet injection', async ({ page, mockWallet }) => {
    // Wallet appears 800ms after page load
    await mockWallet.inject({ injectionDelay: 800 });
    await mockWallet.setStorage('eternl');
    await page.goto(ROUTES.HOME);

    // Retry logic should catch the late injection
    await expect(page.locator(`${desktopNav} .bg-success`).first()).toBeVisible({
      timeout: 15_000,
    });
  });
});
```

### Navigation + Wallet Persistence Tests

```typescript
// e2e/navigation.spec.ts
import { test, expect } from './fixtures/wallet-mock';
import { ROUTES } from './fixtures/test-data';

const desktopNav = '.hidden.md\\:flex';

test.describe('Navigation + wallet persistence', () => {
  test('wallet persists: homepage -> pool page (same layout)', async ({
    page,
    mockWallet,
  }) => {
    await mockWallet.inject();
    await page.goto(ROUTES.HOME);

    // Connect wallet
    await page.locator(desktopNav).getByText('Connect Wallet').click();
    await page.locator('button').filter({ hasText: 'Eternl' }).last().click();
    await expect(page.locator(`${desktopNav} .bg-success`).first()).toBeVisible({
      timeout: 10_000,
    });

    // Navigate to pool page via desktop nav link
    await page.locator(`${desktopNav} a[href="/pools/"]`).click();
    await expect(page).toHaveURL(/\/pools\//);

    // Wallet should still be connected (green dot in desktop nav)
    await expect(page.locator(`${desktopNav} .bg-success`).first()).toBeVisible({
      timeout: 10_000,
    });
  });

  test('wallet persists: homepage -> blog (cross-layout)', async ({
    page,
    mockWallet,
  }) => {
    await mockWallet.inject();
    await page.goto(ROUTES.HOME);

    // Connect wallet
    await page.locator(desktopNav).getByText('Connect Wallet').click();
    await page.locator('button').filter({ hasText: 'Eternl' }).last().click();
    await expect(page.locator(`${desktopNav} .bg-success`).first()).toBeVisible({
      timeout: 10_000,
    });

    // Navigate to blog via desktop nav (different layout)
    await page.locator(`${desktopNav} a[href="/blog/"]`).click();
    await expect(page).toHaveURL(/\/blog\//);

    // Wallet should still be connected
    await expect(page.locator(`${desktopNav} .bg-success`).first()).toBeVisible({
      timeout: 10_000,
    });
  });

  test('basic page navigation works', async ({ page }) => {
    await page.goto(ROUTES.ABOUT);
    await expect(page.locator('h1')).toContainText('Securing the Future');

    // Navigate to pools via desktop nav
    await page.locator(`${desktopNav} a[href="/pools/"]`).click();
    await expect(page).toHaveURL(/\/pools\//);
    await expect(page.locator('h1')).toContainText('Choose Your Pool');
  });
});
```

## Selector Reference

| Element | Selector | Notes |
|---------|----------|-------|
| Desktop nav | `.hidden.md\\:flex` | Escaping Tailwind colon |
| Connect button | `desktopNav + getByText('Connect Wallet')` | Scoped to desktop |
| Modal wallet button | `button filter({ hasText: 'Eternl' }).last()` | `.last()` targets modal |
| Connected indicator | `desktopNav + .bg-success` | Green dot |
| Connected button | `button filter({ has: .bg-success })` | Contains green dot |
| Disconnect | `getByText('Disconnect')` | In dropdown |
| Nav link | `desktopNav + a[href="/pools/"]` | Desktop nav links |

## Common Mistakes

### 1. Not Scoping to Desktop Nav

```typescript
// WRONG: Matches both desktop and mobile nav elements
await page.getByText('Connect Wallet').click();
// Error: strict mode violation, resolved to 2 elements

// RIGHT: Scope to desktop nav
await page.locator(desktopNav).getByText('Connect Wallet').click();
```

### 2. Using `.first()` on Modal Buttons

```typescript
// WRONG: .first() might get a non-modal button
await page.locator('button').filter({ hasText: 'Eternl' }).first().click();

// RIGHT: .last() targets the modal (rendered later in DOM via portal)
await page.locator('button').filter({ hasText: 'Eternl' }).last().click();
```

### 3. Insufficient Timeout for Auto-Reconnect

```typescript
// WRONG: 5s might not be enough for late injection + retry
await expect(greenDot).toBeVisible({ timeout: 5_000 });

// RIGHT: Allow for full retry cycle + connection time
await expect(greenDot).toBeVisible({ timeout: 15_000 });
```

### 4. Not Re-Injecting Mock on Reload

The `addInitScript` persists through `page.reload()` -- it re-runs on every
navigation. However, when testing different configurations (e.g., isEnabled change),
you must call `mockWallet.inject()` again with the new config before `reload()`:

```typescript
// Re-inject with new config before reload
await mockWallet.inject({ isEnabled: false });
await page.reload();
```

## See Also

- [Mock Wallet](mock-wallet.md) -- the mock implementation these tests use
- [Testing Reference](../reference/testing.md) -- Playwright configuration
