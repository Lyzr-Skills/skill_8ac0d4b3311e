# 02 — Cloudflare Workers (Hono.js Backend)

## Scaffold

```bash
npm create cloudflare@latest api -- --type=hono
cd api
npm install @supabase/supabase-js jose
```

## `wrangler.toml`

```toml
name = "my-app-api"
main = "src/index.ts"
compatibility_date = "2024-09-23"
compatibility_flags = ["nodejs_compat"]

[vars]
SUPABASE_URL = "https://xxxx.supabase.co"

# Secrets (set via CLI, never in toml):
# wrangler secret put SUPABASE_SERVICE_ROLE_KEY
# wrangler secret put SUPABASE_JWT_SECRET
# wrangler secret put DODO_SECRET_KEY
# wrangler secret put DODO_WEBHOOK_SECRET
```

## Main App Entry (`src/index.ts`)

```typescript
import { Hono } from 'hono'
import { cors } from 'hono/cors'
import { authMiddleware } from './middleware/auth'
import { authRoutes } from './routes/auth'
import { paymentRoutes } from './routes/payments'
import { userRoutes } from './routes/users'

type Env = {
  SUPABASE_URL: string
  SUPABASE_SERVICE_ROLE_KEY: string
  SUPABASE_JWT_SECRET: string
  DODO_SECRET_KEY: string
  DODO_WEBHOOK_SECRET: string
}

const app = new Hono<{ Bindings: Env }>()

// CORS — allow your Vercel domain
app.use('*', cors({
  origin: ['https://yourdomain.com', 'http://localhost:3000'],
  allowHeaders: ['Authorization', 'Content-Type'],
  allowMethods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
}))

app.get('/health', (c) => c.json({ ok: true }))

// Public routes
app.route('/auth', authRoutes)
app.route('/payments/webhook', paymentRoutes) // webhook is public (verified by signature)

// Protected routes — verify JWT
app.use('/api/*', authMiddleware)
app.route('/api/users', userRoutes)

export default app
```

## JWT Auth Middleware (`src/middleware/auth.ts`)

```typescript
import { createMiddleware } from 'hono/factory'
import { verify } from 'jose'

export const authMiddleware = createMiddleware(async (c, next) => {
  const authHeader = c.req.header('Authorization')
  if (!authHeader?.startsWith('Bearer ')) {
    return c.json({ error: 'Unauthorized' }, 401)
  }

  const token = authHeader.replace('Bearer ', '')
  try {
    const secret = new TextEncoder().encode(c.env.SUPABASE_JWT_SECRET)
    const { payload } = await verify(token, secret)
    c.set('userId', payload.sub as string)
    c.set('userEmail', payload.email as string)
    await next()
  } catch {
    return c.json({ error: 'Invalid token' }, 401)
  }
})
```

## Supabase Admin Client in Worker (`src/lib/supabase.ts`)

```typescript
import { createClient } from '@supabase/supabase-js'

export function getSupabase(env: { SUPABASE_URL: string; SUPABASE_SERVICE_ROLE_KEY: string }) {
  return createClient(env.SUPABASE_URL, env.SUPABASE_SERVICE_ROLE_KEY, {
    auth: { persistSession: false }
  })
}
```

## Sample Route (`src/routes/users.ts`)

```typescript
import { Hono } from 'hono'
import { getSupabase } from '../lib/supabase'

const users = new Hono<{ Bindings: any; Variables: { userId: string } }>()

users.get('/me', async (c) => {
  const supabase = getSupabase(c.env)
  const userId = c.get('userId')
  const { data, error } = await supabase
    .from('profiles')
    .select('*')
    .eq('id', userId)
    .single()
  if (error) return c.json({ error: error.message }, 500)
  return c.json(data)
})

export { users as userRoutes }
```

## Deploy

```bash
# Development
npx wrangler dev

# Production
npx wrangler deploy

# Add secrets
npx wrangler secret put SUPABASE_SERVICE_ROLE_KEY
npx wrangler secret put SUPABASE_JWT_SECRET
npx wrangler secret put DODO_SECRET_KEY
npx wrangler secret put DODO_WEBHOOK_SECRET
```
