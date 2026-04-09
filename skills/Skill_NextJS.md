# Next.js App Router Skill — Datapath VCM

## Version
Next.js 14 with App Router, TypeScript strict mode.

---

## Project Structure

```
app/
  (auth)/           # Auth pages — no dashboard layout
    login/
    forgot-password/
    reset-password/
  (dashboard)/      # Protected pages — dashboard layout wrapper
    dashboard/
    vendors/
    contracts/
      [id]/         # Contract detail
    finance-tasks/
    reports/
    admin/
      users/
      groups/
      settings/
      audit-log/
      access-requests/
  api/              # API routes
    contracts/
    vendors/
    users/
    groups/
    settings/
    cron/
    slack/
    search/
  auth/
    callback/       # Supabase auth code exchange
```

---

## Server vs Client Components

### Default to Server Components
Every component is a server component by default. Only add `'use client'` when needed.

**Use server components for**:
- Data fetching
- Database queries
- Page components
- Layout wrappers
- Static content

**Use client components for**:
- Event handlers (onClick, onChange)
- useState, useEffect, useRef
- Browser APIs
- Interactive UI (modals, forms, dropdowns)
- shadcn components that use Radix (they need client)

### Pattern: Server fetches, client renders
```typescript
// app/(dashboard)/contracts/page.tsx — SERVER
export default async function ContractsPage() {
  const supabase = createServerClient()
  const { data: contracts } = await supabase.from('contracts').select('*, vendors(*)')
  return <ContractsClient contracts={contracts} />  // pass to client
}

// components/contracts/contracts-client.tsx — CLIENT
'use client'
export function ContractsClient({ contracts }) {
  const [filter, setFilter] = useState('')
  // interactive logic here
}
```

### Never do data fetching in client components
```typescript
// BAD - data fetching in client component
'use client'
export function ContractsList() {
  const [contracts, setContracts] = useState([])
  useEffect(() => {
    fetch('/api/contracts').then(r => r.json()).then(setContracts)
  }, [])
}

// GOOD - fetch in server, pass as props
```

---

## API Routes

### Standard API route structure
```typescript
// app/api/contracts/route.ts
import { NextResponse } from 'next/server'
import { createServerClient } from '@/lib/supabase/server'

export async function GET(request: Request) {
  const supabase = createServerClient()
  
  // 1. Auth check
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  
  // 2. Role check if needed
  const { data: profile } = await supabase
    .from('user_profiles')
    .select('role')
    .eq('id', user.id)
    .single()
  
  // 3. Business logic
  const { data, error } = await supabase.from('contracts').select('*')
  if (error) return NextResponse.json({ error: error.message }, { status: 500 })
  
  // 4. Return typed response
  return NextResponse.json(data)
}

export async function POST(request: Request) {
  // Always parse and validate body with Zod
  const body = await request.json()
  const parsed = contractSchema.safeParse(body)
  if (!parsed.success) {
    return NextResponse.json({ error: parsed.error.errors }, { status: 400 })
  }
  // ...
}
```

### Dynamic route segments
```typescript
// app/api/contracts/[id]/route.ts
export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  const { id } = params
  // ...
}
```

### HTTP status codes to use
```
200 — success with data
201 — created successfully
204 — success no content (delete)
400 — bad request / validation error
401 — not authenticated
403 — authenticated but forbidden
404 — resource not found
409 — conflict (duplicate, already resolved)
500 — server error
```

---

## Middleware

### Auth middleware pattern
```typescript
// middleware.ts
import { createServerClient } from '@supabase/ssr'
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export async function middleware(request: NextRequest) {
  // Refresh session
  // Redirect unauthenticated to /login
  // Redirect authenticated /login to /dashboard
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico).*)']
}
```

---

## Data Fetching Patterns

### Parallel fetching (always prefer over sequential)
```typescript
// GOOD - parallel
const [contracts, vendors, settings] = await Promise.all([
  supabase.from('contracts').select('*'),
  supabase.from('vendors').select('*'),
  supabase.from('system_settings').select('*')
])

// BAD - sequential (3x slower)
const contracts = await supabase.from('contracts').select('*')
const vendors = await supabase.from('vendors').select('*')
const settings = await supabase.from('system_settings').select('*')
```

### Error handling in server components
```typescript
export default async function Page() {
  const { data, error } = await supabase.from('contracts').select('*')
  
  if (error) {
    // Don't throw — render error state
    return <div>Failed to load contracts. Please refresh.</div>
  }
  
  return <ContractsClient contracts={data ?? []} />
}
```

---

## Forms

### Always use react-hook-form + Zod
```typescript
'use client'
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'

const schema = z.object({
  contract_name: z.string().min(1, 'Contract name is required'),
  website: z.string().nullable().optional(),  // optional text fields
  base_value: z.number().min(0)
})

type FormData = z.infer<typeof schema>

export function ContractForm() {
  const form = useForm<FormData>({
    resolver: zodResolver(schema),
    defaultValues: { contract_name: '', website: null }
  })
  
  async function onSubmit(data: FormData) {
    const response = await fetch('/api/contracts', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    })
    // handle response
  }
}
```

---

## Layout Patterns

### Dashboard layout (app/(dashboard)/layout.tsx)
- Server component
- Checks auth — redirects to /login if not authenticated
- Renders sidebar + main content area
- All dashboard pages inherit this layout automatically

### Two-column contract detail layout
- Left column (flex-1): main content tabs (Overview, History, Documents)
- Right column (w-80): metadata card, renewal countdown, watchers, permissions
- Stack to single column on mobile

---

## Performance

### Don't use `cache: 'force-cache'` with user data
All user-specific data must be fresh:
```typescript
// For user-specific queries always use no-store
fetch('/api/contracts', { cache: 'no-store' })
```

### Image optimization
```typescript
import Image from 'next/image'
// Always use next/image for any images, never <img>
```

### Route prefetching
```typescript
import Link from 'next/link'
// Always use next/link for internal navigation, never <a>
<Link href="/contracts">Contracts</Link>
```

---

## Environment Variables

```typescript
// Client-safe (can use in both server and client)
process.env.NEXT_PUBLIC_SUPABASE_URL
process.env.NEXT_PUBLIC_APP_URL

// Server-only (never expose to client)
process.env.SUPABASE_SERVICE_ROLE_KEY
process.env.SLACK_BOT_TOKEN
process.env.SLACK_SIGNING_SECRET
process.env.RESEND_API_KEY
process.env.CRON_SECRET
```

**Rule**: If a variable does NOT start with `NEXT_PUBLIC_` it is server-only. Never reference server-only variables in client components — they will be undefined and may expose secrets.

---

## Common Mistakes to Avoid

1. **Calling API routes from server components** — fetch directly from DB instead
2. **Using `useEffect` for initial data loading** — use server components
3. **Forgetting `'use client'`** on components that use hooks or event handlers
4. **Using `<a>` tags** instead of `<Link>` for internal navigation
5. **Sequential awaits** instead of `Promise.all` for parallel data
6. **Storing sensitive env vars with NEXT_PUBLIC_ prefix** — they become public
7. **Not handling loading and error states** in client components
