# Loading & Error States

Modern loading and error handling patterns for Next.js 15 App Router. Learn loading.tsx convention, Suspense streaming, error boundaries, and skeleton states.

---

## Next.js 15 Loading Patterns

### loading.tsx File Convention

**Automatic loading UI** - Next.js wraps your page in Suspense automatically.

```typescript
// app/orders/loading.tsx
import { Skeleton } from '@/components/ui/skeleton';

export default function OrdersLoading() {
    return (
        <div className="container py-8 space-y-4">
            <Skeleton className="h-8 w-48" /> {/* Page title */}
            <Skeleton className="h-12 w-full" /> {/* Search bar */}
            <Skeleton className="h-96 w-full" /> {/* Table */}
        </div>
    );
}

// app/orders/page.tsx
export default async function OrdersPage() {
    const orders = await getOrders(); // Suspends while fetching
    return <OrdersList orders={orders} />;
}
```

**What happens:**
1. User navigates to `/orders`
2. `loading.tsx` renders immediately
3. `page.tsx` async function runs
4. Once resolved, `page.tsx` replaces `loading.tsx`
5. **No layout shift** - loading UI reserves space

---

## Suspense Boundaries

### Manual Suspense for Streaming

**Stream independent sections** as they become ready:

```typescript
// app/dashboard/page.tsx
import { Suspense } from 'react';
import { Skeleton } from '@/components/ui/skeleton';

export default function DashboardPage() {
    return (
        <div className="container py-8">
            <h1 className="text-3xl font-bold mb-6">Dashboard</h1>

            <div className="grid grid-cols-3 gap-4">
                {/* Stats load fast - show first */}
                <Suspense fallback={<Skeleton className="h-32" />}>
                    <StatsSection />
                </Suspense>

                {/* Orders load medium speed */}
                <Suspense fallback={<Skeleton className="h-64" />}>
                    <RecentOrdersSection />
                </Suspense>

                {/* Analytics slow - but doesn't block other sections */}
                <Suspense fallback={<Skeleton className="h-64" />}>
                    <AnalyticsSection />
                </Suspense>
            </div>
        </div>
    );
}

// Each section is async Server Component
async function StatsSection() {
    const stats = await getStats(); // Fast query
    return <StatsCard data={stats} />;
}

async function RecentOrdersSection() {
    const orders = await getRecentOrders(); // Medium query
    return <OrdersList orders={orders} />;
}

async function AnalyticsSection() {
    const analytics = await getAnalytics(); // Slow query
    return <AnalyticsChart data={analytics} />;
}
```

**Benefits:**
- **Progressive rendering** - fast sections appear first
- **Better perceived performance** - user sees content sooner
- **No blocking** - slow query doesn't block page

---

## Skeleton Components (ShadCN)

### Basic Skeleton Usage

```typescript
import { Skeleton } from '@/components/ui/skeleton';

export function OrderCardSkeleton() {
    return (
        <div className="rounded-lg border bg-card p-6 space-y-3">
            <Skeleton className="h-4 w-24" /> {/* Order number */}
            <Skeleton className="h-8 w-full" /> {/* Customer name */}
            <div className="flex gap-2">
                <Skeleton className="h-4 w-20" /> {/* Status badge */}
                <Skeleton className="h-4 w-20" /> {/* Date */}
            </div>
            <Skeleton className="h-10 w-full" /> {/* Amount */}
        </div>
    );
}
```

### Reusable Skeleton Patterns

```typescript
// components/skeletons/table-skeleton.tsx
export function TableSkeleton({ rows = 5 }: { rows?: number }) {
    return (
        <div className="space-y-2">
            {/* Header */}
            <div className="flex gap-4 border-b pb-2">
                <Skeleton className="h-4 w-32" />
                <Skeleton className="h-4 w-48" />
                <Skeleton className="h-4 w-24" />
            </div>

            {/* Rows */}
            {Array.from({ length: rows }).map((_, i) => (
                <div key={i} className="flex gap-4 py-2">
                    <Skeleton className="h-4 w-32" />
                    <Skeleton className="h-4 w-48" />
                    <Skeleton className="h-4 w-24" />
                </div>
            ))}
        </div>
    );
}

// Usage
<Suspense fallback={<TableSkeleton rows={10} />}>
    <OrdersTable />
</Suspense>
```

### Card Grid Skeleton

```typescript
export function CardGridSkeleton({ count = 6 }: { count?: number }) {
    return (
        <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
            {Array.from({ length: count }).map((_, i) => (
                <div key={i} className="rounded-lg border bg-card p-6 space-y-3">
                    <Skeleton className="h-6 w-3/4" />
                    <Skeleton className="h-4 w-full" />
                    <Skeleton className="h-4 w-full" />
                    <Skeleton className="h-10 w-24" />
                </div>
            ))}
        </div>
    );
}
```

---

## Error Handling

### error.tsx File Convention

**Automatic error boundaries** for every route:

```typescript
// app/orders/error.tsx
'use client'; // Error components MUST be Client Components

import { useEffect } from 'react';
import { Button } from '@/components/ui/button';
import { AlertCircle } from 'lucide-react';

export default function OrdersError({
    error,
    reset,
}: {
    error: Error & { digest?: string };
    reset: () => void;
}) {
    useEffect(() => {
        // Log error to error reporting service
        console.error('Orders page error:', error);
    }, [error]);

    return (
        <div className="flex flex-col items-center justify-center min-h-screen p-4">
            <AlertCircle className="h-12 w-12 text-red-500 mb-4" />
            <h2 className="text-2xl font-bold mb-2">Something went wrong!</h2>
            <p className="text-muted-foreground mb-4">
                {error.message || 'An unexpected error occurred'}
            </p>
            <Button onClick={reset}>Try again</Button>
        </div>
    );
}
```

**What triggers error.tsx:**
- Errors in `page.tsx` Server Component
- Errors in nested Server Components
- Errors during data fetching

### global-error.tsx (Root Error Boundary)

```typescript
// app/global-error.tsx
'use client';

export default function GlobalError({
    error,
    reset,
}: {
    error: Error & { digest?: string };
    reset: () => void;
}) {
    return (
        <html>
            <body>
                <div className="flex flex-col items-center justify-center min-h-screen">
                    <h2 className="text-2xl font-bold">Application Error</h2>
                    <p className="text-muted-foreground">{error.message}</p>
                    <button onClick={reset}>Try again</button>
                </div>
            </body>
        </html>
    );
}
```

**Use case:** Catches errors in root layout.tsx

---

## not-found.tsx (404 Handling)

### Route-Specific 404

```typescript
// app/orders/[id]/not-found.tsx
import { Button } from '@/components/ui/button';
import Link from 'next/link';
import { FileQuestion } from 'lucide-react';

export default function OrderNotFound() {
    return (
        <div className="flex flex-col items-center justify-center min-h-screen">
            <FileQuestion className="h-16 w-16 text-muted-foreground mb-4" />
            <h2 className="text-2xl font-bold mb-2">Order Not Found</h2>
            <p className="text-muted-foreground mb-4">
                The order you're looking for doesn't exist or has been removed.
            </p>
            <Button asChild>
                <Link href="/orders">Back to Orders</Link>
            </Button>
        </div>
    );
}
```

### Trigger notFound()

```typescript
// app/orders/[id]/page.tsx
import { notFound } from 'next/navigation';

export default async function OrderPage({ params }: { params: { id: string } }) {
    const order = await prisma.order.findUnique({
        where: { id: params.id },
    });

    if (!order) {
        notFound(); // Triggers not-found.tsx
    }

    return <OrderDetails order={order} />;
}
```

---

## Toast Notifications (Sonner)

### Setup Sonner

```typescript
// app/layout.tsx
import { Toaster } from 'sonner';

export default function RootLayout({ children }: { children: React.ReactNode }) {
    return (
        <html>
            <body>
                {children}
                <Toaster />
            </body>
        </html>
    );
}
```

### Using toast in Client Components

```typescript
'use client';

import { useState, useTransition } from 'react';
import { toast } from 'sonner';
import { Button } from '@/components/ui/button';
import { deleteOrder } from '@/actions/order-actions';

export function DeleteOrderButton({ orderId }: { orderId: string }) {
    const [isPending, startTransition] = useTransition();

    function handleDelete() {
        startTransition(async () => {
            const result = await deleteOrder(orderId);

            if (result.success) {
                toast.success('Order deleted successfully');
            } else {
                toast.error(result.errors?._form?.[0] || 'Failed to delete order');
            }
        });
    }

    return (
        <Button
            onClick={handleDelete}
            disabled={isPending}
            variant="destructive"
        >
            {isPending ? 'Deleting...' : 'Delete'}
        </Button>
    );
}
```

### Toast Variants

```typescript
import { toast } from 'sonner';

// Success
toast.success('Order created successfully');

// Error
toast.error('Failed to create order');

// Info
toast.info('Order is being processed');

// Warning
toast.warning('Low stock remaining');

// Loading (with promise)
toast.promise(
    createOrder(formData),
    {
        loading: 'Creating order...',
        success: 'Order created!',
        error: 'Failed to create order',
    }
);

// Custom action
toast('Order updated', {
    action: {
        label: 'Undo',
        onClick: () => undoUpdate(),
    },
});
```

---

## Client-Side Loading States

### useTransition for Mutations

```typescript
'use client';

import { useTransition } from 'react';
import { Button } from '@/components/ui/button';
import { updateOrderStatus } from '@/actions/order-actions';

export function OrderActions({ orderId }: { orderId: string }) {
    const [isPending, startTransition] = useTransition();

    function handleComplete() {
        startTransition(async () => {
            await updateOrderStatus(orderId, 'completed');
        });
    }

    return (
        <Button
            onClick={handleComplete}
            disabled={isPending}
        >
            {isPending ? 'Updating...' : 'Mark Complete'}
        </Button>
    );
}
```

### Loading Spinner Component

```typescript
import { Loader2 } from 'lucide-react';

export function LoadingSpinner({ size = 'default' }: { size?: 'sm' | 'default' | 'lg' }) {
    const sizeClasses = {
        sm: 'h-4 w-4',
        default: 'h-8 w-8',
        lg: 'h-12 w-12',
    };

    return (
        <Loader2 className={`animate-spin text-muted-foreground ${sizeClasses[size]}`} />
    );
}

// Usage
<LoadingSpinner size="sm" />
```

---

## Complete Examples

### Example 1: Page with loading.tsx

```typescript
// app/orders/loading.tsx
import { Skeleton } from '@/components/ui/skeleton';

export default function OrdersLoading() {
    return (
        <div className="container py-8 space-y-4">
            <Skeleton className="h-8 w-48" />
            <div className="flex gap-2">
                <Skeleton className="h-10 w-64" />
                <Skeleton className="h-10 w-32" />
            </div>
            <Skeleton className="h-96 w-full" />
        </div>
    );
}

// app/orders/page.tsx
export default async function OrdersPage() {
    const orders = await prisma.order.findMany();

    return (
        <div className="container py-8">
            <h1 className="text-3xl font-bold mb-6">Orders</h1>
            <OrdersFilter />
            <OrdersTable orders={orders} />
        </div>
    );
}
```

### Example 2: Streaming with Suspense

```typescript
// app/dashboard/page.tsx
import { Suspense } from 'react';
import { CardGridSkeleton } from '@/components/skeletons/card-grid-skeleton';

export default function DashboardPage() {
    return (
        <div className="container py-8 space-y-8">
            <h1 className="text-3xl font-bold">Dashboard</h1>

            {/* Stats load first */}
            <Suspense fallback={<CardGridSkeleton count={3} />}>
                <StatsCards />
            </Suspense>

            {/* Recent orders */}
            <Suspense fallback={<TableSkeleton />}>
                <RecentOrders />
            </Suspense>
        </div>
    );
}

async function StatsCards() {
    const stats = await getStats();
    return <StatsDisplay stats={stats} />;
}

async function RecentOrders() {
    const orders = await getRecentOrders();
    return <OrdersTable orders={orders} />;
}
```

### Example 3: Error Handling with Toast

```typescript
// features/orders/components/create-order-form.tsx
'use client';

import { useState, useTransition } from 'react';
import { toast } from 'sonner';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { createOrder } from '@/actions/order-actions';

export function CreateOrderForm() {
    const [isPending, startTransition] = useTransition();
    const [errors, setErrors] = useState<Record<string, string[]>>({});

    async function handleSubmit(formData: FormData) {
        startTransition(async () => {
            const result = await createOrder(formData);

            if (result.success) {
                toast.success('Order created successfully');
                setErrors({});
            } else if (result.errors) {
                setErrors(result.errors);
                toast.error('Failed to create order');
            }
        });
    }

    return (
        <form action={handleSubmit} className="space-y-4">
            <Input name="customerId" placeholder="Customer ID" />
            {errors.customerId && (
                <p className="text-sm text-red-500">{errors.customerId[0]}</p>
            )}

            <Input name="amount" type="number" placeholder="Amount" />
            {errors.amount && (
                <p className="text-sm text-red-500">{errors.amount[0]}</p>
            )}

            <Button type="submit" disabled={isPending}>
                {isPending ? 'Creating...' : 'Create Order'}
            </Button>
        </form>
    );
}
```

---

## Best Practices

### DO:

**✅ Use loading.tsx for automatic loading UI**
```typescript
// app/orders/loading.tsx
export default function Loading() {
    return <OrdersSkeleton />;
}
```

**✅ Use Suspense for progressive rendering**
```typescript
<Suspense fallback={<Skeleton />}>
    <SlowComponent />
</Suspense>
```

**✅ Match skeleton layout to actual content**
```typescript
// Same structure - no layout shift
{isLoading ? (
    <Skeleton className="h-64 w-full" />
) : (
    <div className="h-64 w-full">Content</div>
)}
```

**✅ Use toast for user feedback**
```typescript
toast.success('Action completed');
toast.error('Action failed');
```

### DON'T:

**❌ Don't use early returns with loading states**
```typescript
// ❌ BAD - causes layout shift
if (isLoading) {
    return <Spinner />;
}
return <Content />;
```

**❌ Don't change layout between loading and loaded**
```typescript
// ❌ BAD - different heights
{isLoading ? (
    <div className="h-32">Loading</div>
) : (
    <div className="h-64">Content</div>
)}
```

**❌ Don't block entire page for slow sections**
```typescript
// ❌ BAD - everything waits for slow query
const [stats, analytics, reports] = await Promise.all([
    getStats(),      // Fast
    getAnalytics(),  // SLOW - blocks everything!
    getReports(),    // Fast
]);

// ✅ GOOD - slow section streams separately
<Suspense fallback={<Skeleton />}>
    <AnalyticsSection /> {/* Loads independently */}
</Suspense>
```

---

## Summary

**Loading Patterns:**
1. **loading.tsx** - Automatic loading UI for entire page
2. **Suspense boundaries** - Stream independent sections
3. **Skeleton components** - Match actual content layout
4. **useTransition** - Client-side mutation loading states

**Error Handling:**
1. **error.tsx** - Automatic error boundary for routes
2. **not-found.tsx** - Custom 404 pages
3. **global-error.tsx** - Root-level errors
4. **Toast (Sonner)** - User feedback for actions

**Key Principles:**
- ✅ No layout shift (reserve space with skeletons)
- ✅ Progressive rendering (show fast content first)
- ✅ Independent sections (Suspense boundaries)
- ✅ User feedback (toast notifications)
- ❌ Avoid early returns
- ❌ Avoid blocking entire page

**See Also:**
- [server-components.md](server-components.md) - Suspense and streaming
- [data-fetching.md](data-fetching.md) - Data loading patterns
- [component-patterns.md](component-patterns.md) - Server/Client components
- [complete-examples.md](complete-examples.md) - Full examples
