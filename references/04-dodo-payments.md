# 04 — Dodo Payments (Checkout + Webhooks + Billing)

## Overview

Dodo Payments is a Stripe-alternative with no monthly fee. It uses checkout sessions, webhooks, and a REST API. Integrate from the **CF Worker** (never expose secret keys to the browser).

## Setup

1. Create account at https://dodopayments.com
2. Get: `DODO_PUBLIC_KEY`, `DODO_SECRET_KEY`, `DODO_WEBHOOK_SECRET`
3. Create Products & Price IDs in the Dodo dashboard

## CF Worker: Create Checkout Session (`src/routes/payments.ts`)

```typescript
import { Hono } from 'hono'

const payments = new Hono<{ Bindings: any; Variables: { userId: string; userEmail: string } }>()

// POST /api/payments/checkout  — creates a checkout session
payments.post('/checkout', async (c) => {
  const { priceId, successUrl, cancelUrl } = await c.req.json()
  const userId = c.get('userId')
  const userEmail = c.get('userEmail')

  const res = await fetch('https://api.dodopayments.com/v1/checkout/sessions', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${c.env.DODO_SECRET_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      price_id: priceId,
      customer_email: userEmail,
      metadata: { user_id: userId },
      success_url: successUrl ?? 'https://yourdomain.com/dashboard?upgraded=true',
      cancel_url: cancelUrl ?? 'https://yourdomain.com/pricing',
    }),
  })

  if (!res.ok) {
    const err = await res.json()
    return c.json({ error: err }, 400)
  }

  const session = await res.json()
  return c.json({ url: session.url })
})

// POST /payments/webhook  — PUBLIC route, verify Dodo signature
payments.post('/webhook', async (c) => {
  const body = await c.req.text()
  const signature = c.req.header('dodo-signature') ?? ''

  // Verify signature
  const isValid = await verifyDodoSignature(body, signature, c.env.DODO_WEBHOOK_SECRET)
  if (!isValid) return c.json({ error: 'Invalid signature' }, 401)

  const event = JSON.parse(body)

  switch (event.type) {
    case 'payment.succeeded':
      await handlePaymentSucceeded(event.data, c.env)
      break
    case 'subscription.created':
      await handleSubscriptionCreated(event.data, c.env)
      break
    case 'subscription.cancelled':
      await handleSubscriptionCancelled(event.data, c.env)
      break
    default:
      console.log('Unhandled event:', event.type)
  }

  return c.json({ received: true })
})

// HMAC-SHA256 signature verification
async function verifyDodoSignature(
  body: string,
  signature: string,
  secret: string
): Promise<boolean> {
  const encoder = new TextEncoder()
  const key = await crypto.subtle.importKey(
    'raw', encoder.encode(secret),
    { name: 'HMAC', hash: 'SHA-256' },
    false, ['verify']
  )
  const sigBytes = hexToBytes(signature)
  return crypto.subtle.verify('HMAC', key, sigBytes, encoder.encode(body))
}

function hexToBytes(hex: string): ArrayBuffer {
  const arr = new Uint8Array(hex.match(/../g)!.map(b => parseInt(b, 16)))
  return arr.buffer
}

// Handlers — update Supabase via service role
async function handlePaymentSucceeded(data: any, env: any) {
  const { getSupabase } = await import('../lib/supabase')
  const supabase = getSupabase(env)
  const userId = data.metadata?.user_id
  if (!userId) return

  await supabase
    .from('profiles')
    .update({ plan: 'pro' })
    .eq('id', userId)
}

async function handleSubscriptionCreated(data: any, env: any) {
  const { getSupabase } = await import('../lib/supabase')
  const supabase = getSupabase(env)
  await supabase.from('subscriptions').upsert({
    user_id: data.metadata?.user_id,
    dodo_subscription_id: data.id,
    status: data.status,
    price_id: data.price_id,
    current_period_end: data.current_period_end,
  })
}

async function handleSubscriptionCancelled(data: any, env: any) {
  const { getSupabase } = await import('../lib/supabase')
  const supabase = getSupabase(env)
  await supabase
    .from('profiles')
    .update({ plan: 'free' })
    .eq('id', data.metadata?.user_id)
}

export { payments as paymentRoutes }
```

## Subscriptions Table (Supabase)

```sql
create table public.subscriptions (
  id                    uuid default uuid_generate_v4() primary key,
  user_id               uuid references auth.users(id) on delete cascade,
  dodo_subscription_id  text unique,
  status                text,
  price_id              text,
  current_period_end    timestamptz,
  created_at            timestamptz default now()
);

alter table public.subscriptions enable row level security;
create policy "Users can view own subscription"
  on public.subscriptions for select using (auth.uid() = user_id);
```

## Frontend: Redirect to Checkout (Next.js Client Component)

```typescript
'use client'
import { supabase } from '@/lib/supabase/client'
import { apiCall } from '@/lib/api'

export function UpgradeButton({ priceId }: { priceId: string }) {
  async function handleUpgrade() {
    const { data: { session } } = await supabase.auth.getSession()
    const { url } = await apiCall<{ url: string }>('/api/payments/checkout', {
      method: 'POST',
      body: JSON.stringify({ priceId }),
      token: session?.access_token,
    })
    window.location.href = url
  }

  return <button onClick={handleUpgrade}>Upgrade to Pro</button>
}
```

## Dodo Dashboard Checklist

- [ ] Create product + price IDs
- [ ] Set webhook endpoint: `https://api.yourdomain.com/payments/webhook`
- [ ] Enable events: `payment.succeeded`, `subscription.created`, `subscription.cancelled`
- [ ] Copy webhook secret → `wrangler secret put DODO_WEBHOOK_SECRET`
