# Testing Skill — Datapath VCM

## Stack
- **Unit/API tests**: Vitest
- **E2E tests**: Playwright
- **Type checking**: TypeScript (`npx tsc --noEmit`)

## Required After Every Feature
Run all three before declaring any feature complete:
```bash
npm test                  # Vitest — must be 100% passing
npx playwright test       # Playwright — 0 new failures
npx tsc --noEmit         # TypeScript — 0 errors
```

---

## Vitest

### Config location
`vitest.config.ts` in project root.

### Test file location
`tests/unit/` — unit and API route tests

### Running
```bash
npm test                  # run all
npm test -- --watch      # watch mode
npm test -- wave-dates   # run specific file
```

### Mocking Supabase in unit tests
```typescript
import { vi } from 'vitest'

vi.mock('@/lib/supabase/server', () => ({
  createServiceClient: vi.fn(() => ({
    from: vi.fn(() => ({
      select: vi.fn().mockReturnThis(),
      eq: vi.fn().mockReturnThis(),
      single: vi.fn().mockResolvedValue({ data: mockData, error: null })
    }))
  }))
}))
```

### Test structure
```typescript
import { describe, it, expect, beforeEach } from 'vitest'

describe('featureName', () => {
  beforeEach(() => {
    // reset mocks
  })

  it('should do something specific', () => {
    expect(result).toBe(expected)
  })
})
```

---

## Playwright E2E

### Config location
`playwright.config.ts` in project root

### Test file location
`tests/e2e/` — all E2E spec files

### Auth setup
`tests/e2e/auth.setup.ts` — runs once before all tests, logs in and upgrades test user to admin role via service role key. This file must exist and run first.

### Running
```bash
npx playwright test                    # run all
npx playwright test vendors            # run specific spec
npx playwright test --ui               # interactive UI mode
npx playwright test --headed           # visible browser
npx playwright show-report             # view last report
```

### Standard test structure
```typescript
import { test, expect } from '@playwright/test'

test.describe('Feature Name', () => {
  test('should perform action', async ({ page }) => {
    await page.goto('/contracts')
    await page.getByRole('button', { name: 'Add Contract' }).click()
    await expect(page.getByRole('dialog')).toBeVisible()
  })
})
```

### Selector best practices

**Always use exact matching for text selectors** to prevent substring collisions:
```typescript
// BAD - "Delete" matches "Delete vendor", "Dismiss" matches "Dismissed"
page.getByText('Delete')
page.getByText('Dismiss')

// GOOD - exact match only
page.getByText('Delete', { exact: true })
page.getByRole('button', { name: 'Delete', exact: true })
```

**Prefer role selectors over text**:
```typescript
// Better
page.getByRole('button', { name: 'Add Vendor' })
page.getByRole('dialog')
page.getByRole('heading', { name: 'Vendors' })

// Avoid when role selectors work
page.getByText('Add Vendor')
```

**For inputs use label or placeholder**:
```typescript
page.getByLabel('Contract Name')
page.getByPlaceholder('e.g. ConnectWise Manage', { exact: true })
```

**When multiple elements match use .first() or .nth()**:
```typescript
page.getByText('Edit').first()
page.getByRole('row').nth(2)
```

### Waiting for async operations
```typescript
// Wait for navigation
await page.waitForURL('/contracts')

// Wait for element to appear
await expect(page.getByText('Contract saved')).toBeVisible()

// Wait for element to disappear
await expect(page.getByRole('dialog')).not.toBeVisible()

// Wait for network request
await page.waitForResponse(resp => resp.url().includes('/api/contracts'))
```

### Skipping data-dependent tests
```typescript
test('should show renewal modal', async ({ page }) => {
  // Skip if no contracts with auto_renews in DB
  const hasContract = await page.locator('[data-testid="contract-row"]').count()
  if (hasContract === 0) {
    test.skip(true, 'No contracts in database')
    return
  }
  // ... test
})
```

### The 3 permanently skipped tests
These tests in `renewal-workflow.spec.ts` and `child-records.spec.ts` skip when no matching contracts are found in the database. This is expected behavior — not failures. Target: 0 failing, some skipped is acceptable.

### Test naming convention
```
tests/e2e/
  auth.setup.ts              # auth setup — runs first always
  vendors.spec.ts            # vendor CRUD tests
  contracts.spec.ts          # contract CRUD tests
  child-records.spec.ts      # child record tests
  renewal-workflow.spec.ts   # renewal modal tests
  permissions.spec.ts        # permissions engine tests
  comments.spec.ts           # comment and mention tests
  finance-tasks.spec.ts      # finance task tests
  dashboard.spec.ts          # dashboard tests
  reports.spec.ts            # reports tests
  admin.spec.ts              # admin pages tests
  qa-[feature].spec.ts       # QA pass tests per feature
```

### Common Playwright errors and fixes

| Error | Fix |
|-------|-----|
| Strict mode violation — multiple elements | Add `.first()` or use more specific selector |
| Element not found | Add `await expect(element).toBeVisible()` before interacting |
| Test user is viewer not admin | Check `auth.setup.ts` is upgrading role correctly |
| Race condition in parallel workers | Use HTTP-level auth test not DB-state modification |
| `getByPlaceholder` case sensitive | Add `{ exact: true }` flag |

---

## TypeScript

### Run type check
```bash
npx tsc --noEmit
```

### Common strict mode fixes

**Optional fields in Zod schemas** — always use `.nullable().optional()` not just `.optional()` for text fields that come from forms:
```typescript
// WRONG - doesn't accept null from form submission
website: z.string().optional()

// CORRECT - accepts string | null | undefined
website: z.string().nullable().optional()
```

**Supabase query return types**:
```typescript
// Always type the expected shape
const { data, error } = await supabase
  .from('contracts')
  .select('id, contract_name, vendor:vendors(name)')
  .returns<ContractWithVendor[]>()
```

**No any types** — if you need to escape the type system use `unknown` and narrow:
```typescript
// BAD
const data = response as any

// GOOD
const data = response as unknown as MyExpectedType
```

---

## Test Coverage Targets

| Area | Vitest | Playwright |
|------|--------|------------|
| API route validation | ✅ Required | — |
| Wave date calculations | ✅ Required | — |
| Permission engine logic | ✅ Required | — |
| Cost rollup calculations | ✅ Required | — |
| CRUD operations | — | ✅ Required |
| Form validation errors | — | ✅ Required |
| Role-based access | — | ✅ Required |
| Navigation flows | — | ✅ Required |
