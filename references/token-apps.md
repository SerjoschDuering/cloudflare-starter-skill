# Token Apps — Building on the Token Platform

How to build apps that consume tokens from the **Token Service** (`references/token-service.md`).
Your app doesn't touch Stripe. It calls the Token Service to check balances and deduct tokens.

## How It Works

```
┌─────────────────────────────────────────────────────────────┐
│  User's browser                                             │
│                                                             │
│  1. User logs into Token Service (gets session cookie)      │
│  2. User opens your app (session cookie forwarded)          │
│  3. User triggers an action (e.g. "generate image")         │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  Your App (Worker)                                          │
│                                                             │
│  4. Verify user's identity (session or JWT from platform)   │
│  5. Call Token Service: POST /api/tokens/deduct             │
│     { userId, amount: 5, description: "Image gen" }        │
│  6. If 200 → do the work, return result                     │
│     If 402 → return "buy more tokens" error                 │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  Token Service (Worker)                                     │
│                                                             │
│  Checks balance → deducts → logs to ledger → returns        │
└─────────────────────────────────────────────────────────────┘
```

---

## Setup

### What You Need from the Platform Admin

When your app is registered in the Token Service, you'll receive:

| Value | What it is | Store as |
|-------|-----------|----------|
| API Key | `ts_a1b2c3d4...` — authenticates your app to the Token Service | `wrangler secret put TOKEN_SERVICE_API_KEY` |
| Service URL | `https://token-service.example.workers.dev` | `TOKEN_SERVICE_URL` env var in `wrangler.toml` |
| App ID | `image-gen` — your registered app slug | Returned automatically, no action needed |
| Cost Table | How many tokens each action costs | Define in your app's constants |

### Wrangler Config

```toml
# wrangler.toml
name = "my-cool-app"
compatibility_flags = ["nodejs_compat"]

[vars]
TOKEN_SERVICE_URL = "https://token-service.example.workers.dev"

[[d1_databases]]
binding = "DB"
database_name = "my-app-db"
database_id = "xxx"
```

### Secrets

```bash
wrangler secret put TOKEN_SERVICE_API_KEY    # ts_a1b2c3d4... from platform admin
```

---

## Token Service Client

A typed client that wraps the Token Service API. Drop this into any app.

```typescript
// src/lib/token-client.ts

export interface TokenDeductResult {
  balance: number
  ledgerEntryId: number
}

export interface TokenError {
  error: string
  available?: number
  required?: number
}

export class TokenClient {
  constructor(
    private serviceUrl: string,
    private apiKey: string
  ) {}

  private async request<T>(path: string, opts?: RequestInit): Promise<T> {
    const res = await fetch(`${this.serviceUrl}${path}`, {
      ...opts,
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${this.apiKey}`,
        ...opts?.headers,
      },
    })

    const body = await res.json() as any

    if (!res.ok) {
      const err = new TokenServiceError(
        body.error || 'Token service error',
        res.status,
        body.available,
        body.required
      )
      throw err
    }

    return body.data as T
  }

  // Check a user's token balance
  async getBalance(userId: string): Promise<number> {
    const result = await this.request<{ balance: number }>(`/api/tokens/balance/${userId}`)
    return result.balance
  }

  // Deduct tokens from a user's wallet
  async deduct(userId: string, amount: number, description?: string, referenceId?: string): Promise<TokenDeductResult> {
    return this.request<TokenDeductResult>('/api/tokens/deduct', {
      method: 'POST',
      body: JSON.stringify({ userId, amount, description, referenceId }),
    })
  }

  // Refund tokens to a user's wallet
  async refund(userId: string, amount: number, description?: string, referenceId?: string): Promise<{ balance: number }> {
    return this.request<{ balance: number }>('/api/tokens/refund', {
      method: 'POST',
      body: JSON.stringify({ userId, amount, description, referenceId }),
    })
  }
}

// Custom error with balance info
export class TokenServiceError extends Error {
  constructor(
    message: string,
    public statusCode: number,
    public available?: number,
    public required?: number,
  ) {
    super(message)
    this.name = 'TokenServiceError'
  }

  get isInsufficientTokens(): boolean {
    return this.statusCode === 402
  }
}
```

---

## Token Middleware — Deduct Before Processing

Wrap any route to deduct tokens BEFORE doing the work. If the user doesn't
have enough tokens, they get a 402 response immediately.

```typescript
// src/middleware/require-tokens.ts
import { TokenClient, TokenServiceError } from '../lib/token-client'

// Middleware factory — deduct a fixed token cost per request
export function requireTokens(cost: number, description?: string) {
  return async (c: any, next: any) => {
    const userId = c.get('userId')
    if (!userId) return c.json({ error: 'Unauthorized' }, 401)

    const client = new TokenClient(c.env.TOKEN_SERVICE_URL, c.env.TOKEN_SERVICE_API_KEY)

    try {
      const result = await client.deduct(userId, cost, description)
      c.set('tokenBalance', result.balance)
      c.set('tokenLedgerEntryId', result.ledgerEntryId)
      await next()
    } catch (err) {
      if (err instanceof TokenServiceError && err.isInsufficientTokens) {
        return c.json({
          error: 'Insufficient tokens',
          required: err.required,
          available: err.available,
          buyTokensUrl: `${c.env.TOKEN_SERVICE_URL}/tokens`,  // where to buy more
        }, 402)
      }
      throw err
    }
  }
}

// Usage in routes:
//
// import { requireTokens } from '../middleware/require-tokens'
//
// app.post('/generate', requireTokens(5, 'Image generation'), async (c) => {
//   // This only runs if the user had 5 tokens (already deducted)
//   const result = await generateImage(...)
//   return c.json({ data: result })
// })
```

---

## Dynamic Token Costs

Some actions cost different amounts. Calculate the cost inside the handler
and call the Token Client directly.

```typescript
// src/domains/generation/generation.routes.ts
import { Hono } from 'hono'
import { zValidator } from '@hono/zod-validator'
import { z } from 'zod'
import { TokenClient, TokenServiceError } from '../../lib/token-client'

// Define your cost table
const COSTS = {
  'image-256': 2,
  'image-512': 5,
  'image-1024': 12,
  'video-short': 25,
  'video-long': 50,
} as const

type Env = {
  Bindings: { TOKEN_SERVICE_URL: string; TOKEN_SERVICE_API_KEY: string }
  Variables: { userId: string }
}
const app = new Hono<Env>()

app.post('/generate', zValidator('json', z.object({
  type: z.enum(['image-256', 'image-512', 'image-1024', 'video-short', 'video-long']),
  prompt: z.string().min(1),
})), async (c) => {
  const { type, prompt } = c.req.valid('json')
  const userId = c.get('userId')
  const cost = COSTS[type]

  const client = new TokenClient(c.env.TOKEN_SERVICE_URL, c.env.TOKEN_SERVICE_API_KEY)

  // 1. Deduct tokens
  let deductResult
  try {
    deductResult = await client.deduct(userId, cost, `${type} generation`)
  } catch (err) {
    if (err instanceof TokenServiceError && err.isInsufficientTokens) {
      return c.json({
        error: 'Insufficient tokens',
        required: cost,
        available: err.available,
      }, 402)
    }
    throw err
  }

  // 2. Do the expensive work
  try {
    const result = await doGeneration(type, prompt)
    return c.json({
      data: {
        ...result,
        tokensUsed: cost,
        remainingBalance: deductResult.balance,
      }
    })
  } catch (err) {
    // 3. If the work fails, REFUND the tokens
    await client.refund(userId, cost, `Refund: ${type} generation failed`)
    return c.json({ error: 'Generation failed — tokens refunded' }, 500)
  }
})

export { app as generationRouter }
```

---

## Cost Estimation Endpoint

Let users preview how many tokens an action will cost BEFORE they commit.

```typescript
// GET /costs — show all action costs
app.get('/costs', (c) => {
  return c.json({
    data: Object.entries(COSTS).map(([action, tokens]) => ({
      action,
      tokens,
      estimatedUsd: `$${(tokens * 0.10).toFixed(2)}`,  // assuming $0.10/token
    }))
  })
})

// POST /estimate — estimate cost for a specific request
app.post('/estimate', zValidator('json', z.object({
  type: z.enum(['image-256', 'image-512', 'image-1024', 'video-short', 'video-long']),
})), async (c) => {
  const { type } = c.req.valid('json')
  const userId = c.get('userId')
  const cost = COSTS[type]

  const client = new TokenClient(c.env.TOKEN_SERVICE_URL, c.env.TOKEN_SERVICE_API_KEY)
  const balance = await client.getBalance(userId)

  return c.json({
    data: {
      action: type,
      cost,
      currentBalance: balance,
      canAfford: balance >= cost,
      remainingAfter: Math.max(0, balance - cost),
    }
  })
})
```

---

## User Identity — How Apps Know Who the User Is

Three options, from simplest to most flexible:

### Option A: Shared Auth (Recommended for Platform Apps)

If your app is part of the same platform, share the Better Auth session.
Users log in once at the Token Service and the cookie works across subdomains.

```typescript
// Your app verifies the session against the Token Service's auth
import { createAuth } from './auth'  // same Better Auth config, same DB

export function authMiddleware() {
  return async (c: any, next: any) => {
    const auth = createAuth(c.env)
    const session = await auth.api.getSession({ headers: c.req.raw.headers })
    if (!session) return c.json({ error: 'Unauthorized' }, 401)
    c.set('userId', session.user.id)
    await next()
  }
}
```

> **Cookie sharing requires same domain.** Deploy the Token Service at
> `api.yourplatform.com` and apps at `app1.yourplatform.com`, `app2.yourplatform.com`.
> Set the cookie domain to `.yourplatform.com` in Better Auth config.

### Option B: Token Forwarding (Third-Party Apps)

The user logs in via the Token Service, gets a short-lived JWT, and sends it
to your app. Your app verifies the JWT without calling the Token Service.

```typescript
// Token Service issues a JWT after login
import { SignJWT } from 'jose'

const jwt = await new SignJWT({ sub: user.id, email: user.email })
  .setProtectedHeader({ alg: 'HS256' })
  .setExpirationTime('1h')
  .sign(new TextEncoder().encode(env.JWT_SECRET))
```

```typescript
// Your app verifies the JWT
import { jwtVerify } from 'jose'

export function jwtAuthMiddleware() {
  return async (c: any, next: any) => {
    const token = c.req.header('Authorization')?.replace('Bearer ', '')
    if (!token) return c.json({ error: 'Unauthorized' }, 401)

    try {
      const secret = new TextEncoder().encode(c.env.PLATFORM_JWT_SECRET)
      const { payload } = await jwtVerify(token, secret)
      c.set('userId', payload.sub as string)
      await next()
    } catch {
      return c.json({ error: 'Invalid or expired token' }, 401)
    }
  }
}
```

### Option C: User ID Passthrough (Internal/Trusted Apps Only)

For internal apps where the Token Service trusts the calling app completely.
The app sends the user ID directly to the Token Service — no user auth needed
on the app side (the API key is the trust boundary).

```typescript
// App just passes userId from its own auth to the Token Client
const result = await client.deduct(userId, 5, 'Image generation')
```

> **Security note:** Only use this for apps you fully control. The API key gives
> the app permission to deduct tokens from ANY user — the app is responsible for
> verifying the user's identity before calling deduct.

---

## Frontend — Token Balance Widget

A reusable React component that shows the user's token balance. Works in any app
that connects to the Token Service.

```typescript
// client/src/components/TokenBalance.tsx
import { useQuery } from '@tanstack/react-query'
import { apiFetch } from '@/lib/api'

export function TokenBalance() {
  const { data, isLoading } = useQuery({
    queryKey: ['tokenBalance'],
    queryFn: () => apiFetch<{ data: { balance: number } }>('/tokens/balance'),
    refetchInterval: 30_000,  // refresh every 30s
  })

  if (isLoading) return <span>...</span>
  const balance = data?.data?.balance ?? 0

  return (
    <div style={{ display: 'flex', alignItems: 'center', gap: '0.5rem' }}>
      <span>{balance} tokens</span>
      {balance < 10 && (
        <a href={`${import.meta.env.VITE_TOKEN_SERVICE_URL}/tokens`}
           style={{ color: '#f59e0b', fontSize: '0.875rem' }}>
          Buy more
        </a>
      )}
    </div>
  )
}
```

### Handling 402 Responses

```typescript
// client/src/lib/api.ts — enhanced error handling
export async function apiFetch<T>(path: string, options?: RequestInit): Promise<T> {
  const res = await fetch(`${baseURL}${path}`, {
    headers: { 'Content-Type': 'application/json' },
    ...options,
  })

  if (res.status === 402) {
    const body = await res.json() as any
    // Show "buy tokens" UI or redirect
    throw new InsufficientTokensError(body.available, body.required, body.buyTokensUrl)
  }

  if (!res.ok) throw new Error((await res.json()).error || res.statusText)
  return res.json()
}

export class InsufficientTokensError extends Error {
  constructor(
    public available: number,
    public required: number,
    public buyUrl: string,
  ) {
    super(`Need ${required} tokens, have ${available}`)
    this.name = 'InsufficientTokensError'
  }
}
```

```typescript
// client/src/components/GenerateButton.tsx
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { apiFetch, InsufficientTokensError } from '@/lib/api'

export function GenerateButton({ type, prompt }: { type: string; prompt: string }) {
  const qc = useQueryClient()

  const generate = useMutation({
    mutationFn: () => apiFetch('/generate', {
      method: 'POST',
      body: JSON.stringify({ type, prompt }),
    }),
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: ['tokenBalance'] })  // refresh balance
    },
    onError: (err) => {
      if (err instanceof InsufficientTokensError) {
        // Show upgrade prompt instead of generic error
        window.open(err.buyUrl, '_blank')
      }
    },
  })

  return (
    <button onClick={() => generate.mutate()} disabled={generate.isPending}>
      {generate.isPending ? 'Generating...' : `Generate (${COSTS[type]} tokens)`}
    </button>
  )
}
```

---

## Complete App Example — Folder Structure

```
my-cool-app/
├── src/
│   ├── index.ts                          # Hono entry point
│   ├── lib/
│   │   ├── token-client.ts               # Token Service client (copy from above)
│   │   └── api.ts                        # Generic fetch helper
│   ├── middleware/
│   │   ├── auth.ts                       # User auth (shared session or JWT)
│   │   └── require-tokens.ts            # Token deduction middleware
│   └── domains/
│       └── generation/
│           ├── generation.routes.ts      # App-specific endpoints
│           └── generation.schema.ts      # App-specific DB tables (if any)
├── client/
│   └── src/
│       ├── components/
│       │   ├── TokenBalance.tsx          # Balance widget
│       │   └── GenerateButton.tsx        # Token-aware action button
│       └── lib/
│           └── api.ts                    # Enhanced fetch with 402 handling
├── wrangler.toml
└── package.json
```

---

## Checklist

- [ ] Get API key + service URL from platform admin
- [ ] Store `TOKEN_SERVICE_API_KEY` as secret: `wrangler secret put TOKEN_SERVICE_API_KEY`
- [ ] Add `TOKEN_SERVICE_URL` to `wrangler.toml` vars
- [ ] Drop `token-client.ts` into your app
- [ ] Add `require-tokens` middleware to paid routes
- [ ] Define your cost table in constants
- [ ] Handle 402 responses in frontend (show "buy more tokens")
- [ ] Add refund logic for failed operations
- [ ] Test with the Token Service's test environment

---

## Official Documentation Links

| Topic | URL |
|-------|-----|
| Token Service setup | `references/token-service.md` (this repo) |
| SaaS billing (single product) | `references/saas-billing.md` (this repo) |
| Workers Service Bindings | https://developers.cloudflare.com/workers/runtime-apis/bindings/service-bindings/ |
| Better Auth | https://www.better-auth.com/docs |
| jose (JWT) | https://github.com/panva/jose |
