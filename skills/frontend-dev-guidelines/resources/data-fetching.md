# Data Fetching Patterns

Modern Next.js 15 App Router data fetching using Server Components, Server Actions, and client-side hooks for mutations. Eliminate API routes in favor of direct database access and Server Actions.

---

## Primary Pattern: Server Components

### Server-Side Data Fetching (Default)

**Next.js 15 App Router makes Server Components the default data fetching pattern.** Fetch data directly in async Server Components - no API routes needed.

**Basic Pattern:**

```typescript
// app/orders/page.tsx
import { prisma } from '@/lib/db';
import { OrdersTable } from '@/features/orders/components/OrdersTable';

export default async function OrdersPage() {
    // Direct database access in Server Component
    const orders = await prisma.order.findMany({
        where: { status: 'active' },
        include: { customer: true },
        orderBy: { createdAt: 'desc' },
    });

    return (
        <div className="container">
            <h1>Active Orders</h1>
            <OrdersTable orders={orders} />
        </div>
    );
}
```

**Key Points:**
- Component is `async` function
- Fetch directly from database (Prisma)
- No API route needed
- Data passed as props to Client Components
- Automatic request deduplication

---

### Server Actions for Data Fetching

**Alternative pattern** - Create reusable server actions for data fetching:

```typescript
// actions/orders.ts
'use server';

import { prisma } from '@/lib/db';

export async function getOrders() {
    const orders = await prisma.order.findMany({
        where: { status: 'active' },
        include: { customer: true },
        orderBy: { createdAt: 'desc' },
    });

    return orders;
}

export async function getOrder(id: string) {
    const order = await prisma.order.findUnique({
        where: { id },
        include: { customer: true, items: true },
    });

    if (!order) {
        throw new Error('Order not found');
    }

    return order;
}
```

**Usage in Server Component:**

```typescript
// app/orders/page.tsx
import { getOrders } from '@/actions/orders';
import { OrdersTable } from '@/features/orders/components/OrdersTable';

export default async function OrdersPage() {
    const orders = await getOrders();

    return (
        <div className="container">
            <h1>Orders</h1>
            <OrdersTable orders={orders} />
        </div>
    );
}
```

**When to use Server Actions for fetching:**
- Need to reuse fetch logic across multiple pages
- Complex query logic (20+ lines)
- Want separation of concerns
- Testing isolation

**When to fetch directly in page:**
- Simple query (<10 lines)
- Single use case
- Co-located with component

---

### Parallel Data Fetching

**Use `Promise.all()` to fetch multiple resources in parallel:**

```typescript
// app/dashboard/page.tsx
import { getStats } from '@/actions/stats';
import { getOrders } from '@/actions/orders';
import { getCustomers } from '@/actions/customers';

export default async function DashboardPage() {
    // Fetch all data in parallel - total time = slowest query
    const [stats, orders, customers] = await Promise.all([
        getStats(),
        getOrders({ limit: 10 }),
        getCustomers({ limit: 10 }),
    ]);

    return (
        <div className="grid grid-cols-3 gap-4">
            <StatsCard data={stats} />
            <OrdersList orders={orders} />
            <CustomersList customers={customers} />
        </div>
    );
}
```

**Benefits:**
- Executes queries concurrently
- Total time = slowest query (not sum of all)
- Type-safe with array destructuring

---

### Streaming with Suspense

**For better UX, stream components independently as data arrives:**

```typescript
// app/dashboard/page.tsx
import { Suspense } from 'react';
import { StatsSection } from '@/features/dashboard/components/StatsSection';
import { OrdersSection } from '@/features/dashboard/components/OrdersSection';
import { CustomersSection } from '@/features/dashboard/components/CustomersSection';
import { Skeleton } from '@/components/ui/skeleton';

export default function DashboardPage() {
    return (
        <div className="grid grid-cols-3 gap-4">
            {/* Each section streams independently */}
            <Suspense fallback={<Skeleton className="h-32" />}>
                <StatsSection />
            </Suspense>

            <Suspense fallback={<Skeleton className="h-64" />}>
                <OrdersSection />
            </Suspense>

            <Suspense fallback={<Skeleton className="h-64" />}>
                <CustomersSection />
            </Suspense>
        </div>
    );
}

// features/dashboard/components/OrdersSection.tsx
import { getOrders } from '@/actions/orders';
import { OrdersList } from './OrdersList';

export async function OrdersSection() {
    const orders = await getOrders({ limit: 10 });
    return <OrdersList orders={orders} />;
}
```

**When to use:**
- Independent data sources
- Different fetch speeds
- Better perceived performance
- Show content as it loads

**When to use Promise.all:**
- Data needed together
- Render at once
- All data fast
- Related queries

---

## fetch API with Caching

### Using fetch in Server Components

**For external APIs, use `fetch` with built-in caching:**

```typescript
// app/weather/page.tsx
export default async function WeatherPage() {
    // Cached for 1 hour
    const res = await fetch('https://api.weather.com/forecast', {
        next: { revalidate: 3600 },
    });

    const weather = await res.json();

    return <WeatherDisplay data={weather} />;
}
```

### Caching Strategies

**Static (cached indefinitely):**
```typescript
const res = await fetch('https://api.example.com/data', {
    cache: 'force-cache', // Default
});
```

**Revalidate after time:**
```typescript
const res = await fetch('https://api.example.com/data', {
    next: { revalidate: 60 }, // Revalidate every 60 seconds
});
```

**No caching (always fresh):**
```typescript
const res = await fetch('https://api.example.com/data', {
    cache: 'no-store', // Opt out of caching
});
```

**Tag-based revalidation:**
```typescript
const res = await fetch('https://api.example.com/data', {
    next: { tags: ['orders'] },
});

// Later, revalidate by tag:
// revalidateTag('orders');
```

---

## Server Actions for Mutations

### Form Submissions

**Primary pattern for creating/updating data:**

```typescript
// actions/orders.ts
'use server';

import { prisma } from '@/lib/db';
import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';

export async function createOrder(formData: FormData) {
    // Extract form data
    const customerId = formData.get('customerId') as string;
    const amount = parseFloat(formData.get('amount') as string);

    // Validate (use Zod for production)
    if (!customerId || !amount) {
        throw new Error('Missing required fields');
    }

    // Create order
    const order = await prisma.order.create({
        data: {
            customerId,
            amount,
            status: 'pending',
        },
    });

    // Revalidate affected pages
    revalidatePath('/orders');

    // Redirect to new order
    redirect(`/orders/${order.id}`);
}
```

**Usage in Client Component:**

```typescript
// features/orders/components/CreateOrderForm.tsx
'use client';

import { createOrder } from '@/actions/orders';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';

export function CreateOrderForm() {
    return (
        <form action={createOrder} className="space-y-4">
            <Input name="customerId" placeholder="Customer ID" required />
            <Input name="amount" type="number" placeholder="Amount" required />
            <Button type="submit">Create Order</Button>
        </form>
    );
}
```

**Benefits:**
- Progressive enhancement (works without JS)
- Type-safe with TypeScript
- Direct database access
- Automatic revalidation

---

### Server Actions with useTransition

**For better UX, use `useTransition` to show pending state:**

```typescript
// features/orders/components/CreateOrderForm.tsx
'use client';

import { useTransition } from 'react';
import { createOrder } from '@/actions/orders';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';

export function CreateOrderForm() {
    const [isPending, startTransition] = useTransition();

    async function handleSubmit(formData: FormData) {
        startTransition(async () => {
            await createOrder(formData);
        });
    }

    return (
        <form action={handleSubmit} className="space-y-4">
            <Input name="customerId" placeholder="Customer ID" required />
            <Input name="amount" type="number" placeholder="Amount" required />
            <Button type="submit" disabled={isPending}>
                {isPending ? 'Creating...' : 'Create Order'}
            </Button>
        </form>
    );
}
```

---

### Update/Delete Mutations

**Bind arguments to server actions:**

```typescript
// actions/orders.ts
'use server';

import { prisma } from '@/lib/db';
import { revalidatePath } from 'next/cache';

export async function updateOrder(id: string, formData: FormData) {
    const status = formData.get('status') as string;

    await prisma.order.update({
        where: { id },
        data: { status },
    });

    revalidatePath('/orders');
    revalidatePath(`/orders/${id}`);
}

export async function deleteOrder(id: string) {
    await prisma.order.delete({
        where: { id },
    });

    revalidatePath('/orders');
}
```

**Usage with .bind():**

```typescript
// features/orders/components/OrderActions.tsx
'use client';

import { updateOrder, deleteOrder } from '@/actions/orders';
import { Button } from '@/components/ui/button';

interface OrderActionsProps {
    orderId: string;
}

export function OrderActions({ orderId }: OrderActionsProps) {
    // Bind orderId to server actions
    const updateWithId = updateOrder.bind(null, orderId);
    const deleteWithId = deleteOrder.bind(null, orderId);

    return (
        <div className="space-x-2">
            <form action={updateWithId} className="inline">
                <input type="hidden" name="status" value="completed" />
                <Button type="submit">Complete</Button>
            </form>

            <form action={deleteWithId} className="inline">
                <Button type="submit" variant="destructive">
                    Delete
                </Button>
            </form>
        </div>
    );
}
```

---

## Client-Side Data Fetching

### Custom Hooks for Search/Filtering

**When users need dynamic queries (search, filter), use client-side hooks:**

```typescript
// features/orders/hooks/useSearchOrders.ts
'use client';

import { useState } from 'react';
import { searchOrders } from '@/actions/orders';
import type { Order } from '@/types/order';

export function useSearchOrders() {
    const [results, setResults] = useState<Order[]>([]);
    const [isSearching, setIsSearching] = useState(false);
    const [error, setError] = useState<string | null>(null);

    async function search(query: string) {
        setIsSearching(true);
        setError(null);

        try {
            const orders = await searchOrders(query);
            setResults(orders);
        } catch (err) {
            setError(err instanceof Error ? err.message : 'Search failed');
        } finally {
            setIsSearching(false);
        }
    }

    return {
        results,
        isSearching,
        error,
        search,
    };
}
```

**Server Action for search:**

```typescript
// actions/orders.ts
'use server';

import { prisma } from '@/lib/db';

export async function searchOrders(query: string) {
    const orders = await prisma.order.findMany({
        where: {
            OR: [
                { id: { contains: query, mode: 'insensitive' } },
                { customerName: { contains: query, mode: 'insensitive' } },
            ],
        },
        take: 20,
    });

    return orders;
}
```

**Usage in Component:**

```typescript
// features/orders/components/OrdersSearch.tsx
'use client';

import { useState } from 'react';
import { useSearchOrders } from '../hooks/useSearchOrders';
import { Input } from '@/components/ui/input';
import { Button } from '@/components/ui/button';

export function OrdersSearch() {
    const [query, setQuery] = useState('');
    const { results, isSearching, error, search } = useSearchOrders();

    async function handleSearch(e: React.FormEvent) {
        e.preventDefault();
        await search(query);
    }

    return (
        <div className="space-y-4">
            <form onSubmit={handleSearch} className="flex gap-2">
                <Input
                    value={query}
                    onChange={(e) => setQuery(e.target.value)}
                    placeholder="Search orders..."
                />
                <Button type="submit" disabled={isSearching}>
                    {isSearching ? 'Searching...' : 'Search'}
                </Button>
            </form>

            {error && <p className="text-red-500">{error}</p>}

            {results.length > 0 && (
                <div>
                    <h3>Results ({results.length})</h3>
                    {results.map((order) => (
                        <div key={order.id}>{order.id}</div>
                    ))}
                </div>
            )}
        </div>
    );
}
```

---

### Hook Pattern for Mutations

**Client-side mutations with loading/error states:**

```typescript
// features/orders/hooks/useUpdateOrder.ts
'use client';

import { useState } from 'react';
import { updateOrderStatus } from '@/actions/orders';
import { useRouter } from 'next/navigation';

export function useUpdateOrder() {
    const [isUpdating, setIsUpdating] = useState(false);
    const [error, setError] = useState<string | null>(null);
    const router = useRouter();

    async function update(orderId: string, status: string) {
        setIsUpdating(true);
        setError(null);

        try {
            await updateOrderStatus(orderId, status);
            router.refresh(); // Trigger Server Component re-fetch
        } catch (err) {
            setError(err instanceof Error ? err.message : 'Update failed');
        } finally {
            setIsUpdating(false);
        }
    }

    return {
        update,
        isUpdating,
        error,
    };
}
```

**Server Action:**

```typescript
// actions/orders.ts
'use server';

import { prisma } from '@/lib/db';
import { revalidatePath } from 'next/cache';

export async function updateOrderStatus(id: string, status: string) {
    await prisma.order.update({
        where: { id },
        data: { status },
    });

    revalidatePath('/orders');
    revalidatePath(`/orders/${id}`);
}
```

**Usage:**

```typescript
// features/orders/components/OrderStatusButton.tsx
'use client';

import { useUpdateOrder } from '../hooks/useUpdateOrder';
import { Button } from '@/components/ui/button';

interface OrderStatusButtonProps {
    orderId: string;
}

export function OrderStatusButton({ orderId }: OrderStatusButtonProps) {
    const { update, isUpdating, error } = useUpdateOrder();

    return (
        <div>
            <Button
                onClick={() => update(orderId, 'completed')}
                disabled={isUpdating}
            >
                {isUpdating ? 'Updating...' : 'Mark Complete'}
            </Button>
            {error && <p className="text-red-500 text-sm">{error}</p>}
        </div>
    );
}
```

---

## Revalidation Strategies

### revalidatePath()

**Revalidate specific pages after mutations:**

```typescript
'use server';

import { revalidatePath } from 'next/cache';

export async function createOrder(formData: FormData) {
    // ... create order

    // Revalidate list page
    revalidatePath('/orders');

    // Revalidate detail page
    revalidatePath(`/orders/${order.id}`);
}
```

### revalidateTag()

**Tag-based revalidation for related pages:**

```typescript
// Tag data fetches
const orders = await fetch('https://api.example.com/orders', {
    next: { tags: ['orders'] },
});

// Later, revalidate all 'orders' tagged fetches
import { revalidateTag } from 'next/cache';

export async function createOrder(formData: FormData) {
    // ... create order

    // Revalidate all 'orders' fetches
    revalidateTag('orders');
}
```

### router.refresh()

**Client-side refresh from hooks:**

```typescript
'use client';

import { useRouter } from 'next/navigation';

export function useRefresh() {
    const router = useRouter();

    function refresh() {
        router.refresh(); // Re-fetch Server Component data
    }

    return { refresh };
}
```

---

## Error Handling

### try/catch in Server Actions

```typescript
'use server';

import { prisma } from '@/lib/db';

export async function createOrder(formData: FormData) {
    try {
        const order = await prisma.order.create({
            data: {
                customerId: formData.get('customerId') as string,
                amount: parseFloat(formData.get('amount') as string),
            },
        });

        revalidatePath('/orders');
        return { success: true, order };
    } catch (error) {
        console.error('Order creation failed:', error);
        return { success: false, error: 'Failed to create order' };
    }
}
```

**Handle in Client Component:**

```typescript
'use client';

import { useTransition } from 'react';
import { createOrder } from '@/actions/orders';
import { useToast } from '@/hooks/useToast';

export function CreateOrderForm() {
    const [isPending, startTransition] = useTransition();
    const { toast } = useToast();

    async function handleSubmit(formData: FormData) {
        startTransition(async () => {
            const result = await createOrder(formData);

            if (result.success) {
                toast({ title: 'Order created successfully' });
            } else {
                toast({ title: 'Error', description: result.error, variant: 'destructive' });
            }
        });
    }

    return <form action={handleSubmit}>...</form>;
}
```

---

## Complete Data Flow Examples

### Example 1: Server-Side Fetch (Simple)

```typescript
// app/orders/page.tsx
import { prisma } from '@/lib/db';
import { OrdersList } from '@/features/orders/components/OrdersList';

export default async function OrdersPage() {
    const orders = await prisma.order.findMany({
        include: { customer: true },
        orderBy: { createdAt: 'desc' },
    });

    return (
        <div className="container">
            <h1>Orders</h1>
            <OrdersList orders={orders} />
        </div>
    );
}
```

### Example 2: Server Action Fetch (Reusable)

```typescript
// actions/orders.ts
'use server';

import { prisma } from '@/lib/db';

export async function getOrders(filters?: { status?: string; limit?: number }) {
    return await prisma.order.findMany({
        where: filters?.status ? { status: filters.status } : undefined,
        take: filters?.limit,
        include: { customer: true },
        orderBy: { createdAt: 'desc' },
    });
}

// app/orders/page.tsx
import { getOrders } from '@/actions/orders';
import { OrdersList } from '@/features/orders/components/OrdersList';

export default async function OrdersPage() {
    const orders = await getOrders({ status: 'active', limit: 50 });
    return <OrdersList orders={orders} />;
}
```

### Example 3: Parallel Fetching

```typescript
// app/dashboard/page.tsx
import { getStats, getOrders, getCustomers } from '@/actions/dashboard';

export default async function DashboardPage() {
    const [stats, orders, customers] = await Promise.all([
        getStats(),
        getOrders({ limit: 5 }),
        getCustomers({ limit: 5 }),
    ]);

    return (
        <div className="grid grid-cols-3 gap-4">
            <StatsCard data={stats} />
            <OrdersList orders={orders} />
            <CustomersList customers={customers} />
        </div>
    );
}
```

### Example 4: Client Search Hook

```typescript
// features/orders/hooks/useSearchOrders.ts
'use client';

import { useState } from 'react';
import { searchOrders } from '@/actions/orders';

export function useSearchOrders() {
    const [results, setResults] = useState([]);
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

// features/orders/components/OrdersSearch.tsx
'use client';

import { useSearchOrders } from '../hooks/useSearchOrders';

export function OrdersSearch() {
    const { results, isSearching, search } = useSearchOrders();

    return (
        <div>
            <input onChange={(e) => search(e.target.value)} />
            {isSearching ? 'Searching...' : `${results.length} results`}
        </div>
    );
}
```

### Example 5: Form with Server Action

```typescript
// actions/orders.ts
'use server';

import { prisma } from '@/lib/db';
import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';

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

// features/orders/components/CreateOrderForm.tsx
'use client';

import { useTransition } from 'react';
import { createOrder } from '@/actions/orders';

export function CreateOrderForm() {
    const [isPending, startTransition] = useTransition();

    return (
        <form action={(formData) => startTransition(async () => await createOrder(formData))}>
            <input name="customerId" required />
            <input name="amount" type="number" required />
            <button disabled={isPending}>
                {isPending ? 'Creating...' : 'Create'}
            </button>
        </form>
    );
}
```

---

## Summary

**Modern Data Fetching Recipe:**

1. **Server Components (Default)**: async fetch directly in page.tsx
2. **Server Actions**: Reusable fetch/mutation logic in actions/*.ts
3. **Parallel Fetching**: Promise.all() for concurrent requests
4. **Streaming**: Suspense boundaries for progressive rendering
5. **Client Hooks**: useSearchOrders, useUpdateOrder for dynamic client queries
6. **Revalidation**: revalidatePath(), revalidateTag(), router.refresh()
7. **Error Handling**: try/catch with user feedback
8. **No API Routes**: Direct database + Server Actions replace API layer

**See Also:**
- [component-patterns.md](component-patterns.md) - Server/Client component patterns
- [server-actions.md](server-actions.md) - Deep dive on Server Actions
- [server-components.md](server-components.md) - Advanced Server Component patterns
- [complete-examples.md](complete-examples.md) - Full working examples
