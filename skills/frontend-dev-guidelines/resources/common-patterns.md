# Common Patterns

Frequently used patterns for authentication, forms, dialogs, search/filtering, and other common UI elements in Next.js 15 App Router applications.

---

## Authentication Patterns

### Getting Current User in Server Components

```typescript
// app/dashboard/page.tsx
import { auth } from '@/lib/auth';
import { redirect } from 'next/navigation';

export default async function DashboardPage() {
    const session = await auth();

    if (!session?.user) {
        redirect('/login');
    }

    // Available properties:
    // - session.user.id: string
    // - session.user.email: string
    // - session.user.name: string
    // - session.user.role: string

    return (
        <div>
            <h1>Dashboard</h1>
            <p>Logged in as: {session.user.email}</p>
            <p>Name: {session.user.name}</p>
            <p>Role: {session.user.role}</p>
        </div>
    );
}
```

### Getting Current User in Client Components

```typescript
// features/profile/components/user-menu.tsx
'use client';

import { useSession } from 'next-auth/react';

export function UserMenu() {
    const { data: session } = useSession();

    if (!session?.user) {
        return null;
    }

    return (
        <div>
            <p>Welcome, {session.user.name}</p>
            <p>{session.user.email}</p>
        </div>
    );
}
```

**Rules:**
- ✅ Use `auth()` in Server Components (page.tsx, layout.tsx)
- ✅ Use `useSession()` in Client Components (interactive UI)
- ❌ NEVER make direct API calls for auth - use auth library

---

## Forms with Server Actions

### Basic Form with Server Action

```typescript
// actions/user-actions.ts
'use server';

import { z } from 'zod';
import { revalidatePath } from 'next/cache';

const userSchema = z.object({
    username: z.string().min(3, 'Username must be at least 3 characters'),
    email: z.string().email('Invalid email address'),
    age: z.number().min(18, 'Must be 18 or older'),
});

export async function createUser(formData: FormData) {
    // Extract and validate data
    const parsed = userSchema.safeParse({
        username: formData.get('username'),
        email: formData.get('email'),
        age: Number(formData.get('age')),
    });

    if (!parsed.success) {
        return {
            success: false,
            errors: parsed.error.flatten().fieldErrors,
        };
    }

    try {
        // Call your backend API
        const response = await fetch(`${process.env.API_URL}/users`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(parsed.data),
        });

        if (!response.ok) {
            throw new Error('Failed to create user');
        }

        revalidatePath('/users');
        return { success: true };
    } catch (error) {
        return {
            success: false,
            errors: { _form: ['Failed to create user'] },
        };
    }
}
```

```typescript
// features/users/components/create-user-form.tsx
'use client';

import { useState, useTransition } from 'react';
import { createUser } from '@/actions/user-actions';
import { Input } from '@/components/ui/input';
import { Button } from '@/components/ui/button';
import { Label } from '@/components/ui/label';

export function CreateUserForm() {
    const [isPending, startTransition] = useTransition();
    const [errors, setErrors] = useState<Record<string, string[]>>({});

    async function handleSubmit(formData: FormData) {
        startTransition(async () => {
            const result = await createUser(formData);

            if (!result.success && result.errors) {
                setErrors(result.errors);
            } else {
                setErrors({});
                // Reset form or show success message
            }
        });
    }

    return (
        <form action={handleSubmit} className="space-y-4">
            <div>
                <Label htmlFor="username">Username</Label>
                <Input
                    id="username"
                    name="username"
                    type="text"
                />
                {errors.username && (
                    <p className="text-sm text-red-500">{errors.username[0]}</p>
                )}
            </div>

            <div>
                <Label htmlFor="email">Email</Label>
                <Input
                    id="email"
                    name="email"
                    type="email"
                />
                {errors.email && (
                    <p className="text-sm text-red-500">{errors.email[0]}</p>
                )}
            </div>

            <div>
                <Label htmlFor="age">Age</Label>
                <Input
                    id="age"
                    name="age"
                    type="number"
                />
                {errors.age && (
                    <p className="text-sm text-red-500">{errors.age[0]}</p>
                )}
            </div>

            {errors._form && (
                <p className="text-sm text-red-500">{errors._form[0]}</p>
            )}

            <Button type="submit" disabled={isPending}>
                {isPending ? 'Creating...' : 'Create User'}
            </Button>
        </form>
    );
}
```

---

## Dialog Component Pattern

### ShadCN Dialog with Form

```typescript
// features/users/components/add-user-dialog.tsx
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
import { UserPlus } from 'lucide-react';
import { createUser } from '@/actions/user-actions';

export function AddUserDialog() {
    const [open, setOpen] = useState(false);
    const [isPending, startTransition] = useTransition();
    const [errors, setErrors] = useState<Record<string, string[]>>({});

    async function handleSubmit(formData: FormData) {
        startTransition(async () => {
            const result = await createUser(formData);

            if (result.success) {
                setOpen(false);
                setErrors({});
            } else if (result.errors) {
                setErrors(result.errors);
            }
        });
    }

    return (
        <Dialog open={open} onOpenChange={setOpen}>
            <DialogTrigger asChild>
                <Button>
                    <UserPlus className="mr-2 h-4 w-4" />
                    Add User
                </Button>
            </DialogTrigger>
            <DialogContent>
                <DialogHeader>
                    <DialogTitle>Add New User</DialogTitle>
                </DialogHeader>
                <form action={handleSubmit} className="space-y-4">
                    <div>
                        <Label htmlFor="name">Name</Label>
                        <Input
                            id="name"
                            name="name"
                            placeholder="John Doe"
                        />
                        {errors.name && (
                            <p className="text-sm text-red-500">{errors.name[0]}</p>
                        )}
                    </div>

                    <div>
                        <Label htmlFor="email">Email</Label>
                        <Input
                            id="email"
                            name="email"
                            type="email"
                            placeholder="john@example.com"
                        />
                        {errors.email && (
                            <p className="text-sm text-red-500">{errors.email[0]}</p>
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
                            {isPending ? 'Adding...' : 'Add User'}
                        </Button>
                    </div>
                </form>
            </DialogContent>
        </Dialog>
    );
}
```

---

## Search and Filtering Patterns

### Client-Side Search Hook

```typescript
// features/orders/hooks/use-search-orders.ts
'use client';

import { useState, useMemo } from 'react';
import { useDebounce } from '@/hooks/use-debounce';
import type { Order } from '@/types/order';

export function useSearchOrders(orders: Order[]) {
    const [searchTerm, setSearchTerm] = useState('');
    const [statusFilter, setStatusFilter] = useState<string>('all');

    const debouncedSearch = useDebounce(searchTerm, 300);

    const filteredOrders = useMemo(() => {
        let filtered = orders;

        // Apply search filter
        if (debouncedSearch) {
            filtered = filtered.filter(order =>
                order.customerName.toLowerCase().includes(debouncedSearch.toLowerCase()) ||
                order.orderNumber.toLowerCase().includes(debouncedSearch.toLowerCase())
            );
        }

        // Apply status filter
        if (statusFilter !== 'all') {
            filtered = filtered.filter(order => order.status === statusFilter);
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

### Search Component Using Hook

```typescript
// features/orders/components/orders-list.tsx
'use client';

import { Input } from '@/components/ui/input';
import {
    Select,
    SelectContent,
    SelectItem,
    SelectTrigger,
    SelectValue,
} from '@/components/ui/select';
import { useSearchOrders } from '../hooks/use-search-orders';
import type { Order } from '@/types/order';

interface OrdersListProps {
    orders: Order[];
}

export function OrdersList({ orders }: OrdersListProps) {
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
                        <SelectItem value="completed">Completed</SelectItem>
                        <SelectItem value="cancelled">Cancelled</SelectItem>
                    </SelectContent>
                </Select>
            </div>

            <div className="space-y-2">
                {filteredOrders.map((order) => (
                    <div key={order.id} className="border p-4 rounded-lg">
                        <p className="font-semibold">{order.customerName}</p>
                        <p className="text-sm text-muted-foreground">
                            Order #{order.orderNumber}
                        </p>
                    </div>
                ))}
            </div>
        </div>
    );
}
```

---

## Data Mutation Patterns

### Update with Revalidation

```typescript
// actions/order-actions.ts
'use server';

import { z } from 'zod';
import { revalidatePath, revalidateTag } from 'next/cache';
import { auth } from '@/lib/auth';

const updateOrderSchema = z.object({
    status: z.enum(['pending', 'processing', 'completed', 'cancelled']),
    notes: z.string().optional(),
});

export async function updateOrder(orderId: string, formData: FormData) {
    // Check authentication
    const session = await auth();
    if (!session) {
        return { success: false, errors: { _form: ['Not authenticated'] } };
    }

    // Validate input
    const parsed = updateOrderSchema.safeParse({
        status: formData.get('status'),
        notes: formData.get('notes'),
    });

    if (!parsed.success) {
        return {
            success: false,
            errors: parsed.error.flatten().fieldErrors,
        };
    }

    try {
        const response = await fetch(`${process.env.API_URL}/orders/${orderId}`, {
            method: 'PATCH',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(parsed.data),
        });

        if (!response.ok) {
            throw new Error('Failed to update order');
        }

        // Revalidate specific paths
        revalidatePath(`/orders/${orderId}`);
        revalidatePath('/orders');

        // Or revalidate by tag
        revalidateTag('orders');

        return { success: true };
    } catch (error) {
        return {
            success: false,
            errors: { _form: ['Failed to update order'] },
        };
    }
}
```

---

## State Management Patterns

### Server Components for Data (PRIMARY)

Use Server Components for **all server data**:
- Fetching: async/await in page.tsx
- Caching: Next.js automatic cache
- Revalidation: revalidatePath/revalidateTag

```typescript
// ✅ CORRECT - Server Component fetches data
// app/orders/page.tsx
export default async function OrdersPage() {
    const orders = await fetch(`${process.env.API_URL}/orders`, {
        next: { revalidate: 60 }, // Cache for 60 seconds
    }).then(res => res.json());

    return <OrdersList orders={orders} />;
}
```

### useState for UI State

Use `useState` for **local UI state only**:
- Form inputs
- Modal open/closed
- Selected tab
- Temporary UI flags

```typescript
// ✅ CORRECT - useState for UI state
'use client';

export function OrderFilters() {
    const [modalOpen, setModalOpen] = useState(false);
    const [selectedTab, setSelectedTab] = useState(0);

    return (
        // ... UI
    );
}
```

### URL Search Params for Sharable State

Use search params for state that should be shareable/bookmarkable:

```typescript
// app/orders/page.tsx
interface PageProps {
    searchParams: { status?: string; page?: string };
}

export default async function OrdersPage({ searchParams }: PageProps) {
    const status = searchParams.status || 'all';
    const page = Number(searchParams.page) || 1;

    const orders = await fetch(
        `${process.env.API_URL}/orders?status=${status}&page=${page}`
    ).then(res => res.json());

    return <OrdersList orders={orders} />;
}
```

**Avoid prop drilling** - pass data from Server Component to Client Component directly.

---

## Toast Notifications Pattern

### Using Sonner for Toasts

```typescript
// features/orders/components/delete-order-button.tsx
'use client';

import { useTransition } from 'react';
import { Button } from '@/components/ui/button';
import { toast } from 'sonner';
import { deleteOrder } from '@/actions/order-actions';

interface DeleteOrderButtonProps {
    orderId: string;
}

export function DeleteOrderButton({ orderId }: DeleteOrderButtonProps) {
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
            variant="destructive"
            onClick={handleDelete}
            disabled={isPending}
        >
            {isPending ? 'Deleting...' : 'Delete'}
        </Button>
    );
}
```

---

## Summary

**Common Patterns:**
- ✅ `auth()` for Server Components, `useSession()` for Client Components
- ✅ Server Actions in actions/*.ts for all mutations
- ✅ Zod validation in Server Actions
- ✅ ShadCN Dialog components with forms
- ✅ Client-side search/filter hooks
- ✅ useTransition for pending states
- ✅ revalidatePath/revalidateTag after mutations
- ✅ Server Components for data fetching
- ✅ useState for UI state only
- ✅ Search params for sharable state
- ✅ Sonner for toast notifications

**See Also:**
- [data-fetching.md](data-fetching.md) - Server Component patterns
- [component-patterns.md](component-patterns.md) - Component structure
- [loading-and-error-states.md](loading-and-error-states.md) - Error handling
