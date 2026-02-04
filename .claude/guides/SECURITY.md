# Security Practices Guide

## API Keys & Secrets Management

- **NEVER commit secrets** to version control (API keys, tokens, passwords, private keys)
- **Environment Variables:** Store all secrets in `.env.local` (never `.env`)
- **Git Ignore:** Ensure `.env.local`, `.env.*.local` are in `.gitignore`
- **Example Files:** Provide `.env.example` with placeholder values, no real secrets

```bash
# .env.example (commit this)
NEXT_PUBLIC_API_URL=https://api.example.com
API_SECRET_KEY=your_secret_key_here
DATABASE_URL=postgresql://localhost:5432/dbname

# .env.local (NEVER commit this)
NEXT_PUBLIC_API_URL=https://api.production.com
API_SECRET_KEY=sk_live_actual_secret_key
DATABASE_URL=postgresql://user:password@prod-db:5432/prod
```

---

## Environment Variable Naming

- **Public (client-side):** Prefix with `NEXT_PUBLIC_` (exposed to browser)
- **Private (server-side):** No prefix (only available in server components/API routes)
- **Validation:** Validate required env vars at startup

```typescript
// lib/env.ts
const requiredEnvVars = ['DATABASE_URL', 'API_SECRET_KEY'] as const;

for (const envVar of requiredEnvVars) {
  if (!process.env[envVar]) {
    throw new Error(`Missing required environment variable: ${envVar}`);
  }
}
```

---

## CORS (Cross-Origin Resource Sharing)

- **API Routes:** Configure CORS headers explicitly in Next.js API routes
- **Whitelist Origins:** Never use `Access-Control-Allow-Origin: *` in production
- **Credentials:** Only allow credentials for trusted origins

```typescript
// Example: Next.js API route with CORS
export async function GET(request: Request) {
  const origin = request.headers.get('origin');
  const allowedOrigins = ['https://yourdomain.com', 'https://app.yourdomain.com'];
  
  const headers = new Headers();
  if (origin && allowedOrigins.includes(origin)) {
    headers.set('Access-Control-Allow-Origin', origin);
    headers.set('Access-Control-Allow-Credentials', 'true');
  }
  
  return new Response(JSON.stringify(data), { headers });
}
```

---

## XSS (Cross-Site Scripting) Prevention

- **React Auto-Escapes:** Trust React's default escaping for variables in JSX
- **NEVER use `dangerouslySetInnerHTML`** without sanitization
- **Sanitize User Input:** Use DOMPurify for HTML content from users/external sources

```typescript
import DOMPurify from 'isomorphic-dompurify';

// Safe: React auto-escapes
<div>{userInput}</div>

// Unsafe without sanitization
<div dangerouslySetInnerHTML={{ __html: userInput }} />

// Safe with sanitization
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userInput) }} />
```

---

## Content Security Policy (CSP)

```typescript
// next.config.js
const cspHeader = `
  default-src 'self';
  script-src 'self' 'unsafe-eval' 'unsafe-inline';
  style-src 'self' 'unsafe-inline';
  img-src 'self' blob: data:;
  font-src 'self';
  object-src 'none';
  base-uri 'self';
  form-action 'self';
  frame-ancestors 'none';
  upgrade-insecure-requests;
`;

module.exports = {
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          { key: 'Content-Security-Policy', value: cspHeader.replace(/\n/g, '') }
        ],
      },
    ];
  },
};
```

---

## Authentication & Authorization

- **JWT Storage:** Store in httpOnly cookies, never localStorage
- **Session Management:** Use established libraries (NextAuth.js, Auth.js)
- **API Route Protection:** Verify authentication on every protected route
- **RBAC:** Implement role-based access control where needed

---

## Input Validation

- **Server-Side:** ALWAYS validate on server, even if client validates
- **Schema Validation:** Use Zod or similar for type-safe validation
- **Sanitize Inputs:** Remove/escape dangerous characters before database operations

```typescript
import { z } from 'zod';

const userSchema = z.object({
  email: z.string().email(),
  age: z.number().min(13).max(120),
  username: z.string().min(3).max(20).regex(/^[a-zA-Z0-9_]+$/),
});

// API route
const result = userSchema.safeParse(requestBody);
if (!result.success) {
  return new Response(JSON.stringify({ error: result.error }), { status: 400 });
}
```
