# API Integration Guide

## General Patterns

1. **Error Handling:** Wrap all API calls in try-catch with specific error types
2. **Timeouts:** Set reasonable timeouts (default: 10s for most APIs)
3. **Retry Logic:** Implement exponential backoff for transient failures
4. **Rate Limiting:** Respect API rate limits, implement client-side throttling

---

## API Client Structure

```typescript
// lib/api/client.ts
export class APIClient {
  private baseURL: string;
  private timeout: number;
  
  constructor(baseURL: string, timeout = 10000) {
    this.baseURL = baseURL;
    this.timeout = timeout;
  }
  
  async request<T>(endpoint: string, options?: RequestInit): Promise<T> {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), this.timeout);
    
    try {
      const response = await fetch(`${this.baseURL}${endpoint}`, {
        ...options,
        signal: controller.signal,
        headers: {
          'Content-Type': 'application/json',
          ...options?.headers,
        },
      });
      
      clearTimeout(timeoutId);
      
      if (!response.ok) {
        throw new APIError(response.status, await response.text());
      }
      
      return await response.json();
    } catch (error) {
      clearTimeout(timeoutId);
      
      if (error instanceof DOMException && error.name === 'AbortError') {
        throw new APIError(408, 'Request timeout');
      }
      throw error;
    }
  }
}

class APIError extends Error {
  constructor(public status: number, message: string) {
    super(message);
    this.name = 'APIError';
  }
}
```

---

## Rate Limiting & Throttling

```typescript
// lib/utils/throttle.ts
export class RateLimiter {
  private tokens: number;
  private lastRefill: number;
  
  constructor(
    private maxTokens: number,
    private refillRate: number // tokens per second
  ) {
    this.tokens = maxTokens;
    this.lastRefill = Date.now();
  }
  
  async acquire(): Promise<void> {
    this.refill();
    
    if (this.tokens < 1) {
      const waitTime = (1 - this.tokens) / this.refillRate * 1000;
      await new Promise(resolve => setTimeout(resolve, waitTime));
      this.refill();
    }
    
    this.tokens -= 1;
  }
  
  private refill(): void {
    const now = Date.now();
    const elapsed = (now - this.lastRefill) / 1000;
    this.tokens = Math.min(this.maxTokens, this.tokens + elapsed * this.refillRate);
    this.lastRefill = now;
  }
}
```

---

## Retry Logic

```typescript
// lib/utils/retry.ts
export async function withRetry<T>(
  fn: () => Promise<T>,
  options = {
    maxRetries: 3,
    baseDelay: 1000,
    maxDelay: 10000,
    retryableStatuses: [408, 429, 500, 502, 503, 504],
  }
): Promise<T> {
  let lastError: Error;
  
  for (let attempt = 0; attempt <= options.maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;
      
      // Don't retry on non-retryable errors
      if (error instanceof APIError && !options.retryableStatuses.includes(error.status)) {
        throw error;
      }
      
      if (attempt < options.maxRetries) {
        const delay = Math.min(
          options.baseDelay * Math.pow(2, attempt),
          options.maxDelay
        );
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
  }
  
  throw lastError!;
}
```

---

## Response Caching

- **Use SWR or React Query** for client-side caching and revalidation
- **Cache-Control Headers:** Set appropriate cache headers on API routes
- **Stale-While-Revalidate:** Serve cached data while fetching updates

```typescript
// Example with SWR
import useSWR from 'swr';

const fetcher = (url: string) => fetch(url).then(r => r.json());

function Component() {
  const { data, error, isLoading } = useSWR('/api/data', fetcher, {
    revalidateOnFocus: false,
    revalidateOnReconnect: false,
    refreshInterval: 60000, // Refresh every 60s
  });
  
  if (error) return <div>Failed to load</div>;
  if (isLoading) return <div>Loading...</div>;
  return <div>{data}</div>;
}
```
