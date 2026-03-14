# Example: API Failover

Resilient fetch wrapper with `AbortController` timeout and automatic failover
from primary to DR API endpoint.

## Key Concepts

- **`AbortController` + 5s timeout**: Abort stalled requests quickly
- **Primary/fallback endpoints**: Automatic switch on failure
- **Dev mode bypass**: Skip fallback during local development
- **4xx vs 5xx handling**: Client errors returned directly; only server errors trigger failover
- **`import.meta.env.DEV`**: Astro/Vite environment variable for dev detection

## Source Code

```typescript
// src/config.ts

// API endpoint configuration
// Production: data from api.adavault.com with DR failover to api2
// Development: api-dev.adavault.com with localhost CORS

const isDev = import.meta.env.DEV;

const API_PRIMARY = isDev
  ? 'https://api-dev.adavault.com/api/v1'  // Dev API with localhost CORS
  : 'https://api.adavault.com/api/v1';

const API_FALLBACK = 'https://api2.adavault.com/api/v1';

// Legacy export for backwards compatibility
export const API_BASE_URL = API_PRIMARY;

// Helper function to build API URLs
export function apiUrl(path: string): string {
  const normalizedPath = path.startsWith('/') ? path : `/${path}`;
  return `${API_PRIMARY}${normalizedPath}`;
}

// Resilient fetch with automatic failover to backup API
export async function apiFetch(path: string, options?: RequestInit): Promise<Response> {
  const normalizedPath = path.startsWith('/') ? path : `/${path}`;
  const primaryUrl = `${API_PRIMARY}${normalizedPath}`;
  const fallbackUrl = `${API_FALLBACK}${normalizedPath}`;

  try {
    // Primary request with 5-second timeout
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), 5000);

    const res = await fetch(primaryUrl, {
      ...options,
      signal: controller.signal,
    });

    clearTimeout(timeoutId);

    // Success -- return response
    if (res.ok) return res;

    // Server error -- trigger failover
    throw new Error(`Primary API error: ${res.status}`);
  } catch (err) {
    // In dev mode, don't try fallback (api2 won't be available locally)
    if (isDev) throw err;

    console.warn('Primary API failed, trying fallback:', err);
    return fetch(fallbackUrl, options);
  }
}
```

## Enhanced Version

The production code above has two areas for improvement. This enhanced version
adds timeout on the fallback and proper 4xx handling:

```typescript
export async function apiFetch(path: string, options?: RequestInit): Promise<Response> {
  const normalizedPath = path.startsWith('/') ? path : `/${path}`;
  const primaryUrl = `${API_PRIMARY}${normalizedPath}`;
  const fallbackUrl = `${API_FALLBACK}${normalizedPath}`;

  try {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), 5000);

    const res = await fetch(primaryUrl, {
      ...options,
      signal: controller.signal,
    });

    clearTimeout(timeoutId);

    // 2xx or 4xx: return to caller (client errors don't need failover)
    if (res.ok || res.status < 500) return res;

    // 5xx: server error, trigger failover
    throw new Error(`Primary API error: ${res.status}`);
  } catch (err) {
    if (isDev) throw err;

    console.warn('Primary API failed, trying fallback:', err);

    // Fallback with its own timeout
    const controller2 = new AbortController();
    const timeoutId2 = setTimeout(() => controller2.abort(), 5000);
    try {
      return await fetch(fallbackUrl, {
        ...options,
        signal: controller2.signal,
      });
    } finally {
      clearTimeout(timeoutId2);
    }
  }
}
```

## Wallet Hook Variant

The wallet integration hook has its own copy of the failover logic to keep it
self-contained (no import dependency on `config.ts`):

```typescript
// Inside useWallet.ts
const isDev = typeof window !== 'undefined' && window.location.hostname === 'localhost';
const API_PRIMARY = isDev ? 'https://api-dev.adavault.com/api/v1' : 'https://api.adavault.com/api/v1';
const API_FALLBACK = 'https://api2.adavault.com/api/v1';

async function apiFetchWithFailover(path: string): Promise<Response> {
  const primaryUrl = `${API_PRIMARY}${path}`;
  const fallbackUrl = `${API_FALLBACK}${path}`;

  try {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), 5000);

    const res = await fetch(primaryUrl, { signal: controller.signal });
    clearTimeout(timeoutId);

    if (res.ok) return res;
    throw new Error(`Primary API error: ${res.status}`);
  } catch (err) {
    if (isDev) throw err;
    console.warn('[Wallet] Primary API failed, trying fallback:', err);
    return fetch(fallbackUrl);
  }
}
```

Note the different dev detection: `window.location.hostname === 'localhost'` instead
of `import.meta.env.DEV`. This is because the wallet hook runs in the browser, where
`import.meta.env.DEV` may not be available depending on the build tool.

## Usage

```typescript
import { apiFetch } from '../config';

// Fetch pool data
const res = await apiFetch('/pools');
const data = await res.json();

// Fetch delegation status (proxied Koios call)
const res = await apiFetch(`/account/${stakeAddress}/delegation`);
if (!res.ok) {
  // 404 = not delegated, 400 = bad address, etc.
  return null;
}
const json = await res.json();
```

## Common Mistakes

### 1. No Timeout on Primary

```typescript
// WRONG: Waits 30-60 seconds for browser default timeout
const res = await fetch(primaryUrl, options);

// RIGHT: AbortController with 5s timeout
const controller = new AbortController();
const timeoutId = setTimeout(() => controller.abort(), 5000);
const res = await fetch(primaryUrl, { ...options, signal: controller.signal });
clearTimeout(timeoutId);
```

### 2. Failing Over on 4xx

```typescript
// WRONG: 404 triggers failover (fallback returns same 404)
if (!res.ok) throw new Error('Failed');

// RIGHT: Only failover on 5xx
if (res.ok || res.status < 500) return res;
throw new Error(`Server error: ${res.status}`);
```

### 3. Not Clearing Timeout

```typescript
// WRONG: Timeout fires after successful response
const timeoutId = setTimeout(() => controller.abort(), 5000);
const res = await fetch(url, { signal: controller.signal });
// Missing: clearTimeout(timeoutId)

// RIGHT: Always clear
clearTimeout(timeoutId);
```

### 4. Trying Fallback in Dev Mode

```typescript
// WRONG: api2 rejects localhost CORS, obscuring the real error
return fetch(fallbackUrl, options);

// RIGHT: Throw immediately in dev
if (isDev) throw err;
```

## See Also

- [Wallet Connect](wallet-connect.md) -- uses failover for delegation status fetch
- [API Resilience Reference](../reference/api-resilience.md) -- full architecture
