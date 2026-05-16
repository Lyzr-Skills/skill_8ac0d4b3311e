# 00 — Architecture Decisions & Project Setup

## Capture Intent Checklist

Before writing code, ask/confirm:

- [ ] App type: SaaS / Marketplace / Tool / Content site?
- [ ] Auth needed? (email+pass / OAuth / magic link)
- [ ] Payments? (one-time / subscription / usage-based)
- [ ] Expected scale at launch (users, req/s)
- [ ] Custom domain? (needed for Vercel + CF Worker routing)
- [ ] AI inference? (OpenAI/Anthropic via CF Worker proxy)
- [ ] File uploads? (Supabase Storage)

## Free Tier Limits (as of 2025)

| Service | Free Tier Highlights | Watch Out For |
|---------|---------------------|---------------|
| Vercel | 100GB bandwidth, unlimited deploys | 12 serverless function regions |
| Cloudflare Workers | 100k req/day, 10ms CPU/req | 128MB memory, no persistent TCP |
| Supabase | 500MB DB, 1GB storage, 50k MAU | Pauses after 1 week inactivity |
| Dodo Payments | No monthly fee — % per transaction | 2.9% + $0.30 per charge |
| Umami Cloud | 10k events/month free | Self-host on Railway for unlimited |

## Recommended Project Structure

```
my-app/
├── src/
│   ├── app/                    # Next.js App Router
│   │   ├── (auth)/             # Auth route group
│   │   ├── (dashboard)/        # Protected pages
│   │   ├── api/                # Next.js API routes (minimal — prefer CF Worker)
│   │   └── layout.tsx
│   ├── components/
│   ├── lib/
│   │   ├── supabase/
│   │   │   ├── client.ts       # Browser client
│   │   │   └── server.ts       # Server component client
│   │   ├── api.ts              # CF Worker fetch wrapper
│   │   └── analytics.ts        # Umami helpers
│   └── middleware.ts           # Supabase auth + protected routes
│
api/                            # Cloudflare Worker (separate repo/folder)
├── src/
│   ├── index.ts                # Hono app entry
│   ├── routes/
│   │   ├── auth.ts
│   │   ├── payments.ts
│   │   └── ...
│   ├── middleware/
│   │   └── auth.ts             # JWT verification
│   └── lib/
│       └── supabase.ts         # Service-role client
├── wrangler.toml
└── package.json
```

## Custom Domain Routing (Recommended)

```
yourdomain.com         → Vercel (Next.js frontend)
api.yourdomain.com     → Cloudflare Worker
```

Set `api.yourdomain.com` as a custom domain in your CF Worker dashboard. In Next.js, set `NEXT_PUBLIC_API_URL=https://api.yourdomain.com`.
