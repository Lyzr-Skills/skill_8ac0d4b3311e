# 06 — Integration Patterns (Cross-Service Wiring)

## Auth Flow: End-to-End

```
Browser                 Next.js (Vercel)        CF Worker          Supabase
   │                          │                     │                  │
   │──signIn(email,pw)────────▶                     │                  │
   │                          │──auth.signIn────────────────────────▶  │
   │                          │◀──JWT (access_token)────────────────── │
   │◀──session (JWT stored)───│                     │                  │
   │                          │                     │                  │
   │──fetch /api/users/me ───────────────────────▶  │                  │
   │   (Bearer: JWT)          │                     │──verify JWT──▶   │
   │                          │                     │◀──valid──────    │
   │                          │                     │──query DB───────▶│
   │◀──user data ────────────────────────────────── │◀──row data───    │
```

## Payment Flow: Checkout → Webhook → DB Update

```
Browser           Next.js           CF Worker         Dodo          Supabase
   │                 │                  │               │               │
   │──"Upgrade" ────▶│                  │               │               │
   │                 │──POST /checkout──▶               │               │
   │                 │                  │──create session▶              │
   │                 │                  │◀──{url}───────│               │
   │◀──redirect URL──│◀──{url}──────────│               │               │
   │                 │                  │               │               │
   │──────────────── Dodo Checkout ──────────────────▶  │               │
   │◀──────────────── success_url redirect ─────────────│               │
   │                 │                  │               │               │
   │                 │                  │◀──webhook─────│               │
   │                 │                  │  (verified)   │               │
   │                 │                  │──update plan──────────────────▶
```

## AI Inference Pattern (OpenAI/Anthropic via CF Worker)

Never call AI APIs from Next.js directly (exposes API keys). Route through CF Worker:

```typescript
// CF Worker route: POST /api/ai/chat
app.post('/api/ai/chat', authMiddleware, async (c) => {
  const { messages } = await c.req.json()
  const userId = c.get('userId')

  // Optional: check user's plan/credits in Supabase
  const supabase = getSupabase(c.env)
  const { data: profile } = await supabase
    .from('profiles')
    .select('plan')
    .eq('id', userId)
    .single()

  if (profile?.plan === 'free' && /* over limit */) {
    return c.json({ error: 'Upgrade to Pro for more AI credits' }, 402)
  }

  // Forward to AI provider
  const res = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: {
      'x-api-key': c.env.ANTHROPIC_API_KEY,
      'anthropic-version': '2023-06-01',
      'content-type': 'application/json',
    },
    body: JSON.stringify({ model: 'claude-sonnet-4-20250514', max_tokens: 1024, messages }),
  })

  const data = await res.json()
  return c.json(data)
})
```

## Protected Route Pattern (Next.js)

```typescript
// Higher-order component for client-side protection
'use client'
import { useEffect } from 'react'
import { useRouter } from 'next/navigation'
import { supabase } from '@/lib/supabase/client'

export function useRequireAuth() {
  const router = useRouter()
  useEffect(() => {
    supabase.auth.getSession().then(({ data: { session } }) => {
      if (!session) router.push('/login')
    })
  }, [router])
}
```

## Plan-Gating UI Pattern

```typescript
// src/components/PlanGate.tsx
import { createClient } from '@/lib/supabase/server'

type Plan = 'free' | 'pro' | 'enterprise'
const PLAN_RANK: Record<Plan, number> = { free: 0, pro: 1, enterprise: 2 }

export async function PlanGate({
  requiredPlan,
  children,
  fallback,
}: {
  requiredPlan: Plan
  children: React.ReactNode
  fallback?: React.ReactNode
}) {
  const supabase = createClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) return fallback ?? null

  const { data: profile } = await supabase
    .from('profiles')
    .select('plan')
    .eq('id', user.id)
    .single()

  const userRank = PLAN_RANK[(profile?.plan ?? 'free') as Plan]
  const requiredRank = PLAN_RANK[requiredPlan]

  return userRank >= requiredRank ? <>{children}</> : <>{fallback}</>
}

// Usage in Server Component:
// <PlanGate requiredPlan="pro" fallback={<UpgradePrompt />}>
//   <ProFeature />
// </PlanGate>
```

## Error Handling Patterns

```typescript
// Unified API error type
export class ApiError extends Error {
  constructor(public status: number, message: string) {
    super(message)
  }
}

// CF Worker error handler
app.onError((err, c) => {
  console.error(err)
  if (err instanceof ApiError) {
    return c.json({ error: err.message }, err.status as any)
  }
  return c.json({ error: 'Internal Server Error' }, 500)
})

// Next.js API error handling
export async function apiCall<T>(path: string, options?: RequestInit & { token?: string }): Promise<T> {
  const res = await fetch(...)
  if (!res.ok) {
    const body = await res.json().catch(() => ({ error: 'Unknown error' }))
    throw new ApiError(res.status, body.error ?? 'Request failed')
  }
  return res.json()
}
```

## Cron: Keep Supabase Alive (CF Worker Cron Trigger)

```toml
# wrangler.toml
[triggers]
crons = ["0 */12 * * *"]  # Every 12 hours
```

```typescript
// src/index.ts — add scheduled handler
export default {
  async fetch(request: Request, env: Env) { return app.fetch(request, env) },
  async scheduled(_event: ScheduledEvent, env: Env) {
    // Ping Supabase to prevent project pausing
    const supabase = getSupabase(env)
    await supabase.from('profiles').select('id').limit(1)
    console.log('Supabase ping OK')
  }
}
```

## Local Development Setup

```bash
# Terminal 1: Next.js
cd my-app && npm run dev  # http://localhost:3000

# Terminal 2: CF Worker
cd api && npx wrangler dev --port 8787  # http://localhost:8787

# .env.local
NEXT_PUBLIC_API_URL=http://localhost:8787
```
