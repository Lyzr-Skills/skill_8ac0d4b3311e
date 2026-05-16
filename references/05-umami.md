# 05 — Umami Analytics

## Overview

Umami is a privacy-friendly, open-source analytics alternative to Google Analytics. Two deployment options:

| Option | Cost | Events/month | Setup Time |
|--------|------|-------------|------------|
| Umami Cloud | Free up to 10k events | 10k | 2 min |
| Self-hosted (Railway) | ~$5/mo or free hobby | Unlimited | 15 min |

## Option A: Umami Cloud (Fastest)

1. Sign up at https://umami.is
2. Add website → copy `Website ID`
3. Add script to `layout.tsx`:

```tsx
<script
  defer
  src="https://analytics.umami.is/script.js"
  data-website-id={process.env.NEXT_PUBLIC_UMAMI_WEBSITE_ID}
/>
```

4. Set env var: `NEXT_PUBLIC_UMAMI_WEBSITE_ID=your-website-id`

## Option B: Self-Host on Railway (Recommended for Unlimited Events)

```bash
# 1. Fork or use Railway template
# https://railway.app/template/umami

# 2. Set env vars in Railway:
DATABASE_URL=<postgres connection string>
HASH_SALT=<random string>
APP_SECRET=<random string>
```

Then point the script `src` to your Railway URL:
```tsx
src="https://your-umami.up.railway.app/script.js"
```

## Page View Tracking (Automatic)

The script auto-tracks all page views. No additional code needed.

## Custom Event Tracking

### Via HTML attributes (no JS needed)

```html
<button
  data-umami-event="upgrade-clicked"
  data-umami-event-plan="pro"
  data-umami-event-location="pricing-page"
>
  Upgrade to Pro
</button>
```

### Via JS (TypeScript)

```typescript
// src/lib/analytics.ts
declare global {
  interface Window {
    umami?: {
      track: (event: string, data?: Record<string, string | number>) => void
    }
  }
}

export function trackEvent(event: string, data?: Record<string, string | number>) {
  if (typeof window !== 'undefined' && window.umami) {
    window.umami.track(event, data)
  }
}
```

```typescript
// Usage in components
import { trackEvent } from '@/lib/analytics'

// Track signup
trackEvent('user-signed-up', { plan: 'free' })

// Track purchase
trackEvent('purchase-completed', { plan: 'pro', amount: 29 })

// Track feature usage
trackEvent('feature-used', { feature: 'ai-generation', count: 1 })
```

## useUmami Hook (React)

```typescript
// src/hooks/useUmami.ts
'use client'
import { useCallback } from 'react'

export function useUmami() {
  const track = useCallback((event: string, data?: Record<string, string | number>) => {
    if (typeof window !== 'undefined' && window.umami) {
      window.umami.track(event, data)
    }
  }, [])
  return { track }
}

// Usage
const { track } = useUmami()
track('button-clicked', { button: 'upgrade' })
```

## Key Events to Track for SaaS

```typescript
// Auth events
trackEvent('signup', { method: 'email' | 'google' })
trackEvent('login', { method: 'email' | 'google' })

// Activation
trackEvent('onboarding-completed')
trackEvent('first-project-created')

// Revenue
trackEvent('checkout-started', { plan: 'pro' })
trackEvent('upgrade-completed', { plan: 'pro', mrr: 29 })
trackEvent('subscription-cancelled', { plan: 'pro' })

// Engagement
trackEvent('feature-used', { feature: 'ai-chat' })
trackEvent('export-downloaded', { format: 'pdf' })
```

## Next.js Script Component (Alternative)

```tsx
// src/app/layout.tsx
import Script from 'next/script'

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        {children}
        <Script
          defer
          src="https://analytics.umami.is/script.js"
          data-website-id={process.env.NEXT_PUBLIC_UMAMI_WEBSITE_ID}
          strategy="afterInteractive"
        />
      </body>
    </html>
  )
}
```
