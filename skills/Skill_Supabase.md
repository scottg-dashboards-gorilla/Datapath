# Supabase Skill — Datapath VCM

## Project Context
- Project: datapath-vcm
- Organization: Datapath's Org (corporate paid plan)
- Region: aws-us-west-1
- Auth: email/password only (no SSO in V1)

---

## Client Setup

Always use the correct client for the context:

```typescript
// Browser client (client components)
import { createBrowserClient } from '@supabase/ssr'

// Server client (server components, API routes)
import { createServerClient } from '@supabase/ssr'

// Service role client (cron jobs, admin operations)
// CRITICAL: Always use cache: 'no-store' to prevent stale data
const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!,
  { global: { fetch: (url, opts) => fetch(url, { ...opts, cache: 'no-store' }) } }
)
```

**Never use the service role key in client components** — it only belongs in server-side code and API routes.

---

## Environment Variables

```bash
NEXT_PUBLIC_SUPABASE_URL=        # Project URL
NEXT_PUBLIC_SUPABASE_ANON_KEY=   # Starts with eyJ — browser safe
SUPABASE_SERVICE_ROLE_KEY=       # Starts with eyJ — server only, longer
```

**Common mistake**: Anon key and service role key both start with `eyJ` and are easy to swap. The service role key is significantly longer. If getting 401 errors verify these are not swapped.

**PowerShell warning**: Avoid `^`, `` ` ``, `"`, `'`, `\`, `&`, `|` in passwords — these are PowerShell escape characters and will break auth.

---

## Schema Migrations

### Naming Convention
```
supabase/migrations/001_description.sql
supabase/migrations/002_description.sql
```
Always increment the number. Always save migrations to this folder.

### After Every Manual Schema Change
Run these GRANT statements or service role operations will silently fail:

```sql
grant usage on schema public to service_role;
grant all privileges on all tables in schema public to service_role;
grant all privileges on all sequences in schema public to service_role;
alter default privileges in schema public grant all on tables to service_role;
alter default privileges in schema public grant all on sequences to service_role;
```

### Generated Columns
Use `make_interval()` not `::interval` cast for generated columns:

```sql
-- CORRECT
cancellation_deadline date generated always as
  (current_term_end_date - make_interval(days => cancellation_notice_days))::date stored

-- WRONG - breaks in Supabase
cancellation_deadline date generated always as
  (current_term_end_date - (cancellation_notice_days || ' days')::interval)::date stored
```

### Type Generation
After any schema change regenerate TypeScript types:
```bash
npx supabase gen types typescript --project-ref [ref] > types/database.types.ts
```

---

## Row Level Security (RLS)

**Every new table must have RLS enabled.** No exceptions.

```sql
-- Enable RLS
alter table your_table enable row level security;

-- Standard authenticated access policy
create policy "authenticated_all" on your_table
  for all to authenticated using (true) with check (true);

-- Admin only policy
create policy "admin_only" on your_table
  for all to authenticated
  using (
    exists (
      select 1 from user_profiles
      where id = auth.uid() and role = 'admin'
    )
  );

-- Owner only policy
create policy "owner_only" on your_table
  for all to authenticated
  using (user_id = auth.uid());
```

**Common RLS gotcha**: If using service role key, RLS is bypassed. If using anon key, RLS applies. Use service role key in cron jobs and admin API routes. Use anon/user key in client-facing routes.

---

## Auth Patterns

### Auto-create user profile on first login
This trigger must exist in the database:

```sql
create or replace function handle_new_user()
returns trigger as $$
begin
  insert into public.user_profiles (id, email, role)
  values (new.id, new.email, 'viewer')
  on conflict (id) do nothing;
  return new;
end;
$$ language plpgsql security definer;

create trigger on_auth_user_created
  after insert on auth.users
  for each row execute procedure handle_new_user();
```

### Session handling in API routes
```typescript
const supabase = createServerClient(...)
const { data: { user }, error } = await supabase.auth.getUser()
if (!user) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
```

### User role check
```typescript
const { data: profile } = await supabase
  .from('user_profiles')
  .select('role, department')
  .eq('id', user.id)
  .single()

if (profile?.role !== 'admin') {
  return NextResponse.json({ error: 'Forbidden' }, { status: 403 })
}
```

---

## Supabase Storage

### Bucket naming convention
- `contract-documents` — contract PDF/Word/Excel attachments
- All buckets: private (public: false), auth required

### File path convention
```
contracts/{contract_id}/{timestamp}-{sanitized_filename}
```

### Upload pattern
```typescript
const { data, error } = await supabase.storage
  .from('contract-documents')
  .upload(filePath, file, {
    contentType: file.type,
    upsert: false
  })
```

### Download/signed URL
```typescript
const { data } = await supabase.storage
  .from('contract-documents')
  .createSignedUrl(storagePath, 3600) // 1 hour expiry
```

---

## Query Patterns

### Always use typed client
```typescript
import type { Database } from '@/types/database.types'
const supabase = createServerClient<Database>(...)
```

### Avoid N+1 queries
```typescript
// BAD - N+1
const contracts = await supabase.from('contracts').select('*')
for (const contract of contracts.data) {
  const vendor = await supabase.from('vendors').select('*').eq('id', contract.vendor_id)
}

// GOOD - single query with join
const contracts = await supabase
  .from('contracts')
  .select('*, vendors(*)')
```

### Two-step queries for auth user profiles
Supabase cannot join `auth.users` directly. Use two-step pattern:
```typescript
// Step 1: get IDs from your table
const { data: records } = await supabase.from('your_table').select('user_id')
const userIds = records.map(r => r.user_id)

// Step 2: get profiles separately
const { data: profiles } = await supabase
  .from('user_profiles')
  .select('id, first_name, last_name, email')
  .in('id', userIds)
```

### Money storage
Always store as integer cents. Never store as float.
```typescript
// Store
const cents = Math.round(dollars * 100)

// Display
const formatted = new Intl.NumberFormat('en-US', {
  style: 'currency',
  currency: 'USD'
}).format(cents / 100)
```

---

## Cron Jobs

### vercel.json schedule
```json
{
  "crons": [
    { "path": "/api/cron/send-alerts", "schedule": "0 8 * * *" },
    { "path": "/api/cron/send-digest", "schedule": "0 8 * * *" }
  ]
}
```

### Cron route protection
```typescript
export async function GET(request: Request) {
  const authHeader = request.headers.get('authorization')
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }
  // ... cron logic
}
```

### Test cron locally
```bash
curl -H "Authorization: Bearer $CRON_SECRET" http://localhost:3000/api/cron/send-alerts
```

---

## Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| 401 on API calls | Swapped anon/service role keys | Verify anon key is shorter, service role is longer |
| Role shows as "viewer" despite DB showing "admin" | Stale cache | Add `cache: 'no-store'` to service client |
| `invalid input syntax for type interval` | Using `::interval` in generated column | Use `make_interval(days => n)` instead |
| `permission denied for table` | Missing GRANT after manual migration | Run GRANT statements above |
| EPERM errors on Windows | Project in Dropbox/OneDrive folder | Move project to `C:\Coding-Projects\` |
| `expected string received null` on API | Zod `.optional()` doesn't accept null | Use `.nullable().optional()` for all optional text fields |
