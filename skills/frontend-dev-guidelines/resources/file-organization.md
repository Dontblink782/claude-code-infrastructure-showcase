# File Organization

Proper file and directory structure for maintainable, scalable Next.js 15 App Router applications.

---

## Next.js 15 App Router Structure

### Root Directory Layout

```
project-root/
├── app/                          # Next.js App Router
│   ├── (auth)/                   # Route groups (auth pages)
│   │   ├── login/
│   │   │   └── page.tsx
│   │   └── signup/
│   │       └── page.tsx
│   ├── orders/                   # Order routes
│   │   ├── page.tsx              # /orders
│   │   ├── loading.tsx           # Loading UI
│   │   ├── error.tsx             # Error UI
│   │   └── [id]/
│   │       └── page.tsx          # /orders/[id]
│   ├── dashboard/
│   │   └── page.tsx
│   ├── layout.tsx                # Root layout
│   └── page.tsx                  # Home page
│
├── actions/                      # Server Actions (flat structure)
│   ├── orders.ts                 # Order mutations/fetches
│   ├── customers.ts              # Customer operations
│   └── auth.ts                   # Authentication actions
│
├── features/                     # Domain-specific features
│   ├── orders/
│   │   ├── components/           # Order-specific components
│   │   │   ├── OrdersTable.tsx
│   │   │   ├── OrdersSearch.tsx
│   │   │   └── badges/
│   │   │       └── OrderPaymentBadge.tsx
│   │   └── hooks/                # Order-specific hooks
│   │       ├── useSearchOrders.ts
│   │       └── useUpdateOrder.ts
│   ├── customers/
│   └── dashboard/
│
├── components/                   # Shared components (3+ uses)
│   ├── ui/                       # ShadCN base components
│   │   ├── button.tsx
│   │   ├── badge.tsx
│   │   ├── card.tsx
│   │   └── table.tsx
│   └── domain/                   # Shared domain wrappers
│       └── badges/
│           └── PaymentStatusBadge.tsx
│
├── lib/                          # Utilities and configuration
│   ├── db.ts                     # Prisma client
│   ├── utils.ts                  # cn() helper, etc.
│   └── validations/              # Zod schemas
│       └── order.ts
│
├── types/                        # TypeScript types
│   ├── order.ts
│   ├── customer.ts
│   └── index.ts
│
└── public/                       # Static assets
    └── images/
```

---

## app/ Directory (Routes)

### Route Structure

**Next.js App Router uses file-system based routing:**

```
app/
├── page.tsx              → /
├── about/
│   └── page.tsx          → /about
├── orders/
│   ├── page.tsx          → /orders
│   └── [id]/
│       └── page.tsx      → /orders/123
└── dashboard/
    ├── page.tsx          → /dashboard
    └── settings/
        └── page.tsx      → /dashboard/settings
```

### Special Files

**Next.js reserves specific filenames:**

| File | Purpose | Example |
|------|---------|---------|
| `page.tsx` | Route UI | `/orders` page |
| `layout.tsx` | Shared layout | Wraps pages |
| `loading.tsx` | Loading UI | Suspense fallback |
| `error.tsx` | Error UI | Error boundary |
| `not-found.tsx` | 404 page | Custom 404 |
| `route.ts` | API endpoint | API route handler |

### Route Groups

**Use parentheses for organization without URL segments:**

```
app/
├── (marketing)/          # NOT in URL
│   ├── about/
│   │   └── page.tsx      → /about
│   └── contact/
│       └── page.tsx      → /contact
└── (dashboard)/          # NOT in URL
    ├── orders/
    │   └── page.tsx      → /orders
    └── customers/
        └── page.tsx      → /customers
```

**Benefits:**
- Organize routes logically
- Apply different layouts
- No impact on URL structure

---

## actions/ Directory

### Server Actions Structure

**Flat structure** - one file per domain:

```
actions/
├── orders.ts             # All order operations
├── customers.ts          # All customer operations
├── products.ts           # All product operations
└── auth.ts               # Authentication operations
```

### Server Action File Pattern

```typescript
// actions/orders.ts
'use server';

import { prisma } from '@/lib/db';
import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';

// Fetch operations
export async function getOrders(filters?: { status?: string }) {
    return await prisma.order.findMany({
        where: filters?.status ? { status: filters.status } : undefined,
        include: { customer: true },
        orderBy: { createdAt: 'desc' },
    });
}

export async function getOrder(id: string) {
    const order = await prisma.order.findUnique({
        where: { id },
        include: { customer: true, items: true },
    });

    if (!order) throw new Error('Order not found');
    return order;
}

// Mutation operations
export async function createOrder(formData: FormData) {
    const order = await prisma.order.create({
        data: {
            customerId: formData.get('customerId') as string,
            amount: parseFloat(formData.get('amount') as string),
        },
    });

    revalidatePath('/orders');
    redirect(`/orders/${order.id}`);
}

export async function updateOrder(id: string, formData: FormData) {
    await prisma.order.update({
        where: { id },
        data: { status: formData.get('status') as string },
    });

    revalidatePath('/orders');
    revalidatePath(`/orders/${id}`);
}

export async function deleteOrder(id: string) {
    await prisma.order.delete({ where: { id } });
    revalidatePath('/orders');
}

// Search/filter operations
export async function searchOrders(query: string) {
    return await prisma.order.findMany({
        where: {
            OR: [
                { id: { contains: query, mode: 'insensitive' } },
                { customerName: { contains: query, mode: 'insensitive' } },
            ],
        },
        take: 20,
    });
}
```

**Key Points:**
- `'use server'` at top of file
- Group by domain (orders, customers, etc.)
- Export individual functions
- Include fetches AND mutations
- Revalidation after mutations

---

## features/ Directory

### Purpose

**Domain-specific components and logic** - start here, move to `components/` when used 3+ times.

### Feature Structure

```
features/
  orders/
    components/          # Order-specific UI components
      OrdersTable.tsx
      OrdersSearch.tsx
      OrderCard.tsx
      badges/            # Subdirectory for related components
        OrderPaymentBadge.tsx
        OrderStatusBadge.tsx
      buttons/
        OrderActionButton.tsx
    hooks/               # Order-specific client hooks
      useSearchOrders.ts
      useUpdateOrder.ts
      useOrderFilters.ts
    helpers/             # Utility functions (optional)
      orderHelpers.ts
    types/               # Feature-specific types (optional)
      index.ts
```

### Component Subdirectories

**Organize by component type when >5 components:**

```
features/orders/components/
├── OrdersTable.tsx       # Main components (flat)
├── OrdersSearch.tsx
├── badges/               # Group related components
│   ├── OrderPaymentBadge.tsx
│   └── OrderStatusBadge.tsx
├── buttons/
│   └── OrderActionButton.tsx
└── cards/
    └── OrderCard.tsx
```

### When to Create a New Feature

**Create new feature when:**
- Multiple related components (3+)
- Domain-specific logic
- Custom hooks needed
- Will grow over time

**Examples:**
- `features/orders/` - Order management
- `features/customers/` - Customer operations
- `features/dashboard/` - Dashboard widgets
- `features/auth/` - Authentication flows

---

## components/ Directory

### Purpose

**Truly reusable components** used across 3+ features.

### Structure

```
components/
├── ui/                      # ShadCN base components
│   ├── button.tsx
│   ├── badge.tsx
│   ├── card.tsx
│   ├── table.tsx
│   ├── input.tsx
│   └── dialog.tsx
└── domain/                  # Shared domain wrappers
    ├── badges/
    │   └── PaymentStatusBadge.tsx
    ├── cards/
    │   └── StatCard.tsx
    └── layouts/
        └── PageHeader.tsx
```

### components/ui/ (ShadCN)

**Generated by ShadCN CLI** - DO NOT modify structure:

```bash
npx shadcn-ui@latest add button
npx shadcn-ui@latest add badge
npx shadcn-ui@latest add card
```

Creates:
```
components/ui/
├── button.tsx
├── badge.tsx
└── card.tsx
```

**Never import base components directly** - use wrappers instead.

### components/domain/ (Wrappers)

**Shared domain wrappers** moved from features/ when used 3+ times:

```typescript
// components/domain/badges/PaymentStatusBadge.tsx
import { Badge } from '@/components/ui/badge';

interface PaymentStatusBadgeProps {
    status: 'paid' | 'unpaid' | 'pending';
}

export function PaymentStatusBadge({ status }: PaymentStatusBadgeProps) {
    const config = {
        paid: { label: 'Paid', className: 'bg-green-500 text-white' },
        unpaid: { label: 'Unpaid', className: 'bg-red-500 text-white' },
        pending: { label: 'Pending', className: 'bg-yellow-500 text-white' },
    };

    const { label, className } = config[status];
    return <Badge className={className}>{label}</Badge>;
}
```

---

## Hybrid Location Strategy

### The 3+ Rule

**Components start in features/, move to components/ when used 3+ times.**

**Example - OrderPaymentBadge evolution:**

**Step 1: Created in features/ (single use)**
```
features/orders/components/badges/OrderPaymentBadge.tsx
```

**Step 2: Used in 3 places**
- Orders page
- Invoices page
- Payment history page

**Step 3: Move to components/domain/**
```
components/domain/badges/PaymentStatusBadge.tsx
```

**Step 4: Update imports**
```typescript
// Before (feature-specific)
import { OrderPaymentBadge } from '@/features/orders/components/badges/OrderPaymentBadge';

// After (shared)
import { PaymentStatusBadge } from '@/components/domain/badges/PaymentStatusBadge';
```

### Migration Checklist

When moving component to shared:

- [ ] Copy component to `components/domain/`
- [ ] Rename if needed (OrderPaymentBadge → PaymentStatusBadge)
- [ ] Update all import paths
- [ ] Delete original from features/
- [ ] Test all usages

---

## Import Aliases

### Next.js Default: @/

**Next.js configures `@/` to resolve to project root by default.**

**tsconfig.json:**
```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./*"]
    }
  }
}
```

### Usage Examples

```typescript
// Actions
import { getOrders } from '@/actions/orders';

// Features
import { OrdersTable } from '@/features/orders/components/OrdersTable';
import { useSearchOrders } from '@/features/orders/hooks/useSearchOrders';

// Shared Components
import { Button } from '@/components/ui/button';
import { PaymentStatusBadge } from '@/components/domain/badges/PaymentStatusBadge';

// Lib
import { prisma } from '@/lib/db';
import { cn } from '@/lib/utils';

// Types
import type { Order } from '@/types/order';
```

### Why Single Alias?

**Simplicity:**
- One alias to remember
- Consistent across project
- Next.js convention
- Easy to refactor

**vs Multiple Aliases:**
```typescript
// ❌ AVOID - Multiple aliases (old Vite pattern)
import type { Order } from '~types/order';
import { Button } from '~components/ui/button';
import { getOrders } from '~actions/orders';

// ✅ PREFER - Single @/ alias
import type { Order } from '@/types/order';
import { Button } from '@/components/ui/button';
import { getOrders } from '@/actions/orders';
```

---

## File Naming Conventions

### Components

**PascalCase with `.tsx` extension:**

```
OrdersTable.tsx
PaymentStatusBadge.tsx
OrderActionButton.tsx
```

### Hooks

**camelCase with `use` prefix, `.ts` extension:**

```
useSearchOrders.ts
useUpdateOrder.ts
useOrderFilters.ts
```

### Server Actions

**camelCase with `.ts` extension:**

```
orders.ts
customers.ts
auth.ts
```

### Utilities

**camelCase with `.ts` extension:**

```
utils.ts
validations.ts
helpers.ts
```

### Types

**camelCase, `.ts` extension:**

```
order.ts
customer.ts
index.ts
```

---

## Complete Feature Example

### Example: Orders Feature

**Full structure:**

```
project-root/
├── app/
│   └── orders/
│       ├── page.tsx                    # Server Component (container)
│       ├── loading.tsx                 # Loading UI
│       ├── error.tsx                   # Error UI
│       └── [id]/
│           └── page.tsx                # Order detail page
│
├── actions/
│   └── orders.ts                       # Server actions (fetch + mutations)
│
├── features/
│   └── orders/
│       ├── components/
│       │   ├── OrdersTable.tsx         # Main table (Client)
│       │   ├── OrdersSearch.tsx        # Search component (Client)
│       │   ├── OrderCard.tsx           # Card component
│       │   └── badges/
│       │       └── OrderPaymentBadge.tsx
│       └── hooks/
│           ├── useSearchOrders.ts      # Search hook
│           └── useUpdateOrder.ts       # Update hook
│
├── components/
│   └── ui/
│       ├── table.tsx                   # ShadCN base
│       ├── badge.tsx
│       └── button.tsx
│
├── lib/
│   └── validations/
│       └── order.ts                    # Zod schema
│
└── types/
    └── order.ts                        # Order types
```

**File contents:**

**1. app/orders/page.tsx (Server Component)**
```typescript
import { getOrders } from '@/actions/orders';
import { OrdersTable } from '@/features/orders/components/OrdersTable';

export default async function OrdersPage() {
    const orders = await getOrders();

    return (
        <div className="container mx-auto py-8">
            <h1 className="text-3xl font-bold mb-6">Orders</h1>
            <OrdersTable orders={orders} />
        </div>
    );
}
```

**2. actions/orders.ts (Server Actions)**
```typescript
'use server';

import { prisma } from '@/lib/db';
import { revalidatePath } from 'next/cache';

export async function getOrders() {
    return await prisma.order.findMany({
        include: { customer: true },
        orderBy: { createdAt: 'desc' },
    });
}

export async function searchOrders(query: string) {
    return await prisma.order.findMany({
        where: {
            OR: [
                { id: { contains: query, mode: 'insensitive' } },
                { customerName: { contains: query, mode: 'insensitive' } },
            ],
        },
        take: 20,
    });
}

export async function updateOrder(id: string, status: string) {
    await prisma.order.update({
        where: { id },
        data: { status },
    });

    revalidatePath('/orders');
}
```

**3. features/orders/components/OrdersTable.tsx (Client)**
```typescript
'use client';

import { useState } from 'react';
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table';
import { OrderPaymentBadge } from './badges/OrderPaymentBadge';
import type { Order } from '@/types/order';

interface OrdersTableProps {
    orders: Order[];
}

export function OrdersTable({ orders }: OrdersTableProps) {
    const [selectedId, setSelectedId] = useState<string | null>(null);

    return (
        <Table>
            <TableHeader>
                <TableRow>
                    <TableHead>Order ID</TableHead>
                    <TableHead>Customer</TableHead>
                    <TableHead>Status</TableHead>
                    <TableHead>Amount</TableHead>
                </TableRow>
            </TableHeader>
            <TableBody>
                {orders.map((order) => (
                    <TableRow key={order.id} onClick={() => setSelectedId(order.id)}>
                        <TableCell>{order.id}</TableCell>
                        <TableCell>{order.customerName}</TableCell>
                        <TableCell>
                            <OrderPaymentBadge status={order.paymentStatus} />
                        </TableCell>
                        <TableCell>${order.amount}</TableCell>
                    </TableRow>
                ))}
            </TableBody>
        </Table>
    );
}
```

**4. features/orders/hooks/useSearchOrders.ts**
```typescript
'use client';

import { useState } from 'react';
import { searchOrders } from '@/actions/orders';
import type { Order } from '@/types/order';

export function useSearchOrders() {
    const [results, setResults] = useState<Order[]>([]);
    const [isSearching, setIsSearching] = useState(false);

    async function search(query: string) {
        setIsSearching(true);
        try {
            const orders = await searchOrders(query);
            setResults(orders);
        } finally {
            setIsSearching(false);
        }
    }

    return { results, isSearching, search };
}
```

---

## Directory Structure Comparison

### Before (Vite + TanStack)

```
src/
├── features/
│   └── posts/
│       ├── api/                  # ❌ Axios API services
│       │   └── postApi.ts
│       ├── components/
│       ├── hooks/
│       │   └── useSuspensePost.ts  # ❌ TanStack Query
│       └── queries/              # ❌ Query key factories
│           └── postQueries.ts
├── components/
├── routes/                       # ❌ TanStack Router
│   └── posts/
│       └── index.tsx
└── lib/
    └── apiClient.ts              # ❌ Axios instance
```

### After (Next.js 15)

```
app/
├── orders/                       # ✅ Next.js routes
│   ├── page.tsx
│   └── loading.tsx
├── actions/                      # ✅ Server Actions
│   └── orders.ts
├── features/
│   └── orders/
│       ├── components/           # ✅ Client components
│       └── hooks/                # ✅ Client hooks
│           └── useSearchOrders.ts
├── components/
│   └── ui/                       # ✅ ShadCN
└── lib/
    └── db.ts                     # ✅ Prisma client
```

---

## Best Practices Summary

**Directory Organization:**
1. **app/** - Routes only (page.tsx, layout.tsx, etc.)
2. **actions/** - All server-side logic (fetch + mutations)
3. **features/** - Domain components (start here)
4. **components/** - Shared components (3+ uses)
5. **lib/** - Utilities, Prisma client, validations
6. **types/** - Shared TypeScript types

**File Naming:**
- Components: PascalCase.tsx
- Hooks: camelCase.ts with `use` prefix
- Actions: camelCase.ts (domain name)
- Types: camelCase.ts

**Import Alias:**
- Use `@/` for all imports
- Consistent, simple, Next.js convention

**Component Location:**
- Start in features/
- Move to components/ at 3+ uses
- ShadCN base stays in components/ui/

**See Also:**
- [component-patterns.md](component-patterns.md) - Server/Client patterns
- [data-fetching.md](data-fetching.md) - Server Actions usage
- [shadcn-patterns.md](shadcn-patterns.md) - Wrapper patterns
- [complete-examples.md](complete-examples.md) - Full examples
