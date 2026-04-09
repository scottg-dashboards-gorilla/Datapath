# TypeScript Strict Mode Skill — Datapath VCM

## Config
TypeScript strict mode is enabled in `tsconfig.json`:
```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true
  }
}
```

Always run `npx tsc --noEmit` to verify — must show 0 errors before any feature is complete.

---

## Core Rules

### No `any` types — ever
```typescript
// BAD
const data: any = response
function process(input: any) {}

// GOOD — use unknown and narrow
const data: unknown = response
if (typeof data === 'object' && data !== null) { /* narrow here */ }

// GOOD — use specific types
function process(input: Contract) {}

// GOOD — use generics
function process<T>(input: T): T { return input }
```

### No non-null assertions without justification
```typescript
// BAD — can throw at runtime
const name = user!.profile!.name

// GOOD — optional chaining
const name = user?.profile?.name ?? 'Unknown'

// GOOD — explicit null check
if (!user?.profile) return null
const name = user.profile.name
```

---

## Null and Undefined Handling

### Optional form fields in Zod schemas
When a form field is optional and the form submits `null` for empty fields:
```typescript
// BAD — .optional() only accepts string | undefined, not null
website: z.string().optional()

// GOOD — accepts string | null | undefined
website: z.string().nullable().optional()

// Transform empty string to null before storing
website: z.string().nullable().optional().transform(v => v === '' ? null : v)
```

### Database fields that may be null
```typescript
// BAD
const vendorName = contract.vendor.name  // vendor could be null

// GOOD
const vendorName = contract.vendor?.name ?? 'Unknown vendor'

// GOOD — type narrowing
if (!contract.vendor) {
  return <span>No vendor</span>
}
return <span>{contract.vendor.name}</span>
```

### Array access with noUncheckedIndexedAccess
```typescript
// With noUncheckedIndexedAccess enabled, array[0] returns T | undefined
const contracts: Contract[] = []
const first = contracts[0]  // type is Contract | undefined

// GOOD — check before using
if (first) {
  console.log(first.contract_name)
}

// GOOD — use optional chaining
console.log(contracts[0]?.contract_name)
```

---

## Type Definitions

### Database types
Always use generated types from Supabase:
```typescript
import type { Database } from '@/types/database.types'

type Contract = Database['public']['Tables']['contracts']['Row']
type ContractInsert = Database['public']['Tables']['contracts']['Insert']
type ContractUpdate = Database['public']['Tables']['contracts']['Update']
```

### Extending database types
```typescript
// Contract with joined vendor data
type ContractWithVendor = Contract & {
  vendors: Pick<Vendor, 'id' | 'name'> | null
}

// Contract with all related data
type ContractFull = Contract & {
  vendors: Vendor | null
  contract_child_records: ChildRecord[]
  user_profiles: Pick<UserProfile, 'id' | 'first_name' | 'last_name' | 'email'> | null
}
```

### API response types
```typescript
// Always type API responses explicitly
type ApiResponse<T> = {
  data: T
  error?: never
} | {
  data?: never
  error: string
}

// Use in API routes
return NextResponse.json<ApiResponse<Contract>>({ data: contract })
return NextResponse.json<ApiResponse<Contract>>({ error: 'Not found' }, { status: 404 })
```

### Component prop types
```typescript
// Always explicit prop types — no implicit any
interface ContractCardProps {
  contract: ContractWithVendor
  onEdit: (id: string) => void
  onDelete: (id: string) => Promise<void>
  readOnly?: boolean  // optional props get ?
}

export function ContractCard({ contract, onEdit, onDelete, readOnly = false }: ContractCardProps) {
  // ...
}
```

---

## Common Strict Mode Patterns

### Discriminated unions for state
```typescript
// GOOD — exhaustive state handling
type LoadingState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; message: string }

function render(state: LoadingState<Contract[]>) {
  switch (state.status) {
    case 'idle': return null
    case 'loading': return <Skeleton />
    case 'success': return <ContractList contracts={state.data} />
    case 'error': return <Error message={state.message} />
  }
}
```

### Type guards
```typescript
// Custom type guard
function isContract(value: unknown): value is Contract {
  return (
    typeof value === 'object' &&
    value !== null &&
    'contract_name' in value &&
    'vendor_id' in value
  )
}
```

### Enum patterns (use const objects not TypeScript enums)
```typescript
// GOOD — const object (serializable, reversible)
const ContractStatus = {
  ACTIVE: 'active',
  EXPIRED: 'expired',
  CANCELLED: 'cancelled'
} as const

type ContractStatus = typeof ContractStatus[keyof typeof ContractStatus]
// type is 'active' | 'expired' | 'cancelled'

// BAD — TypeScript enum (compilation issues, not serializable)
enum ContractStatus { ACTIVE = 'active' }
```

### Async function return types
```typescript
// Always explicit return types on async functions
async function fetchContract(id: string): Promise<Contract | null> {
  const { data, error } = await supabase
    .from('contracts')
    .select('*')
    .eq('id', id)
    .single()
  
  if (error) return null
  return data
}

// API route handlers
export async function GET(request: Request): Promise<NextResponse> {
  // ...
}
```

---

## Handling Supabase Query Types

### Select with joins loses type inference — use explicit typing
```typescript
// Supabase loses type info on complex selects
const { data } = await supabase
  .from('contracts')
  .select('*, vendors(*), contract_child_records(*)')

// data is typed as any — fix with explicit type assertion
const { data } = await supabase
  .from('contracts')
  .select('*, vendors(*), contract_child_records(*)')
  .returns<ContractFull[]>()
```

### Handle null data from queries
```typescript
const { data: contracts, error } = await supabase.from('contracts').select('*')

// data is Contract[] | null — always provide fallback
const safeContracts = contracts ?? []
```

---

## React-specific TypeScript

### Event handler types
```typescript
// Form events
function handleSubmit(e: React.FormEvent<HTMLFormElement>) {}
function handleChange(e: React.ChangeEvent<HTMLInputElement>) {}
function handleClick(e: React.MouseEvent<HTMLButtonElement>) {}
```

### useRef types
```typescript
// DOM refs — null initial value
const inputRef = useRef<HTMLInputElement>(null)

// Mutable value refs — no null
const savedValueRef = useRef<string>('')
```

### Children prop
```typescript
// Use React.ReactNode for children
interface LayoutProps {
  children: React.ReactNode
}
```

### useState with explicit types
```typescript
// When initial state doesn't reveal the type
const [contract, setContract] = useState<Contract | null>(null)
const [contracts, setContracts] = useState<Contract[]>([])
const [status, setStatus] = useState<'idle' | 'loading' | 'error'>('idle')
```

---

## Common TypeScript Errors and Fixes

| Error | Fix |
|-------|-----|
| `Type 'null' is not assignable to type 'string'` | Add `\| null` to type or use `?? ''` fallback |
| `Object is possibly 'null'` | Add null check or optional chaining `?.` |
| `Object is possibly 'undefined'` | Add undefined check or nullish coalescing `?? defaultValue` |
| `Property does not exist on type` | Check generated types, add to interface, or use type assertion |
| `Argument of type 'string \| null' not assignable to 'string'` | Use `.nullable()` in Zod or add null check |
| `Type 'any' is not assignable` | Replace `any` with proper type |
| `Index signature... type 'undefined'` | Use optional chaining or explicit array index check |
