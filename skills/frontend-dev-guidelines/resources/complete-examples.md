# Complete Examples

Full working examples combining Next.js 15 App Router patterns: Server Components, Server Actions, Client Components, ShadCN UI, form handling, and data fetching.

---

## Example 1: Complete Server Component Page

Combines: async Server Component, data fetching, parallel requests, error handling

```typescript
// app/users/[userId]/page.tsx
import { Suspense } from 'react';
import { notFound } from 'next/navigation';
import { UserProfile } from '@/features/users/components/user-profile';
import { UserActivity } from '@/features/users/components/user-activity';
import { Skeleton } from '@/components/ui/skeleton';

interface PageProps {
    params: { userId: string };
}

async function getUser(userId: string) {
    const res = await fetch(`${process.env.API_URL}/users/${userId}`, {
        next: { revalidate: 300 }, // Cache for 5 minutes
    });

    if (!res.ok) {
        if (res.status === 404) notFound();
        throw new Error('Failed to fetch user');
    }

    return res.json();
}

async function getUserActivity(userId: string) {
    const res = await fetch(`${process.env.API_URL}/users/${userId}/activity`, {
        next: { revalidate: 60 }, // Cache for 1 minute
    });

    if (!res.ok) {
        throw new Error('Failed to fetch activity');
    }

    return res.json();
}

export default async function UserPage({ params }: PageProps) {
    // Fetch user data first (required)
    const user = await getUser(params.userId);

    return (
        <div className="container mx-auto p-6 space-y-6">
            <h1 className="text-3xl font-bold">User Profile</h1>

            {/* User profile - no suspense needed, already awaited */}
            <UserProfile user={user} />

            {/* Activity feed - stream separately */}
            <Suspense fallback={<ActivitySkeleton />}>
                <ActivityFeed userId={params.userId} />
            </Suspense>
        </div>
    );
}

async function ActivityFeed({ userId }: { userId: string }) {
    const activity = await getUserActivity(userId);
    return <UserActivity activity={activity} />;
}

function ActivitySkeleton() {
    return (
        <div className="space-y-2">
            <Skeleton className="h-12 w-full" />
            <Skeleton className="h-12 w-full" />
            <Skeleton className="h-12 w-full" />
        </div>
    );
}
```

---

## Example 2: Complete Feature Structure

Real example based on Orders feature:

```
features/
  orders/
    components/
      order-detail.tsx          # Server Component
      order-list.tsx            # Client Component (interactive)
      order-form-dialog.tsx     # Client Component (form)
      order-status-badge.tsx    # ShadCN Badge wrapper
    hooks/
      use-search-orders.ts      # Client-side search hook
    types/
      index.ts                  # TypeScript interfaces
    index.ts                    # Public API exports

actions/
  order-actions.ts              # Server Actions for mutations

app/
  orders/
    page.tsx                    # Route - fetches data
    [orderId]/
      page.tsx                  # Detail route
```

### Server Component (order-detail.tsx)

```typescript
// features/orders/components/order-detail.tsx
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { OrderStatusBadge } from './order-status-badge';
import type { Order } from '../types';

interface OrderDetailProps {
    order: Order;
}

export function OrderDetail({ order }: OrderDetailProps) {
    return (
        <Card>
            <CardHeader>
                <CardTitle className="flex items-center justify-between">
                    <span>Order #{order.orderNumber}</span>
                    <OrderStatusBadge status={order.status} />
                </CardTitle>
            </CardHeader>
            <CardContent className="space-y-4">
                <div>
                    <p className="text-sm text-muted-foreground">Customer</p>
                    <p className="font-medium">{order.customerName}</p>
                </div>
                <div>
                    <p className="text-sm text-muted-foreground">Email</p>
                    <p>{order.customerEmail}</p>
                </div>
                <div>
                    <p className="text-sm text-muted-foreground">Total</p>
                    <p className="text-lg font-semibold">
                        ${order.total.toFixed(2)}
                    </p>
                </div>
                <div>
                    <p className="text-sm text-muted-foreground">Created</p>
                    <p>{new Date(order.createdAt).toLocaleDateString()}</p>
                </div>
            </CardContent>
        </Card>
    );
}
```

### ShadCN Wrapper (order-status-badge.tsx)

```typescript
// features/orders/components/order-status-badge.tsx
import { Badge } from '@/components/ui/badge';
import type { OrderStatus } from '../types';

interface OrderStatusBadgeProps {
    status: OrderStatus;
}

const statusConfig = {
    pending: { label: 'Pending', variant: 'secondary' as const },
    processing: { label: 'Processing', variant: 'default' as const },
    completed: { label: 'Completed', variant: 'success' as const },
    cancelled: { label: 'Cancelled', variant: 'destructive' as const },
};

export function OrderStatusBadge({ status }: OrderStatusBadgeProps) {
    const config = statusConfig[status];
    return <Badge variant={config.variant}>{config.label}</Badge>;
}
```

### Client Component with Search Hook (order-list.tsx)

```typescript
// features/orders/components/order-list.tsx
'use client';

import { Input } from '@/components/ui/input';
import {
    Select,
    SelectContent,
    SelectItem,
    SelectTrigger,
    SelectValue,
} from '@/components/ui/select';
import { Card, CardContent } from '@/components/ui/card';
import { useSearchOrders } from '../hooks/use-search-orders';
import { OrderStatusBadge } from './order-status-badge';
import type { Order } from '../types';

interface OrderListProps {
    orders: Order[];
}

export function OrderList({ orders }: OrderListProps) {
    const {
        searchTerm,
        setSearchTerm,
        statusFilter,
        setStatusFilter,
        filteredOrders,
    } = useSearchOrders(orders);

    return (
        <div className="space-y-4">
            <div className="flex gap-4">
                <Input
                    placeholder="Search orders..."
                    value={searchTerm}
                    onChange={(e) => setSearchTerm(e.target.value)}
                    className="max-w-sm"
                />
                <Select value={statusFilter} onValueChange={setStatusFilter}>
                    <SelectTrigger className="w-[180px]">
                        <SelectValue placeholder="Filter by status" />
                    </SelectTrigger>
                    <SelectContent>
                        <SelectItem value="all">All</SelectItem>
                        <SelectItem value="pending">Pending</SelectItem>
                        <SelectItem value="processing">Processing</SelectItem>
                        <SelectItem value="completed">Completed</SelectItem>
                        <SelectItem value="cancelled">Cancelled</SelectItem>
                    </SelectContent>
                </Select>
            </div>

            <div className="space-y-2">
                {filteredOrders.map((order) => (
                    <Card key={order.id}>
                        <CardContent className="flex items-center justify-between p-4">
                            <div>
                                <p className="font-semibold">{order.customerName}</p>
                                <p className="text-sm text-muted-foreground">
                                    Order #{order.orderNumber}
                                </p>
                            </div>
                            <div className="flex items-center gap-4">
                                <p className="font-medium">
                                    ${order.total.toFixed(2)}
                                </p>
                                <OrderStatusBadge status={order.status} />
                            </div>
                        </CardContent>
                    </Card>
                ))}
            </div>
        </div>
    );
}
```

### Search Hook (use-search-orders.ts)

```typescript
// features/orders/hooks/use-search-orders.ts
'use client';

import { useState, useMemo } from 'react';
import { useDebounce } from '@/hooks/use-debounce';
import type { Order, OrderStatus } from '../types';

export function useSearchOrders(orders: Order[]) {
    const [searchTerm, setSearchTerm] = useState('');
    const [statusFilter, setStatusFilter] = useState<string>('all');

    const debouncedSearch = useDebounce(searchTerm, 300);

    const filteredOrders = useMemo(() => {
        let filtered = orders;

        // Apply search filter
        if (debouncedSearch) {
            const search = debouncedSearch.toLowerCase();
            filtered = filtered.filter(
                (order) =>
                    order.customerName.toLowerCase().includes(search) ||
                    order.orderNumber.toLowerCase().includes(search) ||
                    order.customerEmail.toLowerCase().includes(search)
            );
        }

        // Apply status filter
        if (statusFilter !== 'all') {
            filtered = filtered.filter((order) => order.status === statusFilter);
        }

        return filtered;
    }, [orders, debouncedSearch, statusFilter]);

    return {
        searchTerm,
        setSearchTerm,
        statusFilter,
        setStatusFilter,
        filteredOrders,
    };
}
```

### Types (types/index.ts)

```typescript
// features/orders/types/index.ts
export type OrderStatus = 'pending' | 'processing' | 'completed' | 'cancelled';

export interface Order {
    id: string;
    orderNumber: string;
    customerName: string;
    customerEmail: string;
    status: OrderStatus;
    total: number;
    createdAt: string;
    updatedAt: string;
}

export interface CreateOrderInput {
    customerName: string;
    customerEmail: string;
    items: OrderItem[];
}

export interface OrderItem {
    productId: string;
    quantity: number;
    price: number;
}
```

### Public Exports (index.ts)

```typescript
// features/orders/index.ts
export { OrderDetail } from './components/order-detail';
export { OrderList } from './components/order-list';
export { OrderStatusBadge } from './components/order-status-badge';

export { useSearchOrders } from './hooks/use-search-orders';

export type { Order, OrderStatus, CreateOrderInput } from './types';
```

---

## Example 3: Complete Route with Data Fetching

```typescript
// app/orders/page.tsx
import { Suspense } from 'react';
import { OrderList } from '@/features/orders/components/order-list';
import { CreateOrderDialog } from '@/features/orders/components/create-order-dialog';
import { Skeleton } from '@/components/ui/skeleton';

interface PageProps {
    searchParams: { status?: string };
}

async function getOrders(status?: string) {
    const url = new URL(`${process.env.API_URL}/orders`);
    if (status && status !== 'all') {
        url.searchParams.set('status', status);
    }

    const res = await fetch(url.toString(), {
        next: { revalidate: 60 },
    });

    if (!res.ok) {
        throw new Error('Failed to fetch orders');
    }

    return res.json();
}

export default async function OrdersPage({ searchParams }: PageProps) {
    return (
        <div className="container mx-auto p-6 space-y-6">
            <div className="flex items-center justify-between">
                <h1 className="text-3xl font-bold">Orders</h1>
                <CreateOrderDialog />
            </div>

            <Suspense fallback={<OrdersListSkeleton />}>
                <OrdersListAsync status={searchParams.status} />
            </Suspense>
        </div>
    );
}

async function OrdersListAsync({ status }: { status?: string }) {
    const orders = await getOrders(status);
    return <OrderList orders={orders} />;
}

function OrdersListSkeleton() {
    return (
        <div className="space-y-2">
            <Skeleton className="h-20 w-full" />
            <Skeleton className="h-20 w-full" />
            <Skeleton className="h-20 w-full" />
        </div>
    );
}
```

---

## Example 4: Complete Form with Server Action

```typescript
// actions/order-actions.ts
'use server';

import { z } from 'zod';
import { revalidatePath } from 'next/cache';
import { auth } from '@/lib/auth';

const createOrderSchema = z.object({
    customerName: z.string().min(1, 'Customer name is required'),
    customerEmail: z.string().email('Invalid email address'),
    total: z.number().min(0.01, 'Total must be greater than 0'),
});

export async function createOrder(formData: FormData) {
    // Check authentication
    const session = await auth();
    if (!session) {
        return {
            success: false,
            errors: { _form: ['You must be logged in to create orders'] },
        };
    }

    // Validate input
    const parsed = createOrderSchema.safeParse({
        customerName: formData.get('customerName'),
        customerEmail: formData.get('customerEmail'),
        total: Number(formData.get('total')),
    });

    if (!parsed.success) {
        return {
            success: false,
            errors: parsed.error.flatten().fieldErrors,
        };
    }

    try {
        const response = await fetch(`${process.env.API_URL}/orders`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify(parsed.data),
        });

        if (!response.ok) {
            throw new Error('Failed to create order');
        }

        const order = await response.json();

        revalidatePath('/orders');
        revalidatePath('/');

        return { success: true, data: order };
    } catch (error) {
        return {
            success: false,
            errors: { _form: ['Failed to create order. Please try again.'] },
        };
    }
}
```

```typescript
// features/orders/components/create-order-dialog.tsx
'use client';

import { useState, useTransition } from 'react';
import {
    Dialog,
    DialogContent,
    DialogHeader,
    DialogTitle,
    DialogTrigger,
} from '@/components/ui/dialog';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { Plus } from 'lucide-react';
import { toast } from 'sonner';
import { createOrder } from '@/actions/order-actions';

export function CreateOrderDialog() {
    const [open, setOpen] = useState(false);
    const [isPending, startTransition] = useTransition();
    const [errors, setErrors] = useState<Record<string, string[]>>({});

    async function handleSubmit(formData: FormData) {
        startTransition(async () => {
            const result = await createOrder(formData);

            if (result.success) {
                toast.success('Order created successfully');
                setOpen(false);
                setErrors({});
            } else if (result.errors) {
                setErrors(result.errors);
                toast.error('Failed to create order');
            }
        });
    }

    return (
        <Dialog open={open} onOpenChange={setOpen}>
            <DialogTrigger asChild>
                <Button>
                    <Plus className="mr-2 h-4 w-4" />
                    Create Order
                </Button>
            </DialogTrigger>
            <DialogContent>
                <DialogHeader>
                    <DialogTitle>Create New Order</DialogTitle>
                </DialogHeader>
                <form action={handleSubmit} className="space-y-4">
                    <div>
                        <Label htmlFor="customerName">Customer Name</Label>
                        <Input
                            id="customerName"
                            name="customerName"
                            placeholder="John Doe"
                        />
                        {errors.customerName && (
                            <p className="text-sm text-red-500 mt-1">
                                {errors.customerName[0]}
                            </p>
                        )}
                    </div>

                    <div>
                        <Label htmlFor="customerEmail">Email</Label>
                        <Input
                            id="customerEmail"
                            name="customerEmail"
                            type="email"
                            placeholder="john@example.com"
                        />
                        {errors.customerEmail && (
                            <p className="text-sm text-red-500 mt-1">
                                {errors.customerEmail[0]}
                            </p>
                        )}
                    </div>

                    <div>
                        <Label htmlFor="total">Total Amount</Label>
                        <Input
                            id="total"
                            name="total"
                            type="number"
                            step="0.01"
                            placeholder="99.99"
                        />
                        {errors.total && (
                            <p className="text-sm text-red-500 mt-1">
                                {errors.total[0]}
                            </p>
                        )}
                    </div>

                    {errors._form && (
                        <p className="text-sm text-red-500">{errors._form[0]}</p>
                    )}

                    <div className="flex justify-end gap-2">
                        <Button
                            type="button"
                            variant="outline"
                            onClick={() => setOpen(false)}
                        >
                            Cancel
                        </Button>
                        <Button type="submit" disabled={isPending}>
                            {isPending ? 'Creating...' : 'Create Order'}
                        </Button>
                    </div>
                </form>
            </DialogContent>
        </Dialog>
    );
}
```

---

## Example 5: Parallel Data Fetching

```typescript
// app/dashboard/page.tsx
import { Suspense } from 'react';
import { StatsCards } from '@/features/dashboard/components/stats-cards';
import { RecentOrders } from '@/features/dashboard/components/recent-orders';
import { ActivityFeed } from '@/features/dashboard/components/activity-feed';
import { Skeleton } from '@/components/ui/skeleton';

async function getStats() {
    const res = await fetch(`${process.env.API_URL}/stats`, {
        next: { revalidate: 300 },
    });
    if (!res.ok) throw new Error('Failed to fetch stats');
    return res.json();
}

async function getRecentOrders() {
    const res = await fetch(`${process.env.API_URL}/orders?limit=5`, {
        next: { revalidate: 60 },
    });
    if (!res.ok) throw new Error('Failed to fetch orders');
    return res.json();
}

async function getActivity() {
    const res = await fetch(`${process.env.API_URL}/activity`, {
        next: { revalidate: 30 },
    });
    if (!res.ok) throw new Error('Failed to fetch activity');
    return res.json();
}

export default async function DashboardPage() {
    // Fetch stats first (critical data)
    const stats = await getStats();

    return (
        <div className="container mx-auto p-6 space-y-6">
            <h1 className="text-3xl font-bold">Dashboard</h1>

            {/* Stats - already awaited */}
            <StatsCards stats={stats} />

            {/* Orders and Activity - stream in parallel */}
            <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
                <Suspense fallback={<CardSkeleton />}>
                    <RecentOrdersAsync />
                </Suspense>

                <Suspense fallback={<CardSkeleton />}>
                    <ActivityFeedAsync />
                </Suspense>
            </div>
        </div>
    );
}

async function RecentOrdersAsync() {
    const orders = await getRecentOrders();
    return <RecentOrders orders={orders} />;
}

async function ActivityFeedAsync() {
    const activity = await getActivity();
    return <ActivityFeed activity={activity} />;
}

function CardSkeleton() {
    return (
        <div className="border rounded-lg p-6">
            <Skeleton className="h-8 w-32 mb-4" />
            <div className="space-y-2">
                <Skeleton className="h-12 w-full" />
                <Skeleton className="h-12 w-full" />
                <Skeleton className="h-12 w-full" />
            </div>
        </div>
    );
}
```

---

## Example 6: Update with Optimistic UI

```typescript
// features/orders/components/update-order-status.tsx
'use client';

import { useTransition } from 'react';
import { useRouter } from 'next/navigation';
import {
    Select,
    SelectContent,
    SelectItem,
    SelectTrigger,
    SelectValue,
} from '@/components/ui/select';
import { toast } from 'sonner';
import { updateOrderStatus } from '@/actions/order-actions';
import type { OrderStatus } from '../types';

interface UpdateOrderStatusProps {
    orderId: string;
    currentStatus: OrderStatus;
}

export function UpdateOrderStatus({ orderId, currentStatus }: UpdateOrderStatusProps) {
    const router = useRouter();
    const [isPending, startTransition] = useTransition();

    async function handleStatusChange(newStatus: OrderStatus) {
        startTransition(async () => {
            const formData = new FormData();
            formData.set('status', newStatus);

            const result = await updateOrderStatus(orderId, formData);

            if (result.success) {
                toast.success('Order status updated');
                router.refresh(); // Refresh Server Component data
            } else {
                toast.error(result.errors?._form?.[0] || 'Failed to update status');
            }
        });
    }

    return (
        <Select
            value={currentStatus}
            onValueChange={handleStatusChange}
            disabled={isPending}
        >
            <SelectTrigger className="w-[180px]">
                <SelectValue />
            </SelectTrigger>
            <SelectContent>
                <SelectItem value="pending">Pending</SelectItem>
                <SelectItem value="processing">Processing</SelectItem>
                <SelectItem value="completed">Completed</SelectItem>
                <SelectItem value="cancelled">Cancelled</SelectItem>
            </SelectContent>
        </Select>
    );
}
```

---

## Summary

**Key Takeaways:**

1. **Server Components**: async/await for data fetching in page.tsx
2. **Client Components**: 'use client' for interactivity (forms, search)
3. **Server Actions**: actions/*.ts for all mutations with Zod validation
4. **Streaming**: Suspense boundaries for progressive loading
5. **ShadCN Wrappers**: Domain-specific badge/button components
6. **Search/Filter**: Client-side hooks with useMemo + debouncing
7. **Forms**: FormData + Server Actions + useTransition
8. **Error Handling**: try/catch in Server Actions, return error objects
9. **Revalidation**: revalidatePath/revalidateTag + router.refresh()
10. **Toast Notifications**: Sonner for user feedback

**Architecture Flow:**
```
page.tsx (Server Component)
  ├─ Fetch data with async/await
  ├─ Pass data to Client Components as props
  └─ Client Components
      ├─ Handle user interaction
      ├─ Call Server Actions for mutations
      └─ Use hooks for local state (search, filters)
```

**See other resources for detailed explanations of each pattern.**
