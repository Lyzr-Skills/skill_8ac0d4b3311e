---
name: free-ai-stack-architect
description: >
  Full-stack web architect for a zero-cost, production-ready AI app stack: Next.js (frontend) 
  deployed on Vercel, Cloudflare Workers (backend API), Supabase (database + auth), Dodo Payments 
  (billing), and Umami (analytics). Use this skill whenever the user wants to: scaffold a new 
  webapp, wire up any of these five services, generate boilerplate code, design API routes, 
  set up authentication, configure payments, embed analytics, troubleshoot integration issues, 
  or architect the data/API layer between these services. Trigger even on vague requests like 
  "help me build an app", "set up my backend", "add payments", "track users", or "connect my 
  database" — if the context suggests this stack, activate immediately.
---

# Free AI Stack Architect

## Stack Overview

```
┌─────────────────────────────────────────────────────────┐
│                    USER / BROWSER                        │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│           FRONTEND — Next.js 14+ (App Router)            │
│                  Deployed on Vercel                      │
│   • Server Components  • RSC  • Edge Runtime             │
└────────────────────────┬────────────────────────────────┘
                         │  fetch() / REST / tRPC
┌────────────────────────▼────────────────────────────────┐
│        BACKEND — Cloudflare Workers (Hono.js)            │
│      api.yourdomain.com  →  workers.dev                  │
│   • Auth middleware (Supabase JWT)                       │
│   • Business logic, AI inference, queue jobs             │
└──────┬─────────────────┬──────────────────┬─────────────┘
       │                 │                  │
┌──────▼──────┐  ┌───────▼──────┐  ┌───────▼──────┐
│  SUPABASE   │  │ DODO PAYMENTS│  │    UMAMI      │
│  Postgres   │  │  Webhooks    │  │  Analytics    │
│  Auth       │  │  Checkout    │  │  Self-hosted  │
│  Storage    │  │  Billing     │  │  or Cloud     │
└─────────────┘  └──────────────┘  └───────────────┘
```

## Quick-Start Workflow

When a user wants to scaffold a new app, follow this order:

1. **Capture intent** → read `references/00-architecture-decisions.md`
2. **Scaffold project** → read `references/01-nextjs-vercel.md`
3. **Build API layer** → read `references/02-cloudflare-workers.md`
4. **Wire database + auth** → read `references/03-supabase.md`
5. **Add payments** → read `references/04-dodo-payments.md`
6. **Embed analytics** → read `references/05-umami.md`
7. **Connect everything** → read `references/06-integration-patterns.md`

## Reference Files

| File | When to Read |
|------|-------------|
| `references/00-architecture-decisions.md` | Project kick-off, stack decisions, env vars map |
| `references/01-nextjs-vercel.md` | Frontend scaffolding, routing, deployment, env config |
| `references/02-cloudflare-workers.md` | Worker setup, Hono routing, D1/KV, Wrangler CLI |
| `references/03-supabase.md` | DB schema, RLS, auth (JWT + OAuth), storage, realtime |
| `references/04-dodo-payments.md` | Checkout sessions, webhooks, subscription lifecycle |
| `references/05-umami.md` | Script install, event tracking, self-hosting on Railway |
| `references/06-integration-patterns.md` | Cross-service patterns, auth propagation, error handling |

## Core Principles

- **Free tier first** — every service has a generous free tier; flag when limits approach
- **Edge-native** — prefer Vercel Edge + CF Workers over serverless functions where latency matters
- **JWT everywhere** — Supabase issues JWTs; Workers verify them; no session cookies needed
- **Webhook-driven** — Dodo Payments and Supabase realtime use webhooks; design for async
- **No vendor lock-in** — abstract third-party calls behind a service layer in Workers

## Boilerplate Commands

```bash
# 1. Scaffold Next.js app
npx create-next-app@latest my-app --typescript --tailwind --app --src-dir

# 2. Install core deps
cd my-app
npm install @supabase/supabase-js @supabase/ssr hono

# 3. Create Cloudflare Worker
npm create cloudflare@latest api -- --type=hono

# 4. Deploy frontend
npx vercel

# 5. Deploy worker
cd api && npx wrangler deploy
```

## Environment Variables Master List

```env
# Next.js (.env.local)
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
NEXT_PUBLIC_UMAMI_WEBSITE_ID=
NEXT_PUBLIC_API_URL=           # Your CF Worker URL
NEXT_PUBLIC_DODO_PUBLIC_KEY=

# Cloudflare Worker (wrangler.toml [vars] or secrets)
SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=
DODO_SECRET_KEY=
DODO_WEBHOOK_SECRET=
SUPABASE_JWT_SECRET=
```

## Decision Tree

```
User wants to...
├── Build a new app → Start at 00-architecture-decisions.md
├── Set up routing/pages → 01-nextjs-vercel.md
├── Create an API endpoint → 02-cloudflare-workers.md
├── Add login / signup → 03-supabase.md#auth
├── Design a DB schema → 03-supabase.md#database
├── Charge users → 04-dodo-payments.md
├── Track page views / events → 05-umami.md
└── Wire two services together → 06-integration-patterns.md
```
