# TypeScript Standards

TypeScript best practices for type safety and maintainability in Next.js 15 App Router applications.

---

## Strict Mode

### Configuration

TypeScript strict mode is **enabled** in Next.js projects:

```json
// tsconfig.json
{
    "compilerOptions": {
        "strict": true,
        "noImplicitAny": true,
        "strictNullChecks": true,
        "target": "ES2017",
        "lib": ["dom", "dom.iterable", "esnext"],
        "module": "esnext",
        "jsx": "preserve"
    }
}
```

**This means:**
- No implicit `any` types
- Null/undefined must be handled explicitly
- Type safety enforced
- Modern JavaScript features

---

## No `any` Type

### The Rule

```typescript
// ❌ NEVER use any
function handleData(data: any) {
    return data.something;
}

// ✅ Use specific types
interface OrderData {
    id: string;
    amount: number;
    status: string;
}

function handleData(data: OrderData) {
    return data.amount;
}

// ✅ Or use unknown for truly unknown data
function handleUnknown(data: unknown) {
    if (typeof data === 'object' && data !== null && 'amount' in data) {
        return (data as OrderData).amount;
    }
    throw new Error('Invalid data');
}
```

**If you truly don't know the type:**
- Use `unknown` (forces type checking)
- Use type guards to narrow
- Document why type is unknown

---

## Server Component Typing

### Async Server Components

```typescript
// app/orders/page.tsx
import type { Order } from '@/types/order';

// Server Components can be async
export default async function OrdersPage() {
    const orders: Order[] = await getOrders();

    return <OrdersList orders={orders} />;
}
```

### Params and SearchParams

```typescript
// app/orders/[id]/page.tsx
interface PageProps {
    params: { id: string };
    searchParams: { status?: string; page?: string };
}

export default async function OrderDetailPage({
    params,
    searchParams,
}: PageProps) {
    const orderId: string = params.id;
    const status: string | undefined = searchParams.status;
    const page: number = Number(searchParams.page) || 1;

    const order = await getOrder(orderId);

    return <OrderDetails order={order} />;
}
```

### generateMetadata Typing

```typescript
// app/orders/[id]/page.tsx
import type { Metadata } from 'next';

interface PageProps {
    params: { id: string };
}

export async function generateMetadata({
    params,
}: PageProps): Promise<Metadata> {
    const order = await getOrder(params.id);

    return {
        title: `Order ${order.orderNumber}`,
        description: `Order details for ${order.customerName}`,
    };
}
```

---

## Client Component Typing

### No React.FC (Deprecated Pattern)

```typescript
// ❌ OLD - React.FC was common in React 17
export const OrderCard: React.FC<OrderCardProps> = ({ order }) => {
    return <div>{order.id}</div>;
};

// ✅ NEW - Direct function typing (Next.js/React 18+)
interface OrderCardProps {
    order: Order;
    onClick?: (id: string) => void;
}

export function OrderCard({ order, onClick }: OrderCardProps) {
    return (
        <div onClick={() => onClick?.(order.id)}>
            {order.id}
        </div>
    );
}
```

**Why avoid React.FC:**
- No implicit children
- Cleaner syntax
- Better type inference
- Modern React pattern

### Props with Children

```typescript
interface ContainerProps {
    children: React.ReactNode;
    title: string;
    className?: string;
}

export function Container({ children, title, className }: ContainerProps) {
    return (
        <div className={className}>
            <h2 className="text-2xl font-bold">{title}</h2>
            {children}
        </div>
    );
}
```

---

## Type Imports

### Use 'type' Keyword

```typescript
// ✅ CORRECT - Explicitly mark as type import
import type { Order } from '@/types/order';
import type { User } from '@/types/user';

// ❌ AVOID - Mixed value and type imports
import { Order } from '@/types/order';  // Unclear if type or value
```

**Benefits:**
- Clearly separates types from values
- Better tree-shaking
- Prevents circular dependencies
- TypeScript compiler optimization

### Import Aliases

**Single @ alias pattern:**

```typescript
// ✅ CORRECT - Use @/ for all imports
import type { Order } from '@/types/order';
import { Button } from '@/components/ui/button';
import { getOrders } from '@/actions/orders';

// ❌ AVOID - Multiple aliases
import type { Order } from '~types/order';
import { Button } from '~components/ui/button';
```

---

## Server Action Typing

### FormData Extraction

```typescript
// actions/orders.ts
'use server';

import { z } from 'zod';

const orderSchema = z.object({
    customerId: z.string().min(1),
    amount: z.number().positive(),
    status: z.enum(['pending', 'completed']),
});

export async function createOrder(formData: FormData) {
    // Type-safe extraction with Zod
    const parsed = orderSchema.safeParse({
        customerId: formData.get('customerId'),
        amount: Number(formData.get('amount')),
        status: formData.get('status'),
    });

    if (!parsed.success) {
        return {
            success: false,
            errors: parsed.error.flatten().fieldErrors,
        };
    }

    // parsed.data is now type-safe
    const order = await prisma.order.create({
        data: parsed.data,
    });

    return { success: true, order };
}
```

### Return Type Unions

```typescript
'use server';

type ActionResult<T> =
    | { success: true; data: T }
    | { success: false; errors: Record<string, string[]> };

export async function updateOrder(
    id: string,
    formData: FormData
): Promise<ActionResult<Order>> {
    // Implementation...
    if (error) {
        return {
            success: false,
            errors: { _form: ['Failed to update'] },
        };
    }

    return {
        success: true,
        data: order,
    };
}
```

---

## Component Prop Interfaces

### Interface Pattern

```typescript
/**
 * Props for OrderCard component
 */
interface OrderCardProps {
    /** The order to display */
    order: Order;

    /** Optional callback when card is clicked */
    onClick?: (orderId: string) => void;

    /** Display mode */
    variant?: 'default' | 'compact';

    /** Additional CSS classes */
    className?: string;
}

export function OrderCard({
    order,
    onClick,
    variant = 'default',
    className,
}: OrderCardProps) {
    return (
        <div className={className} onClick={() => onClick?.(order.id)}>
            {variant === 'compact' ? (
                <span>{order.orderNumber}</span>
            ) : (
                <div>
                    <h3>{order.orderNumber}</h3>
                    <p>${order.amount}</p>
                </div>
            )}
        </div>
    );
}
```

**Key Points:**
- Separate interface for props
- JSDoc comments for each prop
- Optional props use `?`
- Provide defaults in destructuring

---

## Utility Types

### Partial<T>

```typescript
// Make all properties optional
type OrderUpdate = Partial<Order>;

export async function updateOrder(
    id: string,
    updates: Partial<Order>
) {
    // updates can have any subset of Order properties
    await prisma.order.update({
        where: { id },
        data: updates,
    });
}
```

### Pick<T, K>

```typescript
// Select specific properties
type OrderPreview = Pick<Order, 'id' | 'orderNumber' | 'amount'>;

export function OrderPreviewCard({ order }: { order: OrderPreview }) {
    return (
        <div>
            <p>{order.orderNumber}</p>
            <p>${order.amount}</p>
        </div>
    );
}
```

### Omit<T, K>

```typescript
// Exclude specific properties
type OrderWithoutAudit = Omit<Order, 'createdAt' | 'updatedAt'>;

export function createOrderFromInput(
    input: OrderWithoutAudit
): Promise<Order> {
    return prisma.order.create({
        data: input,
    });
}
```

### Required<T>

```typescript
// Make all properties required
type RequiredConfig = Required<Config>;  // All optional props become required
```

### Record<K, V>

```typescript
// Type-safe object/map
const orderStatusColors: Record<string, string> = {
    pending: 'bg-yellow-500',
    completed: 'bg-green-500',
    cancelled: 'bg-red-500',
};

// For variants
const buttonVariants: Record<
    'default' | 'destructive' | 'outline',
    string
> = {
    default: 'bg-primary text-primary-foreground',
    destructive: 'bg-destructive text-destructive-foreground',
    outline: 'border border-input',
};
```

---

## Type Guards

### Basic Type Guards

```typescript
function isOrder(data: unknown): data is Order {
    return (
        typeof data === 'object' &&
        data !== null &&
        'id' in data &&
        'orderNumber' in data &&
        'amount' in data
    );
}

// Usage
if (isOrder(response)) {
    console.log(response.orderNumber);  // TypeScript knows it's Order
}
```

### Discriminated Unions

```typescript
type LoadingState<T> =
    | { status: 'idle' }
    | { status: 'loading' }
    | { status: 'success'; data: T }
    | { status: 'error'; error: Error };

export function OrderDisplay({
    state,
}: {
    state: LoadingState<Order>;
}) {
    // TypeScript narrows type based on status
    if (state.status === 'success') {
        return <OrderDetails order={state.data} />;  // data available
    }

    if (state.status === 'error') {
        return <ErrorDisplay error={state.error} />;  // error available
    }

    return <LoadingSkeleton />;
}
```

---

## Generic Types

### Generic Functions

```typescript
function getById<T extends { id: string }>(
    items: T[],
    id: string
): T | undefined {
    return items.find(item => item.id === id);
}

// Usage with type inference
const orders: Order[] = [...];
const order = getById(orders, '123');  // Type: Order | undefined
```

### Generic Components

```typescript
interface ListProps<T> {
    items: T[];
    renderItem: (item: T) => React.ReactNode;
    keyExtractor: (item: T) => string;
}

export function List<T>({
    items,
    renderItem,
    keyExtractor,
}: ListProps<T>) {
    return (
        <div>
            {items.map(item => (
                <div key={keyExtractor(item)}>
                    {renderItem(item)}
                </div>
            ))}
        </div>
    );
}

// Usage
<List<Order>
    items={orders}
    renderItem={(order) => <OrderCard order={order} />}
    keyExtractor={(order) => order.id}
/>
```

---

## Prisma Type Integration

### Infer Types from Prisma

```typescript
import type { Order, Prisma } from '@prisma/client';

// Use Prisma's generated types
type OrderWithCustomer = Prisma.OrderGetPayload<{
    include: { customer: true };
}>;

export async function getOrderWithCustomer(
    id: string
): Promise<OrderWithCustomer> {
    return await prisma.order.findUnique({
        where: { id },
        include: { customer: true },
    });
}
```

---

## Null/Undefined Handling

### Optional Chaining

```typescript
// ✅ CORRECT
const name = order?.customer?.name;

// Equivalent to:
const name = order && order.customer && order.customer.name;
```

### Nullish Coalescing

```typescript
// ✅ CORRECT
const displayName = order?.customer?.name ?? 'Unknown';

// Only uses default if null or undefined
// (Different from || which triggers on '', 0, false)
const count = order?.items?.length ?? 0;
```

### Non-Null Assertion (Use Carefully)

```typescript
// ✅ OK - When you're certain value exists after check
if (!order) {
    throw new Error('Order not found');
}

const amount = order.amount!;  // We know it exists

// ⚠️ CAREFUL - Only use when you KNOW it's not null
// Better to check explicitly:
if (order.amount) {
    // Use order.amount
}
```

---

## Zod for Runtime Validation

### Schema Definition

```typescript
import { z } from 'zod';

export const orderSchema = z.object({
    customerId: z.string().uuid(),
    amount: z.number().positive(),
    status: z.enum(['pending', 'processing', 'completed', 'cancelled']),
    items: z.array(z.object({
        productId: z.string(),
        quantity: z.number().int().positive(),
        price: z.number().positive(),
    })),
});

// Infer TypeScript type from Zod schema
export type OrderInput = z.infer<typeof orderSchema>;
```

**Benefits:**
- Runtime validation
- Type inference from schema
- Single source of truth

---

## Summary

**TypeScript Checklist:**
- ✅ Strict mode enabled
- ✅ No `any` type (use `unknown` if needed)
- ✅ No `React.FC` (use direct function typing)
- ✅ Use `import type` for type imports
- ✅ Use single `@/` import alias
- ✅ JSDoc comments on prop interfaces
- ✅ Utility types (Partial, Pick, Omit, Required, Record)
- ✅ Type guards for narrowing
- ✅ Optional chaining and nullish coalescing
- ✅ Zod for runtime validation
- ❌ Avoid type assertions unless necessary

**Next.js Specific:**
- ✅ Type `params` and `searchParams` in page.tsx
- ✅ Type `generateMetadata` return as `Promise<Metadata>`
- ✅ Type Server Actions with FormData and return unions
- ✅ Use Prisma generated types
- ✅ Zod schemas for Server Action validation

**See Also:**
- [server-actions.md](server-actions.md) - Server Action typing
- [component-patterns.md](component-patterns.md) - Component typing
- [data-fetching.md](data-fetching.md) - API typing
- [complete-examples.md](complete-examples.md) - Typed examples
