# SaaS Billing — Single Product

Add Stripe billing to a single app. Subscriptions, one-time payments, customer portal.

> **Building a multi-app platform instead?** See `references/token-service.md` + `references/token-apps.md`
> for the shared token wallet pattern.

## Decision Matrix

| Model | Stripe Feature | Best For |
|-------|---------------|----------|
| Monthly/annual subscription | Stripe Checkout + Billing | SaaS with tiers (free/pro/team) |
| One-time payment | Stripe Checkout (payment mode) | Digital products, lifetime deals |
| Usage-based (metered) | Stripe Billing + usage records | Pay-per-API-call, pay-per-seat |
| Credits / token packs | Stripe Checkout + your own ledger | AI tools, marketplaces — see `token-service.md` |

---

## Setup

### Install

```bash
npm install stripe
```

### Wrangler Config

```toml
# wrangler.toml
compatibility_flags = ["nodejs_compat"]

[[d1_databases]]
binding = "DB"
database_name = "my-db"
database_id = "xxx"
```

### Secrets

```bash
wrangler secret put STRIPE_SECRET_KEY          # sk_live_... or sk_test_...
wrangler secret put STRIPE_WEBHOOK_SECRET      # whsec_...
wrangler secret put STRIPE_PUBLISHABLE_KEY     # pk_live_... (optional — usually in frontend env)
```

For local dev, create `.dev.vars`:
```env
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
```

---

## Database Schema (Drizzle)

```typescript
// src/domains/billing/billing.schema.ts
import { sqliteTable, integer, text } from 'drizzle-orm/sqlite-core'
import { createInsertSchema, createSelectSchema } from 'drizzle-zod'
import { z } from 'zod'

// Link your users to Stripe customers
export const customers = sqliteTable('customers', {
  userId: text('user_id').primaryKey(),                    // your app's user ID
  stripeCustomerId: text('stripe_customer_id').notNull().unique(),
  createdAt: integer('created_at', { mode: 'timestamp' }).$defaultFn(() => new Date()),
})

// Track subscriptions
export const subscriptions = sqliteTable('subscriptions', {
  id: text('id').primaryKey(),                             // Stripe subscription ID
  userId: text('user_id').notNull(),
  stripePriceId: text('stripe_price_id').notNull(),
  status: text('status').notNull(),                        // active, canceled, past_due, etc.
  currentPeriodEnd: integer('current_period_end', { mode: 'timestamp' }).notNull(),
  cancelAtPeriodEnd: integer('cancel_at_period_end', { mode: 'boolean' }).default(false),
  createdAt: integer('created_at', { mode: 'timestamp' }).$defaultFn(() => new Date()),
  updatedAt: integer('updated_at', { mode: 'timestamp' }).$defaultFn(() => new Date()),
})

// Track one-time payments
export const payments = sqliteTable('payments', {
  id: text('id').primaryKey(),                             // Stripe payment intent ID
  userId: text('user_id').notNull(),
  amount: integer('amount').notNull(),                     // in cents
  currency: text('currency').notNull().default('usd'),
  status: text('status').notNull(),                        // succeeded, failed, pending
  createdAt: integer('created_at', { mode: 'timestamp' }).$defaultFn(() => new Date()),
})

export const selectSubscriptionSchema = createSelectSchema(subscriptions)
export type Subscription = z.infer<typeof selectSubscriptionSchema>
```

### Migration

```bash
npx drizzle-kit generate
wrangler d1 migrations apply my-db --remote
```

---

## Stripe Client Factory

```typescript
// src/lib/stripe.ts
import Stripe from 'stripe'

export function createStripe(secretKey: string) {
  return new Stripe(secretKey, {
    httpClient: Stripe.createFetchHttpClient(),  // Required for Workers (no Node HTTP)
  })
}
```

> **IMPORTANT:** On Workers you MUST use `Stripe.createFetchHttpClient()`. The default
> Node.js HTTP client doesn't work in the Workers runtime.

---

## Checkout Flow

### Create Checkout Session (API)

```typescript
// src/domains/billing/billing.routes.ts
import { Hono } from 'hono'
import { createStripe } from '../../lib/stripe'
import { createDb } from '../../db'
import { customers } from './billing.schema'
import { eq } from 'drizzle-orm'

type Bindings = { DB: D1Database; STRIPE_SECRET_KEY: string }
const app = new Hono<{ Bindings: Bindings; Variables: { userId: string } }>()

// Create checkout session for subscription
app.post('/checkout', async (c) => {
  const userId = c.get('userId')
  const { priceId } = await c.req.json()
  const stripe = createStripe(c.env.STRIPE_SECRET_KEY)
  const db = createDb(c.env.DB)

  // Get or create Stripe customer
  let customer = await db.select().from(customers).where(eq(customers.userId, userId)).get()

  if (!customer) {
    const stripeCustomer = await stripe.customers.create({
      metadata: { userId },
    })
    customer = (await db.insert(customers).values({
      userId,
      stripeCustomerId: stripeCustomer.id,
    }).returning())[0]
  }

  // Create checkout session
  const session = await stripe.checkout.sessions.create({
    customer: customer.stripeCustomerId,
    mode: 'subscription',                        // or 'payment' for one-time
    line_items: [{ price: priceId, quantity: 1 }],
    success_url: `${c.req.header('origin')}/billing?success=true`,
    cancel_url: `${c.req.header('origin')}/billing?canceled=true`,
    metadata: { userId },
  })

  return c.json({ data: { url: session.url } })
})

// Create customer portal session (manage subscriptions, invoices, payment methods)
app.post('/portal', async (c) => {
  const userId = c.get('userId')
  const stripe = createStripe(c.env.STRIPE_SECRET_KEY)
  const db = createDb(c.env.DB)

  const customer = await db.select().from(customers).where(eq(customers.userId, userId)).get()
  if (!customer) return c.json({ error: 'No billing account' }, 404)

  const session = await stripe.billingPortal.sessions.create({
    customer: customer.stripeCustomerId,
    return_url: `${c.req.header('origin')}/billing`,
  })

  return c.json({ data: { url: session.url } })
})

export { app as billingRouter }
```

### Frontend — Redirect to Checkout

```typescript
// client/src/domains/billing/billing.api.ts
import { apiFetch } from '@/lib/api'

export async function redirectToCheckout(priceId: string) {
  const { data } = await apiFetch<{ data: { url: string } }>('/billing/checkout', {
    method: 'POST',
    body: JSON.stringify({ priceId }),
  })
  window.location.href = data.url  // Redirect to Stripe
}

export async function redirectToPortal() {
  const { data } = await apiFetch<{ data: { url: string } }>('/billing/portal', {
    method: 'POST',
  })
  window.location.href = data.url
}
```

---

## Stripe Webhooks

Stripe sends events to your Worker when payments succeed, subscriptions change, etc.
This is the **source of truth** for billing state — never trust the client.

### Webhook Endpoint

```typescript
// src/domains/billing/billing.webhooks.ts
import Stripe from 'stripe'
import { createStripe } from '../../lib/stripe'
import { createDb } from '../../db'
import { subscriptions, payments, customers } from './billing.schema'
import { eq } from 'drizzle-orm'

// IMPORTANT: Stripe needs the SubtleCrypto provider on Workers
const webCrypto = Stripe.createSubtleCryptoProvider()

export async function handleWebhook(request: Request, env: any) {
  const stripe = createStripe(env.STRIPE_SECRET_KEY)
  const body = await request.text()
  const signature = request.headers.get('stripe-signature')

  if (!signature) return new Response('Missing signature', { status: 400 })

  // Verify webhook signature — use constructEventAsync (NOT constructEvent) on Workers
  let event: Stripe.Event
  try {
    event = await stripe.webhooks.constructEventAsync(
      body,
      signature,
      env.STRIPE_WEBHOOK_SECRET,
      undefined,
      webCrypto
    )
  } catch (err) {
    return new Response(`Webhook signature verification failed`, { status: 400 })
  }

  const db = createDb(env.DB)

  switch (event.type) {
    case 'checkout.session.completed': {
      const session = event.data.object as Stripe.Checkout.Session
      // Subscription was created — webhook for subscription.created handles the rest
      break
    }

    case 'customer.subscription.created':
    case 'customer.subscription.updated': {
      const sub = event.data.object as Stripe.Subscription
      const userId = sub.metadata.userId
        || (await db.select().from(customers)
            .where(eq(customers.stripeCustomerId, sub.customer as string)).get())?.userId

      if (!userId) break

      await db.insert(subscriptions).values({
        id: sub.id,
        userId,
        stripePriceId: sub.items.data[0].price.id,
        status: sub.status,
        currentPeriodEnd: new Date(sub.current_period_end * 1000),
        cancelAtPeriodEnd: sub.cancel_at_period_end,
        updatedAt: new Date(),
      }).onConflictDoUpdate({
        target: subscriptions.id,
        set: {
          status: sub.status,
          stripePriceId: sub.items.data[0].price.id,
          currentPeriodEnd: new Date(sub.current_period_end * 1000),
          cancelAtPeriodEnd: sub.cancel_at_period_end,
          updatedAt: new Date(),
        },
      })
      break
    }

    case 'customer.subscription.deleted': {
      const sub = event.data.object as Stripe.Subscription
      await db.update(subscriptions)
        .set({ status: 'canceled', updatedAt: new Date() })
        .where(eq(subscriptions.id, sub.id))
      break
    }

    case 'payment_intent.succeeded': {
      const pi = event.data.object as Stripe.PaymentIntent
      await db.insert(payments).values({
        id: pi.id,
        userId: pi.metadata.userId,
        amount: pi.amount,
        currency: pi.currency,
        status: 'succeeded',
      }).onConflictDoNothing()
      break
    }
  }

  return new Response(JSON.stringify({ received: true }))
}
```

### Mount Webhook Route

```typescript
// src/index.ts — mount BEFORE auth middleware (webhooks aren't authenticated by your app)
app.post('/webhooks/stripe', async (c) => {
  return handleWebhook(c.req.raw, c.env)
})

// Protected routes (auth middleware applied AFTER webhook route)
app.use('/billing/*', authMiddleware())
app.route('/billing', billingRouter)
```

### Register Webhook in Stripe

```bash
# Local development — use Stripe CLI to forward events
stripe listen --forward-to http://localhost:8787/webhooks/stripe

# Production — register in Stripe Dashboard:
# Developers → Webhooks → Add endpoint
# URL: https://your-api.workers.dev/webhooks/stripe
# Events: checkout.session.completed, customer.subscription.created,
#          customer.subscription.updated, customer.subscription.deleted,
#          payment_intent.succeeded
```

---

## Subscription Gate Middleware

Check if a user has an active subscription before allowing access to premium features.

```typescript
// src/middleware/subscription.ts
import { createDb } from '../db'
import { subscriptions } from '../domains/billing/billing.schema'
import { eq, and } from 'drizzle-orm'

export function requireSubscription(allowedPriceIds?: string[]) {
  return async (c: any, next: any) => {
    const userId = c.get('userId')
    const db = createDb(c.env.DB)

    const sub = await db.select().from(subscriptions)
      .where(and(
        eq(subscriptions.userId, userId),
        eq(subscriptions.status, 'active')
      ))
      .get()

    if (!sub) return c.json({ error: 'Subscription required' }, 403)

    if (allowedPriceIds && !allowedPriceIds.includes(sub.stripePriceId)) {
      return c.json({ error: 'Plan upgrade required' }, 403)
    }

    c.set('subscription', sub)
    await next()
  }
}

// Usage:
// app.use('/premium/*', requireSubscription())
// app.use('/team/*', requireSubscription(['price_team_monthly', 'price_team_annual']))
```

---

## Stripe Products Setup

Create your products and prices in Stripe. You can do this in the Dashboard or via API:

```bash
# Create a product via Stripe CLI
stripe products create --name="Pro Plan" --description="Full access"

# Create prices
stripe prices create \
  --product=prod_xxx \
  --unit-amount=1000 \
  --currency=usd \
  --recurring[interval]=month

stripe prices create \
  --product=prod_xxx \
  --unit-amount=9600 \
  --currency=usd \
  --recurring[interval]=year
```

Store your price IDs as environment variables or constants:

```typescript
// src/lib/constants.ts
export const PLANS = {
  free: { priceId: null, name: 'Free', limits: { apiCalls: 100 } },
  pro: {
    monthly: { priceId: 'price_xxx_monthly', name: 'Pro Monthly', amount: 1000 },
    annual: { priceId: 'price_xxx_annual', name: 'Pro Annual', amount: 9600 },
    limits: { apiCalls: 10_000 },
  },
  team: {
    monthly: { priceId: 'price_xxx_team_monthly', name: 'Team Monthly', amount: 4900 },
    limits: { apiCalls: 100_000 },
  },
} as const
```

---

## Folder Structure

```
src/domains/billing/
├── billing.schema.ts       # customers, subscriptions, payments tables
├── billing.routes.ts       # checkout, portal endpoints
├── billing.webhooks.ts     # Stripe webhook handler
└── billing.middleware.ts   # requireSubscription gate
```

---

## Checklist

- [ ] Create Stripe account + get API keys
- [ ] Store secrets: `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`
- [ ] Create products + prices in Stripe Dashboard
- [ ] Add `compatibility_flags = ["nodejs_compat"]` to `wrangler.toml`
- [ ] Run migration for billing tables
- [ ] Mount webhook route BEFORE auth middleware
- [ ] Register webhook URL in Stripe Dashboard
- [ ] Test with Stripe CLI: `stripe listen --forward-to localhost:8787/webhooks/stripe`
- [ ] Use test cards: `4242 4242 4242 4242` (success), `4000 0000 0000 0002` (decline)

---

## Official Documentation Links

| Topic | URL |
|-------|-----|
| Stripe on Workers (official) | https://blog.cloudflare.com/announcing-stripe-support-in-workers/ |
| Stripe Node SDK | https://github.com/stripe/stripe-node |
| Stripe Checkout quickstart | https://docs.stripe.com/checkout/quickstart |
| Stripe Billing (subscriptions) | https://docs.stripe.com/billing/subscriptions/overview |
| Stripe Customer Portal | https://docs.stripe.com/billing/subscriptions/integrations/customer-portal |
| Stripe Webhooks | https://docs.stripe.com/webhooks |
| Stripe CLI | https://docs.stripe.com/stripe-cli |
| Stripe test cards | https://docs.stripe.com/testing#cards |
| Workers + Stripe template | https://github.com/stripe-samples/stripe-node-cloudflare-worker-template |
