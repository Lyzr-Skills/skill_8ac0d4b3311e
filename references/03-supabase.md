# 03 — Supabase (Database + Auth + Storage)

## Initial Setup

1. Create project at https://supabase.com
2. Copy: Project URL, `anon` key, `service_role` key, JWT secret
3. Install: `npm install @supabase/supabase-js @supabase/ssr`

## Core DB Schema (starter)

```sql
-- Enable UUID extension
create extension if not exists "uuid-ossp";

-- Profiles table (extends auth.users)
create table public.profiles (
  id           uuid references auth.users(id) on delete cascade primary key,
  email        text,
  full_name    text,
  avatar_url   text,
  plan         text default 'free' check (plan in ('free', 'pro', 'enterprise')),
  created_at   timestamptz default now()
);

-- Auto-create profile on signup
create or replace function public.handle_new_user()
returns trigger as $$
begin
  insert into public.profiles (id, email, full_name, avatar_url)
  values (
    new.id,
    new.email,
    new.raw_user_meta_data->>'full_name',
    new.raw_user_meta_data->>'avatar_url'
  );
  return new;
end;
$$ language plpgsql security definer;

create trigger on_auth_user_created
  after insert on auth.users
  for each row execute procedure public.handle_new_user();

-- RLS: users can only see their own profile
alter table public.profiles enable row level security;

create policy "Users can view own profile"
  on public.profiles for select
  using (auth.uid() = id);

create policy "Users can update own profile"
  on public.profiles for update
  using (auth.uid() = id);
```

## Auth: Email + Password

```typescript
// src/lib/supabase/client.ts (browser)
import { createBrowserClient } from '@supabase/ssr'
export const supabase = createBrowserClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
)

// Sign up
const { data, error } = await supabase.auth.signUp({
  email: 'user@example.com',
  password: 'securepassword',
  options: { data: { full_name: 'John Doe' } }
})

// Sign in
const { data, error } = await supabase.auth.signInWithPassword({
  email: 'user@example.com',
  password: 'securepassword',
})

// Get JWT for CF Worker calls
const { data: { session } } = await supabase.auth.getSession()
const jwt = session?.access_token

// Sign out
await supabase.auth.signOut()
```

## Auth: OAuth (Google, GitHub)

```typescript
// Trigger OAuth
await supabase.auth.signInWithOAuth({
  provider: 'google',
  options: { redirectTo: `${window.location.origin}/auth/callback` }
})
```

```typescript
// src/app/auth/callback/route.ts
import { createClient } from '@/lib/supabase/server'
import { NextResponse } from 'next/server'

export async function GET(request: Request) {
  const { searchParams, origin } = new URL(request.url)
  const code = searchParams.get('code')
  if (code) {
    const supabase = createClient()
    await supabase.auth.exchangeCodeForSession(code)
  }
  return NextResponse.redirect(`${origin}/dashboard`)
}
```

## Passing JWT to CF Worker

```typescript
// In a Client Component or Server Action
const { data: { session } } = await supabase.auth.getSession()
const data = await apiCall('/api/users/me', { token: session?.access_token })
```

## Row Level Security Patterns

```sql
-- Allow service_role to bypass (used by CF Worker)
-- Service role key bypasses RLS automatically ✓

-- Allow users to read only their own data
create policy "owner_read" on public.items
  for select using (auth.uid() = user_id);

-- Allow users to insert their own data
create policy "owner_insert" on public.items
  for insert with check (auth.uid() = user_id);
```

## Storage (File Uploads)

```typescript
// Upload
const { data, error } = await supabase.storage
  .from('avatars')
  .upload(`${userId}/avatar.png`, file, { upsert: true })

// Get public URL
const { data: { publicUrl } } = supabase.storage
  .from('avatars')
  .getPublicUrl(`${userId}/avatar.png`)
```

## Prevent Free Tier Pausing

Enable **"No project pausing"** in Supabase dashboard → Settings → General, or ping your project URL periodically with a cron job (e.g., via Cloudflare Workers Cron Triggers).
