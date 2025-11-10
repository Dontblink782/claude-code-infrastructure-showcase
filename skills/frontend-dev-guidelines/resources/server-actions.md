# Server Actions

Deep dive into Next.js 15 Server Actions - the modern replacement for API routes. Learn file structure, validation, error handling, and advanced patterns.

---

## What Are Server Actions?

**Server Actions** are asynchronous functions that run on the server. They replace traditional API routes and enable direct database access from forms and client components.

**Key Benefits:**
- No API route boilerplate
- Direct database access (Prisma, SQL, etc.)
- Automatic request serialization
- Built-in CSRF protection
- Progressive enhancement (works without JS)
- Type-safe end-to-end

**Traditional API Route (Old Way):**
```typescript
// ❌ OLD: app/api/orders/route.ts
export async function POST(request: Request) {
    const body = await request.json();
    const order = await prisma.order.create({ data: body });
    return Response.json(order);
}

// Client component
const response = await fetch('/api/orders', {
    method: 'POST',
    body: JSON.stringify(data),
});
```

**Server Action (Modern Way):**
```typescript
// ✅ NEW: actions/orders.ts
'use server';

export async function createOrder(formData: FormData) {
    const order = await prisma.order.create({ data: extractData(formData) });
    revalidatePath('/orders');
    return order;
}

// Client component
import { createOrder } from '@/actions/orders';
await createOrder(formData);
```

---

## File Structure

### Separate Action Files (Recommended)

**Create dedicated action files in `actions/` directory:**

```
actions/
  order-actions.ts     # CRUD for orders
  user-actions.ts      # User management
  auth-actions.ts      # Authentication
  payment-actions.ts   # Payment processing
```

**Each file starts with `'use server'` directive:**

```typescript
// actions/order-actions.ts
'use server';

export async function createOrder(formData: FormData) { /* ... */ }
export async function updateOrder(id: string, formData: FormData) { /* ... */ }
export async function deleteOrder(id: string) { /* ... */ }
```

### Why Separate Files?

**✅ Benefits:**
- Clear separation of concerns
- Easy to find and maintain
- Reusable across components
- Can add auth checks in one place
- Better code organization

**❌ Avoid Inline 'use server':**
```typescript
// ❌ DON'T: Inline Server Actions
export function MyComponent() {
    async function handleSubmit(formData: FormData) {
        'use server'; // Inline - harder to test/reuse
        await createOrder(formData);
    }

    return <form action={handleSubmit}>...</form>;
}
```

---

## Basic Server Action Pattern

### CRUD Operations

```typescript
// actions/order-actions.ts
'use server';

import { prisma } from '@/lib/db';
import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';

// CREATE
export async function createOrder(formData: FormData) {
    const order = await prisma.order.create({
        data: {
            customerId: formData.get('customerId') as string,
            amount: parseFloat(formData.get('amount') as string),
            status: 'pending',
        },
    });

    revalidatePath('/orders');
    redirect(`/orders/${order.id}`);
}

// READ (data fetching)
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

// UPDATE
export async function updateOrder(id: string, formData: FormData) {
    const status = formData.get('status') as string;

    await prisma.order.update({
        where: { id },
        data: { status },
    });

    revalidatePath('/orders');
    revalidatePath(`/orders/${id}`);
}

// DELETE
export async function deleteOrder(id: string) {
    await prisma.order.delete({
        where: { id },
    });

    revalidatePath('/orders');
}
```

---

## FormData Extraction

### Basic Extraction

```typescript
'use server';

export async function createUser(formData: FormData) {
    // String fields
    const name = formData.get('name') as string;
    const email = formData.get('email') as string;

    // Number fields
    const age = Number(formData.get('age'));

    // Boolean fields
    const isActive = formData.get('isActive') === 'true';

    // Optional fields
    const phone = formData.get('phone') as string | null;

    await prisma.user.create({
        data: { name, email, age, isActive, phone },
    });
}
```

### Helper Function for Extraction

```typescript
// lib/form-utils.ts
export function extractFormData<T extends Record<string, any>>(
    formData: FormData,
    schema: { [K in keyof T]: 'string' | 'number' | 'boolean' }
): T {
    const result = {} as T;

    for (const [key, type] of Object.entries(schema)) {
        const value = formData.get(key);

        if (type === 'number') {
            result[key as keyof T] = Number(value) as any;
        } else if (type === 'boolean') {
            result[key as keyof T] = (value === 'true') as any;
        } else {
            result[key as keyof T] = value as any;
        }
    }

    return result;
}

// Usage:
const data = extractFormData(formData, {
    name: 'string',
    email: 'string',
    age: 'number',
    isActive: 'boolean',
});
```

---

## Validation with Zod

### Schema-Based Validation

```typescript
// actions/user-actions.ts
'use server';

import { z } from 'zod';
import { prisma } from '@/lib/db';
import { revalidatePath } from 'next/cache';

// Define validation schema
const createUserSchema = z.object({
    name: z.string().min(2, 'Name must be at least 2 characters'),
    email: z.string().email('Invalid email address'),
    age: z.number().min(18, 'Must be 18 or older').max(120, 'Invalid age'),
    phone: z.string().regex(/^\d{10}$/, 'Phone must be 10 digits').optional(),
});

export async function createUser(formData: FormData) {
    // Extract raw data
    const rawData = {
        name: formData.get('name'),
        email: formData.get('email'),
        age: Number(formData.get('age')),
        phone: formData.get('phone') || undefined,
    };

    // Validate with Zod
    const parsed = createUserSchema.safeParse(rawData);

    if (!parsed.success) {
        // Return validation errors
        return {
            success: false,
            errors: parsed.error.flatten().fieldErrors,
        };
    }

    // Data is now type-safe and validated
    try {
        const user = await prisma.user.create({
            data: parsed.data,
        });

        revalidatePath('/users');

        return {
            success: true,
            user,
        };
    } catch (error) {
        return {
            success: false,
            errors: { _form: ['Failed to create user'] },
        };
    }
}
```

### Reusable Validation Schemas

```typescript
// lib/validations/user.ts
import { z } from 'zod';

export const userSchema = {
    create: z.object({
        name: z.string().min(2).max(100),
        email: z.string().email(),
        age: z.number().min(18).max(120),
        role: z.enum(['user', 'admin']),
    }),
    update: z.object({
        name: z.string().min(2).max(100).optional(),
        email: z.string().email().optional(),
        age: z.number().min(18).max(120).optional(),
    }),
};

// actions/user-actions.ts
import { userSchema } from '@/lib/validations/user';

export async function createUser(formData: FormData) {
    const parsed = userSchema.create.safeParse({
        name: formData.get('name'),
        email: formData.get('email'),
        age: Number(formData.get('age')),
        role: formData.get('role'),
    });

    if (!parsed.success) {
        return { success: false, errors: parsed.error.flatten().fieldErrors };
    }

    // ... create user
}
```

---

## Error Handling Patterns

### Return Success/Error Objects

```typescript
'use server';

import { z } from 'zod';

export async function createOrder(formData: FormData) {
    // Validation errors
    const parsed = orderSchema.safeParse(extractData(formData));
    if (!parsed.success) {
        return {
            success: false,
            errors: parsed.error.flatten().fieldErrors,
        };
    }

    // Database errors
    try {
        const order = await prisma.order.create({
            data: parsed.data,
        });

        revalidatePath('/orders');

        return {
            success: true,
            order,
        };
    } catch (error) {
        console.error('Order creation failed:', error);

        return {
            success: false,
            errors: { _form: ['Failed to create order. Please try again.'] },
        };
    }
}
```

### Client-Side Error Handling

```typescript
// features/orders/components/create-order-form.tsx
'use client';

import { useState, useTransition } from 'react';
import { createOrder } from '@/actions/order-actions';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';

export function CreateOrderForm() {
    const [isPending, startTransition] = useTransition();
    const [errors, setErrors] = useState<Record<string, string[]>>({});

    async function handleSubmit(formData: FormData) {
        startTransition(async () => {
            const result = await createOrder(formData);

            if (result.success) {
                setErrors({});
                // Success! Maybe show toast or redirect
            } else if (result.errors) {
                setErrors(result.errors);
            }
        });
    }

    return (
        <form action={handleSubmit} className="space-y-4">
            <div>
                <Label htmlFor="customerId">Customer ID</Label>
                <Input id="customerId" name="customerId" />
                {errors.customerId && (
                    <p className="text-sm text-red-500 mt-1">{errors.customerId[0]}</p>
                )}
            </div>

            <div>
                <Label htmlFor="amount">Amount</Label>
                <Input id="amount" name="amount" type="number" />
                {errors.amount && (
                    <p className="text-sm text-red-500 mt-1">{errors.amount[0]}</p>
                )}
            </div>

            {errors._form && (
                <p className="text-sm text-red-500">{errors._form[0]}</p>
            )}

            <Button type="submit" disabled={isPending}>
                {isPending ? 'Creating...' : 'Create Order'}
            </Button>
        </form>
    );
}
```

---

## Authentication in Server Actions

### Check Auth Before Mutations

```typescript
// actions/order-actions.ts
'use server';

import { auth } from '@/lib/auth';
import { prisma } from '@/lib/db';
import { revalidatePath } from 'next/cache';

export async function createOrder(formData: FormData) {
    // Check authentication
    const session = await auth();
    if (!session?.user) {
        return {
            success: false,
            errors: { _form: ['You must be logged in'] },
        };
    }

    // Check authorization (role-based)
    if (session.user.role !== 'admin') {
        return {
            success: false,
            errors: { _form: ['Unauthorized'] },
        };
    }

    // Proceed with mutation
    const order = await prisma.order.create({
        data: {
            userId: session.user.id, // Attach user ID
            // ... other fields
        },
    });

    revalidatePath('/orders');
    return { success: true, order };
}
```

### Reusable Auth Wrapper

```typescript
// lib/auth-wrapper.ts
import { auth } from '@/lib/auth';

export async function requireAuth() {
    const session = await auth();

    if (!session?.user) {
        throw new Error('Unauthorized - must be logged in');
    }

    return session.user;
}

export async function requireRole(role: string) {
    const user = await requireAuth();

    if (user.role !== role) {
        throw new Error(`Unauthorized - requires ${role} role`);
    }

    return user;
}

// actions/order-actions.ts
'use server';

import { requireAuth } from '@/lib/auth-wrapper';

export async function createOrder(formData: FormData) {
    const user = await requireAuth(); // Throws if not authenticated

    const order = await prisma.order.create({
        data: {
            userId: user.id,
            // ... other fields
        },
    });

    return { success: true, order };
}
```

---

## Revalidation Strategies

### revalidatePath()

**Revalidate specific pages after data changes:**

```typescript
'use server';

import { revalidatePath } from 'next/cache';

export async function createOrder(formData: FormData) {
    const order = await prisma.order.create({ /* ... */ });

    // Revalidate list page
    revalidatePath('/orders');

    // Revalidate detail page
    revalidatePath(`/orders/${order.id}`);

    // Revalidate nested routes
    revalidatePath('/dashboard'); // All /dashboard/* routes

    return { success: true, order };
}
```

### revalidateTag()

**Tag-based revalidation for related data:**

```typescript
// Tagging fetches
export async function getOrders() {
    const res = await fetch('https://api.example.com/orders', {
        next: { tags: ['orders'] },
    });
    return res.json();
}

export async function getOrderStats() {
    const res = await fetch('https://api.example.com/stats/orders', {
        next: { tags: ['orders', 'stats'] },
    });
    return res.json();
}

// Revalidate all 'orders' tagged fetches
'use server';

import { revalidateTag } from 'next/cache';

export async function createOrder(formData: FormData) {
    const order = await prisma.order.create({ /* ... */ });

    // Revalidate all fetches tagged with 'orders'
    revalidateTag('orders');

    return { success: true, order };
}
```

### When to Revalidate

**After CREATE:** Revalidate list pages
```typescript
revalidatePath('/orders'); // List page
```

**After UPDATE:** Revalidate detail + list
```typescript
revalidatePath(`/orders/${id}`); // Detail page
revalidatePath('/orders');       // List page
```

**After DELETE:** Revalidate list, redirect
```typescript
revalidatePath('/orders');
redirect('/orders');
```

---

## Progressive Enhancement

### Forms Work Without JavaScript

```typescript
// actions/order-actions.ts
'use server';

import { redirect } from 'next/navigation';

export async function createOrder(formData: FormData) {
    const order = await prisma.order.create({ /* ... */ });

    revalidatePath('/orders');

    // Redirect works even without JS
    redirect(`/orders/${order.id}`);
}

// Component
export function CreateOrderForm() {
    return (
        <form action={createOrder}>
            <input name="customerId" required />
            <input name="amount" type="number" required />
            <button type="submit">Create</button>
        </form>
    );
}
```

**Without JS:** Form submits, creates order, redirects
**With JS:** Add `useTransition` for pending UI

```typescript
'use client';

import { useTransition } from 'react';
import { createOrder } from '@/actions/order-actions';

export function CreateOrderForm() {
    const [isPending, startTransition] = useTransition();

    return (
        <form
            action={(formData) => {
                startTransition(async () => {
                    await createOrder(formData);
                });
            }}
        >
            <input name="customerId" required />
            <input name="amount" type="number" required />
            <button type="submit" disabled={isPending}>
                {isPending ? 'Creating...' : 'Create'}
            </button>
        </form>
    );
}
```

---

## Advanced Patterns

### Binding Arguments with .bind()

**Pass additional arguments to Server Actions:**

```typescript
// actions/order-actions.ts
'use server';

export async function updateOrder(id: string, formData: FormData) {
    const status = formData.get('status') as string;

    await prisma.order.update({
        where: { id },
        data: { status },
    });

    revalidatePath('/orders');
}

// Component
'use client';

import { updateOrder } from '@/actions/order-actions';

interface OrderActionsProps {
    orderId: string;
}

export function OrderActions({ orderId }: OrderActionsProps) {
    // Bind orderId to updateOrder
    const updateWithId = updateOrder.bind(null, orderId);

    return (
        <form action={updateWithId}>
            <select name="status">
                <option value="pending">Pending</option>
                <option value="completed">Completed</option>
            </select>
            <button type="submit">Update</button>
        </form>
    );
}
```

### useTransition for Pending States

```typescript
'use client';

import { useTransition } from 'react';
import { deleteOrder } from '@/actions/order-actions';
import { Button } from '@/components/ui/button';

interface DeleteButtonProps {
    orderId: string;
}

export function DeleteButton({ orderId }: DeleteButtonProps) {
    const [isPending, startTransition] = useTransition();

    function handleDelete() {
        startTransition(async () => {
            await deleteOrder(orderId);
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

### Optimistic Updates

```typescript
'use client';

import { useOptimistic, useTransition } from 'react';
import { updateOrderStatus } from '@/actions/order-actions';
import type { Order } from '@/types/order';

interface OrderListProps {
    orders: Order[];
}

export function OrderList({ orders }: OrderListProps) {
    const [isPending, startTransition] = useTransition();
    const [optimisticOrders, updateOptimisticOrders] = useOptimistic(
        orders,
        (state, updatedOrder: Order) => {
            return state.map((order) =>
                order.id === updatedOrder.id ? updatedOrder : order
            );
        }
    );

    function handleStatusChange(order: Order, newStatus: string) {
        // Optimistically update UI
        updateOptimisticOrders({ ...order, status: newStatus });

        // Then update server
        startTransition(async () => {
            await updateOrderStatus(order.id, newStatus);
        });
    }

    return (
        <div>
            {optimisticOrders.map((order) => (
                <div key={order.id}>
                    <p>{order.id}</p>
                    <p>{order.status}</p>
                    <button onClick={() => handleStatusChange(order, 'completed')}>
                        Complete
                    </button>
                </div>
            ))}
        </div>
    );
}
```

---

## Testing Server Actions

### Unit Testing

```typescript
// actions/order-actions.test.ts
import { describe, it, expect, vi } from 'vitest';
import { createOrder } from './order-actions';
import { prisma } from '@/lib/db';

vi.mock('@/lib/db', () => ({
    prisma: {
        order: {
            create: vi.fn(),
        },
    },
}));

describe('createOrder', () => {
    it('creates order with valid data', async () => {
        const mockOrder = { id: '1', customerId: 'c1', amount: 100 };
        vi.mocked(prisma.order.create).mockResolvedValue(mockOrder);

        const formData = new FormData();
        formData.set('customerId', 'c1');
        formData.set('amount', '100');

        const result = await createOrder(formData);

        expect(result.success).toBe(true);
        expect(result.order).toEqual(mockOrder);
    });

    it('returns error for invalid data', async () => {
        const formData = new FormData();
        formData.set('customerId', '');
        formData.set('amount', '-10');

        const result = await createOrder(formData);

        expect(result.success).toBe(false);
        expect(result.errors).toBeDefined();
    });
});
```

---

## Summary

**Server Actions Best Practices:**

1. **File Structure**: Separate files in `actions/` directory
2. **Validation**: Always use Zod schemas
3. **Error Handling**: Return success/error objects
4. **Authentication**: Check auth before mutations
5. **Revalidation**: Use revalidatePath/revalidateTag after changes
6. **Progressive Enhancement**: Forms work without JS
7. **TypeScript**: Full type safety end-to-end
8. **Testing**: Unit test validation and business logic

**Server Action Recipe:**

```typescript
'use server';

import { z } from 'zod';
import { auth } from '@/lib/auth';
import { prisma } from '@/lib/db';
import { revalidatePath } from 'next/cache';

const schema = z.object({
    // validation
});

export async function myAction(formData: FormData) {
    // 1. Auth check
    const user = await auth();
    if (!user) return { success: false, errors: { _form: ['Unauthorized'] } };

    // 2. Validate
    const parsed = schema.safeParse(extractData(formData));
    if (!parsed.success) return { success: false, errors: parsed.error.flatten().fieldErrors };

    // 3. Mutation
    try {
        const result = await prisma.model.create({ data: parsed.data });

        // 4. Revalidate
        revalidatePath('/path');

        return { success: true, result };
    } catch (error) {
        return { success: false, errors: { _form: ['Failed'] } };
    }
}
```

**See Also:**
- [data-fetching.md](data-fetching.md) - Fetching patterns
- [common-patterns.md](common-patterns.md) - Forms and validation
- [component-patterns.md](component-patterns.md) - Server/Client components
- [complete-examples.md](complete-examples.md) - Full examples
