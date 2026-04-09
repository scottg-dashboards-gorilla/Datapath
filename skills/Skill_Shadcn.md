# shadcn/ui Skill — Datapath VCM

## Version and Setup
- shadcn/ui with Default style, Slate color scheme
- Radix UI primitives underneath
- Tailwind CSS for styling
- All components in `components/ui/`

---

## Installed Components

The following shadcn components are already installed — never reinstall:
```
button, input, label, select, dialog, sheet, table, badge,
card, tabs, toast, dropdown-menu, popover, command, avatar,
form, textarea, checkbox, switch, separator, skeleton,
alert, alert-dialog, scroll-area, tooltip
```

To add a new component:
```bash
npx shadcn@latest add [component-name]
```

---

## Color System — Emerald Green Accent

This project uses **emerald green** as the primary accent color for interactive elements, overriding shadcn's default blue. This applies to:
- Searchable combobox trigger buttons
- Selected option highlights in all dropdowns
- Hover states in select menus
- Checkmarks on selected items

```css
/* app/globals.css */
:root {
  --accent: 152 81% 36%;           /* emerald-600 */
  --accent-foreground: 0 0% 100%;  /* white */
}
```

For explicit emerald classes use:
- `bg-emerald-500` — primary green fill
- `bg-emerald-600` — darker green (hover state)
- `text-emerald-600` — green text
- `hover:bg-emerald-500 hover:text-white` — hover pattern

---

## Component Patterns

### Button variants
```typescript
// Primary action
<Button variant="default">Save Contract</Button>

// Secondary/cancel
<Button variant="outline">Cancel</Button>

// Destructive
<Button variant="destructive">Delete</Button>

// Ghost (subtle)
<Button variant="ghost">Edit</Button>

// Icon button
<Button variant="ghost" size="icon">
  <Pencil className="h-4 w-4" />
</Button>
```

### Dialog (Modal)
```typescript
import { Dialog, DialogContent, DialogHeader, DialogTitle, DialogFooter } from '@/components/ui/dialog'

<Dialog open={open} onOpenChange={setOpen}>
  <DialogContent className="max-w-2xl">
    <DialogHeader>
      <DialogTitle>Add Contract</DialogTitle>
    </DialogHeader>
    {/* content */}
    <DialogFooter>
      <Button variant="outline" onClick={() => setOpen(false)}>Cancel</Button>
      <Button onClick={handleSave}>Save</Button>
    </DialogFooter>
  </DialogContent>
</Dialog>
```

### Sheet (Slide-out panel)
```typescript
import { Sheet, SheetContent, SheetHeader, SheetTitle } from '@/components/ui/sheet'

<Sheet open={open} onOpenChange={setOpen}>
  <SheetContent side="right" className="w-[480px]">
    <SheetHeader>
      <SheetTitle>Contract Details</SheetTitle>
    </SheetHeader>
    {/* content */}
  </SheetContent>
</Sheet>
```

### Card
```typescript
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card'

<Card>
  <CardHeader>
    <CardTitle>Billing</CardTitle>
  </CardHeader>
  <CardContent className="space-y-4">
    {/* content */}
  </CardContent>
</Card>
```

### Tabs
```typescript
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs'

<Tabs defaultValue="overview">
  <TabsList>
    <TabsTrigger value="overview">Overview</TabsTrigger>
    <TabsTrigger value="history">History</TabsTrigger>
    <TabsTrigger value="documents">Documents</TabsTrigger>
  </TabsList>
  <TabsContent value="overview">...</TabsContent>
  <TabsContent value="history">...</TabsContent>
  <TabsContent value="documents">...</TabsContent>
</Tabs>
```

### Badge
```typescript
import { Badge } from '@/components/ui/badge'

// Health score badges
<Badge variant="destructive">Action Required</Badge>  // red
<Badge className="bg-amber-100 text-amber-800">Review Soon</Badge>  // yellow
<Badge className="bg-green-100 text-green-800">On Track</Badge>  // green

// Role badges
<Badge variant="outline">Admin</Badge>
<Badge variant="secondary">Manager</Badge>
```

### Toast notifications
```typescript
import { useToast } from '@/components/ui/use-toast'

const { toast } = useToast()

// Success
toast({ title: 'Contract saved', description: 'Changes have been saved.' })

// Error
toast({ title: 'Error', description: 'Failed to save.', variant: 'destructive' })
```

### Form with react-hook-form
```typescript
import { Form, FormControl, FormField, FormItem, FormLabel, FormMessage } from '@/components/ui/form'

<Form {...form}>
  <form onSubmit={form.handleSubmit(onSubmit)}>
    <FormField
      control={form.control}
      name="contract_name"
      render={({ field }) => (
        <FormItem>
          <FormLabel>Contract Name</FormLabel>
          <FormControl>
            <Input {...field} placeholder="e.g. ConnectWise Manage Annual" />
          </FormControl>
          <FormMessage />  {/* shows Zod validation error */}
        </FormItem>
      )}
    />
  </form>
</Form>
```

### Select (simple dropdown — Type B)
```typescript
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select'

<Select value={value} onValueChange={setValue}>
  <SelectTrigger>
    <SelectValue placeholder="Select department" />
  </SelectTrigger>
  <SelectContent>
    <SelectItem value="finance">Finance</SelectItem>
    <SelectItem value="engineering">Engineering</SelectItem>
  </SelectContent>
</Select>
```

### SearchableCombobox (Type A — custom component)
```typescript
import { SearchableCombobox } from '@/components/ui/searchable-combobox'

<SearchableCombobox
  options={vendors.map(v => ({ value: v.id, label: v.name }))}
  value={vendorId}
  onChange={setVendorId}
  placeholder="Select vendor"
  searchPlaceholder="Search vendors..."
  allowCustomEntry={true}
  onAddCustom={async (name) => {
    // create vendor inline
  }}
/>
```

### Table
```typescript
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table'

<Table>
  <TableHeader>
    <TableRow>
      <TableHead>Contract</TableHead>
      <TableHead>Vendor</TableHead>
      <TableHead>Deadline</TableHead>
    </TableRow>
  </TableHeader>
  <TableBody>
    {contracts.map(contract => (
      <TableRow key={contract.id} className="hover:bg-gray-50">
        <TableCell>{contract.contract_name}</TableCell>
        <TableCell>{contract.vendor?.name}</TableCell>
        <TableCell>{contract.cancellation_deadline}</TableCell>
      </TableRow>
    ))}
  </TableBody>
</Table>
```

### Skeleton (loading state)
```typescript
import { Skeleton } from '@/components/ui/skeleton'

// Always show skeleton while loading, never blank flash
{isLoading ? (
  <div className="space-y-3">
    <Skeleton className="h-12 w-full" />
    <Skeleton className="h-12 w-full" />
    <Skeleton className="h-12 w-3/4" />
  </div>
) : (
  <ContractList contracts={contracts} />
)}
```

---

## Design System Constants

```typescript
// Health score colors (use consistently everywhere)
const healthColors = {
  red: 'bg-red-100 text-red-800 border-red-200',
  yellow: 'bg-amber-100 text-amber-800 border-amber-200',
  green: 'bg-green-100 text-green-800 border-green-200'
}

// Sidebar colors
const sidebar = {
  bg: '#0F1117',
  text: '#9CA3AF',
  activeText: '#FFFFFF',
  activeBg: '#1F2937'
}

// Page layout
const layout = {
  contentBg: '#F8F9FB',
  cardBg: '#FFFFFF',
  border: '#E5E7EB'
}
```

---

## Spacing and Layout Rules

- Card padding: `p-6` always
- Section spacing: `space-y-6` between major sections
- Form field spacing: `space-y-4` between form fields
- Button groups: `flex gap-2`
- Page header pattern:
```typescript
<div className="flex items-center justify-between mb-6">
  <div>
    <h1 className="text-2xl font-bold text-gray-900">Contracts</h1>
    <p className="text-gray-500 mt-1">Manage your vendor contracts</p>
  </div>
  <Button>+ Add Contract</Button>
</div>
```

---

## Empty States

Every list/table must have an empty state:
```typescript
{contracts.length === 0 ? (
  <div className="text-center py-12">
    <FileText className="h-12 w-12 text-gray-300 mx-auto mb-4" />
    <h3 className="text-lg font-medium text-gray-900">No contracts yet</h3>
    <p className="text-gray-500 mt-1">Add your first contract to get started</p>
    <Button className="mt-4">+ Add Contract</Button>
  </div>
) : (
  <ContractTable contracts={contracts} />
)}
```

---

## Icons

Using `lucide-react` — already installed:
```typescript
import { 
  FileText, Building2, AlertCircle, CheckCircle, 
  Clock, ChevronRight, Pencil, Trash2, Plus,
  Search, Bell, Settings, Users, BarChart3,
  Download, Upload, Eye, Lock, Unlock
} from 'lucide-react'

// Standard sizes
<Icon className="h-4 w-4" />   // inline with text
<Icon className="h-5 w-5" />   // buttons
<Icon className="h-6 w-6" />   // section headers
<Icon className="h-12 w-12" /> // empty states
```

---

## Accessibility

- Always include `aria-label` on icon-only buttons
- Use `role="alert"` on error messages
- Ensure all form inputs have associated labels via `htmlFor` or `FormLabel`
- Dialog titles must always be present (use `DialogTitle`)
- Color is never the only indicator — always pair with text or icon
