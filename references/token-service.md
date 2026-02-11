# Token Service — Central Billing Platform

A standalone Cloudflare Worker that acts as the **billing backbone** for multiple apps.
Users sign up once, buy tokens via Stripe, and spend them across any app in your ecosystem.

Think: OpenAI credits (one wallet, used by ChatGPT + API + DALL-E) or Make.com operations.

> **Building a single product instead?** See `references/saas-billing.md` for self-contained
> Stripe subscriptions.
>
> **Building apps that consume tokens?** See `references/token-apps.md` for the consumer pattern.

## How It Works

```
┌──────────────────────────────────────────────────────────┐
│                    Token Service (Worker)                  │
│                                                            │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐  │
│  │ Auth        │  │ Stripe      │  │ Token Wallet     │  │
│  │ (signup/    │  │ (checkout,  │  │ (balance, deduct,│  │
│  │  login)     │  │  webhooks)  │  │  top-up, logs)   │  │
│  └─────────────┘  └─────────────┘  └──────────────────┘  │
│                                                            │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐  │
│  │ App Registry│  │ API Keys    │  │ Usage Dashboard  │  │
│  │ (registered │  │ (per-app    │  │ (spend history,  │  │
│  │  apps)      │  │  secrets)   │  │  per-app stats)  │  │
│  └─────────────┘  └─────────────┘  └──────────────────┘  │
└──────────────────────────────────────────────────────────┘
          │                                     ▲
          │  Apps call the Token Service API     │
          ▼                                     │
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   App A      │  │   App B      │  │   App C      │
│  (2 tokens/  │  │  (10 tokens/ │  │  (1 token/   │
│   request)   │  │   request)   │  │   request)   │
└──────────────┘  └──────────────┘  └──────────────┘
```

## What the Token Service Owns

| Responsibility | Endpoint | Who calls it |
|---------------|----------|-------------|
| User signup/login | `POST /auth/signup`, `POST /auth/login` | Users (via frontend) |
| Buy tokens (Stripe) | `POST /checkout` | Users (via frontend) |
| Check balance | `GET /tokens/balance` | Apps + Users |
| Deduct tokens | `POST /tokens/deduct` | Apps (server-to-server) |
| Refund tokens | `POST /tokens/refund` | Apps (server-to-server) |
| Usage history | `GET /tokens/usage` | Users (via frontend) |
| Register an app | `POST /apps` | Platform admin |
| Stripe webhooks | `POST /webhooks/stripe` | Stripe |

---

## Setup

### Install

```bash
npm install hono drizzle-orm drizzle-zod @hono/zod-validator zod stripe better-auth jose
npm install -D wrangler @cloudflare/workers-types drizzle-kit typescript
```

### Wrangler Config

```toml
# wrangler.toml
name = "token-service"
compatibility_flags = ["nodejs_compat"]

[[d1_databases]]
binding = "DB"
database_name = "token-service-db"
database_id = "xxx"
```

### Secrets

```bash
wrangler secret put STRIPE_SECRET_KEY
wrangler secret put STRIPE_WEBHOOK_SECRET
wrangler secret put BETTER_AUTH_SECRET       # openssl rand -base64 32
wrangler secret put BETTER_AUTH_URL          # https://token-service.your-name.workers.dev
wrangler secret put SERVICE_SIGNING_KEY      # openssl rand -base64 32 — for app API key hashing
```

---

## Database Schema

```typescript
// src/domains/tokens/tokens.schema.ts
import { sqliteTable, integer, text, real, index } from 'drizzle-orm/sqlite-core'
import { createSelectSchema } from 'drizzle-zod'
import { z } from 'zod'

// Token balances — one row per user
export const tokenBalances = sqliteTable('token_balances', {
  userId: text('user_id').primaryKey(),
  balance: integer('balance').notNull().default(0),       // tokens available
  lifetimePurchased: integer('lifetime_purchased').notNull().default(0),
  lifetimeSpent: integer('lifetime_spent').notNull().default(0),
  updatedAt: integer('updated_at', { mode: 'timestamp' }).$defaultFn(() => new Date()),
})

// Every token movement — purchases, deductions, refunds
export const tokenLedger = sqliteTable('token_ledger', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  userId: text('user_id').notNull(),
  amount: integer('amount').notNull(),                    // positive = credit, negative = debit
  type: text('type').notNull(),                           // 'purchase' | 'deduct' | 'refund' | 'bonus'
  appId: text('app_id'),                                  // which app deducted (null for purchases)
  description: text('description'),                       // human-readable: "Image generation", "API call"
  referenceId: text('reference_id'),                      // external ID (Stripe payment ID, app request ID)
  createdAt: integer('created_at', { mode: 'timestamp' }).$defaultFn(() => new Date()),
}, (table) => [
  index('idx_ledger_user').on(table.userId),
  index('idx_ledger_app').on(table.appId),
  index('idx_ledger_created').on(table.createdAt),
])

// Token packs available for purchase
export const tokenPacks = sqliteTable('token_packs', {
  id: text('id').primaryKey(),                            // 'starter', 'pro', 'mega'
  name: text('name').notNull(),
  tokenAmount: integer('token_amount').notNull(),         // how many tokens
  priceInCents: integer('price_in_cents').notNull(),      // USD cents
  stripePriceId: text('stripe_price_id').notNull(),       // Stripe price ID
  active: integer('active', { mode: 'boolean' }).default(true),
})

// Registered apps that can deduct tokens
export const apps = sqliteTable('apps', {
  id: text('id').primaryKey(),                            // slug: 'image-gen', 'chat-bot'
  name: text('name').notNull(),
  description: text('description'),
  apiKeyHash: text('api_key_hash').notNull(),             // hashed API key for server-to-server auth
  defaultCostPerRequest: integer('default_cost_per_request').notNull().default(1),
  active: integer('active', { mode: 'boolean' }).default(true),
  createdAt: integer('created_at', { mode: 'timestamp' }).$defaultFn(() => new Date()),
})

// Stripe customer mapping
export const stripeCustomers = sqliteTable('stripe_customers', {
  userId: text('user_id').primaryKey(),
  stripeCustomerId: text('stripe_customer_id').notNull().unique(),
  createdAt: integer('created_at', { mode: 'timestamp' }).$defaultFn(() => new Date()),
})

export const selectTokenBalanceSchema = createSelectSchema(tokenBalances)
export const selectLedgerEntrySchema = createSelectSchema(tokenLedger)
export type TokenBalance = z.infer<typeof selectTokenBalanceSchema>
export type LedgerEntry = z.infer<typeof selectLedgerEntrySchema>
```

---

## Token Wallet — Core Logic

This is the heart of the service. All balance changes go through these functions
so the ledger stays consistent.

```typescript
// src/domains/tokens/tokens.service.ts
import { eq, desc, and, sql } from 'drizzle-orm'
import { tokenBalances, tokenLedger } from './tokens.schema'

type Db = ReturnType<typeof import('../../db').createDb>

// Check balance
export async function getBalance(db: Db, userId: string): Promise<number> {
  const row = await db.select().from(tokenBalances).where(eq(tokenBalances.userId, userId)).get()
  return row?.balance ?? 0
}

// Credit tokens (purchase, refund, bonus)
export async function creditTokens(
  db: Db,
  userId: string,
  amount: number,
  type: 'purchase' | 'refund' | 'bonus',
  opts: { appId?: string; description?: string; referenceId?: string } = {}
): Promise<number> {
  // Upsert balance + insert ledger entry in a batch (atomic)
  const results = await db.batch([
    db.insert(tokenBalances).values({
      userId,
      balance: amount,
      lifetimePurchased: type === 'purchase' ? amount : 0,
      updatedAt: new Date(),
    }).onConflictDoUpdate({
      target: tokenBalances.userId,
      set: {
        balance: sql`${tokenBalances.balance} + ${amount}`,
        lifetimePurchased: type === 'purchase'
          ? sql`${tokenBalances.lifetimePurchased} + ${amount}`
          : tokenBalances.lifetimePurchased,
        updatedAt: new Date(),
      },
    }).returning(),

    db.insert(tokenLedger).values({
      userId,
      amount,      // positive
      type,
      appId: opts.appId ?? null,
      description: opts.description ?? null,
      referenceId: opts.referenceId ?? null,
    }),
  ])

  return results[0][0].balance
}

// Deduct tokens — returns new balance or throws if insufficient
export async function deductTokens(
  db: Db,
  userId: string,
  amount: number,
  appId: string,
  opts: { description?: string; referenceId?: string } = {}
): Promise<{ balance: number; ledgerEntryId: number }> {
  // Check balance first
  const current = await getBalance(db, userId)
  if (current < amount) {
    throw new InsufficientTokensError(current, amount)
  }

  // Deduct + log in a batch
  const results = await db.batch([
    db.update(tokenBalances).set({
      balance: sql`${tokenBalances.balance} - ${amount}`,
      lifetimeSpent: sql`${tokenBalances.lifetimeSpent} + ${amount}`,
      updatedAt: new Date(),
    }).where(
      and(
        eq(tokenBalances.userId, userId),
        // Double-check balance hasn't changed (optimistic lock)
        sql`${tokenBalances.balance} >= ${amount}`
      )
    ).returning(),

    db.insert(tokenLedger).values({
      userId,
      amount: -amount,   // negative = debit
      type: 'deduct',
      appId,
      description: opts.description ?? null,
      referenceId: opts.referenceId ?? null,
    }).returning(),
  ])

  // If no rows updated, balance changed between check and update
  if (!results[0].length) {
    throw new InsufficientTokensError(current, amount)
  }

  return {
    balance: results[0][0].balance,
    ledgerEntryId: results[1][0].id,
  }
}

// Get usage history
export async function getUsageHistory(
  db: Db,
  userId: string,
  opts: { limit?: number; offset?: number; appId?: string } = {}
) {
  const conditions = [eq(tokenLedger.userId, userId)]
  if (opts.appId) conditions.push(eq(tokenLedger.appId, opts.appId))

  return db.select().from(tokenLedger)
    .where(and(...conditions))
    .orderBy(desc(tokenLedger.createdAt))
    .limit(opts.limit ?? 50)
    .offset(opts.offset ?? 0)
    .all()
}

// Custom error for insufficient balance
export class InsufficientTokensError extends Error {
  constructor(public available: number, public required: number) {
    super(`Insufficient tokens: need ${required}, have ${available}`)
    this.name = 'InsufficientTokensError'
  }
}
```

---

## API Routes — User-Facing

```typescript
// src/domains/tokens/tokens.routes.ts
import { Hono } from 'hono'
import { zValidator } from '@hono/zod-validator'
import { z } from 'zod'
import { createDb } from '../../db'
import { createStripe } from '../../lib/stripe'
import { getBalance, getUsageHistory } from './tokens.service'
import { tokenPacks, stripeCustomers } from './tokens.schema'
import { eq } from 'drizzle-orm'

type Env = { Bindings: { DB: D1Database; STRIPE_SECRET_KEY: string }; Variables: { userId: string } }
const app = new Hono<Env>()

// GET /tokens/balance — check your balance
app.get('/balance', async (c) => {
  const db = createDb(c.env.DB)
  const balance = await getBalance(db, c.get('userId'))
  return c.json({ data: { balance } })
})

// GET /tokens/usage — spending history
app.get('/usage', async (c) => {
  const db = createDb(c.env.DB)
  const limit = Number(c.req.query('limit') || 50)
  const offset = Number(c.req.query('offset') || 0)
  const appId = c.req.query('app') || undefined

  const entries = await getUsageHistory(db, c.get('userId'), { limit, offset, appId })
  return c.json({ data: entries })
})

// GET /tokens/packs — available token packs
app.get('/packs', async (c) => {
  const db = createDb(c.env.DB)
  const packs = await db.select().from(tokenPacks).where(eq(tokenPacks.active, true)).all()
  return c.json({ data: packs })
})

// POST /tokens/buy — buy a token pack via Stripe Checkout
app.post('/buy', zValidator('json', z.object({ packId: z.string() })), async (c) => {
  const { packId } = c.req.valid('json')
  const userId = c.get('userId')
  const db = createDb(c.env.DB)
  const stripe = createStripe(c.env.STRIPE_SECRET_KEY)

  // Look up the pack
  const pack = await db.select().from(tokenPacks).where(eq(tokenPacks.id, packId)).get()
  if (!pack) return c.json({ error: 'Pack not found' }, 404)

  // Get or create Stripe customer
  let customer = await db.select().from(stripeCustomers).where(eq(stripeCustomers.userId, userId)).get()
  if (!customer) {
    const sc = await stripe.customers.create({ metadata: { userId } })
    customer = (await db.insert(stripeCustomers).values({
      userId, stripeCustomerId: sc.id,
    }).returning())[0]
  }

  // Create checkout session (one-time payment for token pack)
  const session = await stripe.checkout.sessions.create({
    customer: customer.stripeCustomerId,
    mode: 'payment',
    line_items: [{ price: pack.stripePriceId, quantity: 1 }],
    success_url: `${c.req.header('origin')}/tokens?purchased=${packId}`,
    cancel_url: `${c.req.header('origin')}/tokens?canceled=true`,
    metadata: { userId, packId, tokenAmount: pack.tokenAmount.toString() },
  })

  return c.json({ data: { url: session.url } })
})

export { app as tokenRoutes }
```

---

## API Routes — App-to-Service (Server-to-Server)

Apps call these endpoints to deduct tokens on behalf of users.
Authenticated by **API key** (not user session).

```typescript
// src/domains/tokens/tokens.app-routes.ts
import { Hono } from 'hono'
import { zValidator } from '@hono/zod-validator'
import { z } from 'zod'
import { createDb } from '../../db'
import { deductTokens, creditTokens, getBalance, InsufficientTokensError } from './tokens.service'
import { verifyAppApiKey } from '../../middleware/app-auth'

type Env = { Bindings: { DB: D1Database; SERVICE_SIGNING_KEY: string } }
const app = new Hono<Env>()

// All routes require valid app API key
app.use('/*', verifyAppApiKey())

// POST /api/tokens/deduct — app deducts tokens from a user
app.post('/deduct', zValidator('json', z.object({
  userId: z.string(),
  amount: z.number().int().positive(),
  description: z.string().optional(),
  referenceId: z.string().optional(),    // your app's request/transaction ID (for idempotency)
})), async (c) => {
  const { userId, amount, description, referenceId } = c.req.valid('json')
  const appId = c.get('appId') as string
  const db = createDb(c.env.DB)

  try {
    const result = await deductTokens(db, userId, amount, appId, { description, referenceId })
    return c.json({ data: result })
  } catch (err) {
    if (err instanceof InsufficientTokensError) {
      return c.json({
        error: 'Insufficient tokens',
        available: err.available,
        required: err.required,
      }, 402)   // 402 Payment Required
    }
    throw err
  }
})

// POST /api/tokens/refund — app refunds tokens to a user
app.post('/refund', zValidator('json', z.object({
  userId: z.string(),
  amount: z.number().int().positive(),
  description: z.string().optional(),
  referenceId: z.string().optional(),
})), async (c) => {
  const { userId, amount, description, referenceId } = c.req.valid('json')
  const appId = c.get('appId') as string
  const db = createDb(c.env.DB)

  const balance = await creditTokens(db, userId, amount, 'refund', {
    appId, description, referenceId,
  })
  return c.json({ data: { balance } })
})

// GET /api/tokens/balance/:userId — app checks a user's balance
app.get('/balance/:userId', async (c) => {
  const db = createDb(c.env.DB)
  const balance = await getBalance(db, c.req.param('userId'))
  return c.json({ data: { balance } })
})

export { app as tokenAppRoutes }
```

---

## App API Key Authentication

Apps authenticate to the Token Service using an API key in the `Authorization` header.

```typescript
// src/middleware/app-auth.ts
import { eq } from 'drizzle-orm'
import { createDb } from '../db'
import { apps } from '../domains/tokens/tokens.schema'

// Verify app API key and set appId in context
export function verifyAppApiKey() {
  return async (c: any, next: any) => {
    const authHeader = c.req.header('Authorization')
    if (!authHeader?.startsWith('Bearer ')) {
      return c.json({ error: 'Missing API key' }, 401)
    }

    const apiKey = authHeader.slice(7)
    const keyHash = await hashApiKey(apiKey, c.env.SERVICE_SIGNING_KEY)
    const db = createDb(c.env.DB)

    const app = await db.select().from(apps)
      .where(eq(apps.apiKeyHash, keyHash))
      .get()

    if (!app || !app.active) {
      return c.json({ error: 'Invalid API key' }, 401)
    }

    c.set('appId', app.id)
    c.set('appName', app.name)
    await next()
  }
}

// Hash API key using HMAC-SHA256
async function hashApiKey(key: string, signingKey: string): Promise<string> {
  const encoder = new TextEncoder()
  const cryptoKey = await crypto.subtle.importKey(
    'raw', encoder.encode(signingKey),
    { name: 'HMAC', hash: 'SHA-256' }, false, ['sign']
  )
  const signature = await crypto.subtle.sign('HMAC', cryptoKey, encoder.encode(key))
  return Array.from(new Uint8Array(signature)).map(b => b.toString(16).padStart(2, '0')).join('')
}

// Helper to generate a new API key (call this when registering an app)
export function generateApiKey(): string {
  const bytes = new Uint8Array(32)
  crypto.getRandomValues(bytes)
  return 'ts_' + Array.from(bytes).map(b => b.toString(16).padStart(2, '0')).join('')
}
```

---

## Stripe Webhook — Credit Tokens on Payment

```typescript
// src/domains/tokens/tokens.webhooks.ts
import Stripe from 'stripe'
import { createStripe } from '../../lib/stripe'
import { createDb } from '../../db'
import { creditTokens } from './tokens.service'

const webCrypto = Stripe.createSubtleCryptoProvider()

export async function handleTokenWebhook(request: Request, env: any) {
  const stripe = createStripe(env.STRIPE_SECRET_KEY)
  const body = await request.text()
  const sig = request.headers.get('stripe-signature')
  if (!sig) return new Response('Missing signature', { status: 400 })

  let event: Stripe.Event
  try {
    event = await stripe.webhooks.constructEventAsync(
      body, sig, env.STRIPE_WEBHOOK_SECRET, undefined, webCrypto
    )
  } catch {
    return new Response('Invalid signature', { status: 400 })
  }

  const db = createDb(env.DB)

  if (event.type === 'checkout.session.completed') {
    const session = event.data.object as Stripe.Checkout.Session

    // Only process token pack purchases
    const userId = session.metadata?.userId
    const tokenAmount = parseInt(session.metadata?.tokenAmount || '0', 10)
    const packId = session.metadata?.packId

    if (!userId || !tokenAmount) {
      return new Response('Missing metadata', { status: 400 })
    }

    // Credit tokens to user's wallet
    await creditTokens(db, userId, tokenAmount, 'purchase', {
      description: `Purchased ${packId} pack (${tokenAmount} tokens)`,
      referenceId: session.payment_intent as string,
    })
  }

  return new Response(JSON.stringify({ received: true }))
}
```

---

## Main Entry Point

```typescript
// src/index.ts
import { Hono } from 'hono'
import { cors } from 'hono/cors'
import { createAuth } from './auth'
import { authMiddleware } from './middleware/auth'
import { tokenRoutes } from './domains/tokens/tokens.routes'
import { tokenAppRoutes } from './domains/tokens/tokens.app-routes'
import { handleTokenWebhook } from './domains/tokens/tokens.webhooks'

type Bindings = {
  DB: D1Database
  STRIPE_SECRET_KEY: string
  STRIPE_WEBHOOK_SECRET: string
  BETTER_AUTH_SECRET: string
  BETTER_AUTH_URL: string
  SERVICE_SIGNING_KEY: string
}

const app = new Hono<{ Bindings: Bindings }>()

app.use('/*', cors({
  origin: ['http://localhost:3001', 'https://your-platform.com'],
  credentials: true,
}))

// Public: Stripe webhooks (no auth)
app.post('/webhooks/stripe', (c) => handleTokenWebhook(c.req.raw, c.env))

// Public: Better Auth routes
app.on(['GET', 'POST'], '/auth/*', (c) => {
  const auth = createAuth(c.env)
  return auth.handler(c.req.raw)
})

// App-to-service API (API key auth, NOT user auth)
app.route('/api/tokens', tokenAppRoutes)

// User-facing routes (session auth)
app.use('/tokens/*', authMiddleware())
app.route('/tokens', tokenRoutes)

export default app
```

---

## Token Pack Configuration

### Create Products in Stripe

```bash
# Create the product
stripe products create --name="Token Pack" --description="Credits for the platform"

# Create prices for each pack
stripe prices create --product=prod_xxx --unit-amount=500   --currency=usd   # $5 = Starter
stripe prices create --product=prod_xxx --unit-amount=2000  --currency=usd   # $20 = Pro
stripe prices create --product=prod_xxx --unit-amount=10000 --currency=usd   # $100 = Mega
```

### Seed the Database

```sql
-- Seed token packs (adjust price IDs after creating in Stripe)
INSERT INTO token_packs (id, name, token_amount, price_in_cents, stripe_price_id) VALUES
  ('starter', 'Starter Pack',  50,    500,   'price_xxx_starter'),
  ('pro',     'Pro Pack',      250,   2000,  'price_xxx_pro'),
  ('mega',    'Mega Pack',     1500,  10000, 'price_xxx_mega');

-- 1 token ≈ $0.10 (starter), $0.08 (pro), $0.067 (mega) — volume discount
```

### Register an App

```typescript
// Run once per app (admin script or API endpoint)
import { generateApiKey } from './middleware/app-auth'

const apiKey = generateApiKey()          // ts_a1b2c3d4... — give this to the app developer
const keyHash = await hashApiKey(apiKey, env.SERVICE_SIGNING_KEY)

await db.insert(apps).values({
  id: 'image-gen',
  name: 'AI Image Generator',
  description: 'Generate images with AI',
  apiKeyHash: keyHash,
  defaultCostPerRequest: 5,              // 5 tokens per image generation
})

// IMPORTANT: Show the apiKey to the admin ONCE. It cannot be recovered after this.
console.log('API Key (save this):', apiKey)
```

---

## Folder Structure

```
token-service/
├── src/
│   ├── index.ts                              # Main entry point
│   ├── auth.ts                               # Better Auth factory
│   ├── lib/
│   │   └── stripe.ts                         # Stripe client factory
│   ├── db/
│   │   ├── index.ts                          # createDb factory
│   │   └── schema.ts                         # barrel export all schemas
│   ├── middleware/
│   │   ├── auth.ts                           # User session middleware
│   │   └── app-auth.ts                       # App API key middleware
│   └── domains/
│       └── tokens/
│           ├── tokens.schema.ts              # DB tables
│           ├── tokens.service.ts             # Core wallet logic
│           ├── tokens.routes.ts              # User-facing API
│           ├── tokens.app-routes.ts          # App-to-service API
│           └── tokens.webhooks.ts            # Stripe webhook handler
├── migrations/                               # Drizzle migrations
├── wrangler.toml
├── drizzle.config.ts
└── package.json
```

---

## API Reference (for App Developers)

Give this to developers building apps on your platform:

### Authentication

All requests require an API key in the `Authorization` header:

```
Authorization: Bearer ts_your_api_key_here
```

### Endpoints

#### Check Balance

```
GET /api/tokens/balance/:userId
```

Response: `{ "data": { "balance": 150 } }`

#### Deduct Tokens

```
POST /api/tokens/deduct
Content-Type: application/json

{
  "userId": "user_abc123",
  "amount": 5,
  "description": "Generated 1 image (512x512)",
  "referenceId": "req_xyz789"           // optional — your request ID for idempotency
}
```

Success (200): `{ "data": { "balance": 145, "ledgerEntryId": 42 } }`

Insufficient (402): `{ "error": "Insufficient tokens", "available": 3, "required": 5 }`

#### Refund Tokens

```
POST /api/tokens/refund
Content-Type: application/json

{
  "userId": "user_abc123",
  "amount": 5,
  "description": "Image generation failed — refund",
  "referenceId": "req_xyz789"
}
```

Response: `{ "data": { "balance": 150 } }`

---

## Security Notes

1. **API keys are hashed** — the raw key is never stored. If lost, generate a new one.
2. **Balance checks use optimistic locking** — the WHERE clause includes `balance >= amount`
   to prevent race conditions between check and deduct.
3. **Ledger is append-only** — never delete or update ledger entries. All corrections
   go through refund/bonus entries.
4. **Stripe webhook signatures are verified** — never process unverified events.
5. **User auth and app auth are separate** — users authenticate with sessions (cookies),
   apps authenticate with API keys (Bearer tokens). Different middleware, different routes.

---

## Official Documentation Links

| Topic | URL |
|-------|-----|
| Stripe Checkout (one-time) | https://docs.stripe.com/checkout/quickstart |
| Stripe on Workers | https://blog.cloudflare.com/announcing-stripe-support-in-workers/ |
| Stripe webhooks | https://docs.stripe.com/webhooks |
| D1 batch operations | https://developers.cloudflare.com/d1/worker-api/d1-database/#batch |
| Web Crypto (HMAC) | https://developers.cloudflare.com/workers/runtime-apis/web-crypto/ |
| Better Auth | https://www.better-auth.com/docs |
