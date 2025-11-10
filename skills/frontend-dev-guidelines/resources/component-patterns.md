# Component Patterns

Modern Next.js 15 App Router component architecture emphasizing Server Components, Client Components, and ShadCN UI composition patterns.

---

## Server Components vs Client Components

### The Default: Server Components

**Next.js 15 App Router makes Server Components the default.** All components are Server Components unless you add `'use client'` directive.

**Server Components characteristics:**
- Render on the server only
- Can be `async` functions
- Can directly access databases, file system, server-only APIs
- Cannot use React hooks (useState, useEffect, etc.)
- Cannot use browser APIs
- Automatic code splitting and streaming

**When to use Server Components:**
- Fetching data from database/APIs
- Accessing backend resources
- Keeping sensitive data on server (API keys, tokens)
- Reducing client bundle size
- Static content that doesn't need interactivity

**Basic Server Component Pattern:**

```typescript
// app/orders/page.tsx
import { getOrders } from '@/actions/orders';
import { OrdersTable } from '@/features/orders/components/OrdersTable';

export default async function OrdersPage() {
    // Fetch data directly in component
    const orders = await getOrders();

    return (
        <div className="container mx-auto py-8">
            <h1 className="text-3xl font-bold mb-6">Orders</h1>
            <OrdersTable orders={orders} />
        </div>
    );
}
```

**Key Points:**
- Component is async function
- No 'use client' directive
- Directly calls server action
- Passes data as props to presentational components

---

### Client Components

**Client Components** run in the browser and enable interactivity. Add `'use client'` directive at the top of the file.

**Client Components characteristics:**
- Render on client (browser)
- Can use React hooks (useState, useEffect, useContext, etc.)
- Can use browser APIs (window, document, localStorage)
- Handle user interactions (onClick, onChange, etc.)
- Cannot be async functions
- Cannot directly access server-only resources

**When to use Client Components:**
- Interactive UI (forms, buttons, inputs)
- Event listeners (onClick, onChange, onSubmit)
- React hooks (state, effects, context)
- Browser APIs (localStorage, geolocation, etc.)
- Third-party libraries that use browser APIs

**Basic Client Component Pattern:**

```typescript
// features/orders/components/OrdersTable.tsx
'use client';

import { useState } from 'react';
import {
    Table,
    TableBody,
    TableCell,
    TableHead,
    TableHeader,
    TableRow,
} from '@/components/ui/table';
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
                    <TableRow
                        key={order.id}
                        onClick={() => setSelectedId(order.id)}
                        className="cursor-pointer hover:bg-muted/50"
                    >
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

**Key Points:**
- `'use client'` directive at top
- Uses `useState` hook for interactivity
- Receives data via props (from Server Component)
- Handles user interactions

---

### Server/Client Boundary Rules

**Data Flow:**
```
Server Component (async, fetches data)
    ↓ (props)
Client Component (interactive, receives data)
```

**Important Rules:**
1. **Server can render Client**, but Client cannot render Server
2. **Pass serializable props only** (JSON-compatible: strings, numbers, arrays, objects)
3. **Cannot pass functions** from Server to Client (use Server Actions instead)
4. **Client Components stop "server-ness"** - all children become Client Components unless imported separately

**Example - Server renders Client:**

```typescript
// app/dashboard/page.tsx (Server Component)
import { getStats } from '@/actions/stats';
import { StatsCard } from '@/features/dashboard/components/StatsCard'; // Client

export default async function DashboardPage() {
    const stats = await getStats();

    return (
        <div>
            {/* Server Component rendering Client Component */}
            <StatsCard data={stats} />
        </div>
    );
}
```

**Example - Client renders Server (via children pattern):**

```typescript
// features/layout/ClientLayout.tsx
'use client';

import { useState } from 'react';

interface ClientLayoutProps {
    children: React.ReactNode; // Can be Server Component!
}

export function ClientLayout({ children }: ClientLayoutProps) {
    const [isOpen, setIsOpen] = useState(true);

    return (
        <div className={isOpen ? 'sidebar-open' : 'sidebar-closed'}>
            {/* children can still be Server Components */}
            {children}
        </div>
    );
}

// app/page.tsx (Server Component)
import { ClientLayout } from '@/features/layout/ClientLayout';

export default async function Page() {
    const data = await fetchData(); // Server-side fetch

    return (
        <ClientLayout>
            {/* This is still a Server Component! */}
            <ServerContent data={data} />
        </ClientLayout>
    );
}
```

---

## ShadCN UI Wrapper Pattern

### The Problem: Large Component Trees

When using ShadCN base components directly, you end up importing the same component repeatedly and duplicating styling logic:

```typescript
// ❌ AVOID - Repeated imports and logic
import { Badge } from '@/components/ui/badge';

// In 10 different files:
<Badge variant="default" className="bg-green-500">Paid</Badge>
<Badge variant="default" className="bg-red-500">Unpaid</Badge>
```

### The Solution: Wrapper Components

**Create wrapper components** that encapsulate ShadCN base components with domain-specific logic:

```typescript
// features/orders/components/badges/OrderPaymentBadge.tsx
import { Badge } from '@/components/ui/badge';

interface OrderPaymentBadgeProps {
    status: 'paid' | 'unpaid' | 'pending';
}

export function OrderPaymentBadge({ status }: OrderPaymentBadgeProps) {
    const config = {
        paid: {
            label: 'Paid',
            className: 'bg-green-500 hover:bg-green-600 text-white',
        },
        unpaid: {
            label: 'Unpaid',
            className: 'bg-red-500 hover:bg-red-600 text-white',
        },
        pending: {
            label: 'Pending',
            className: 'bg-yellow-500 hover:bg-yellow-600 text-white',
        },
    };

    const { label, className } = config[status];

    return <Badge className={className}>{label}</Badge>;
}
```

**Usage - Clean and simple:**

```typescript
// Now in components:
<OrderPaymentBadge status="paid" />
<OrderPaymentBadge status="unpaid" />
```

**Benefits:**
- Single source of truth for styling
- Type-safe props
- Domain-specific naming
- Easy to change globally
- Reduces bundle size (one Badge import internally)

---

### Wrapper Pattern Examples

**Button Wrapper:**

```typescript
// features/orders/components/buttons/OrderActionButton.tsx
import { Button } from '@/components/ui/button';
import { Check, X, Clock } from 'lucide-react';

interface OrderActionButtonProps {
    action: 'approve' | 'reject' | 'pending';
    onClick: () => void;
    disabled?: boolean;
}

export function OrderActionButton({
    action,
    onClick,
    disabled
}: OrderActionButtonProps) {
    const config = {
        approve: {
            label: 'Approve Order',
            icon: Check,
            variant: 'default' as const,
            className: 'bg-green-600 hover:bg-green-700',
        },
        reject: {
            label: 'Reject Order',
            icon: X,
            variant: 'destructive' as const,
        },
        pending: {
            label: 'Mark Pending',
            icon: Clock,
            variant: 'outline' as const,
        },
    };

    const { label, icon: Icon, variant, className } = config[action];

    return (
        <Button
            variant={variant}
            onClick={onClick}
            disabled={disabled}
            className={className}
        >
            <Icon className="mr-2 h-4 w-4" />
            {label}
        </Button>
    );
}
```

**Card Wrapper:**

```typescript
// features/dashboard/components/cards/StatCard.tsx
import { Card, CardHeader, CardTitle, CardContent } from '@/components/ui/card';
import { LucideIcon } from 'lucide-react';

interface StatCardProps {
    title: string;
    value: string | number;
    icon: LucideIcon;
    trend?: {
        value: number;
        isPositive: boolean;
    };
}

export function StatCard({ title, value, icon: Icon, trend }: StatCardProps) {
    return (
        <Card>
            <CardHeader className="flex flex-row items-center justify-between pb-2">
                <CardTitle className="text-sm font-medium text-muted-foreground">
                    {title}
                </CardTitle>
                <Icon className="h-4 w-4 text-muted-foreground" />
            </CardHeader>
            <CardContent>
                <div className="text-2xl font-bold">{value}</div>
                {trend && (
                    <p className={`text-xs ${trend.isPositive ? 'text-green-600' : 'text-red-600'}`}>
                        {trend.isPositive ? '+' : '-'}{Math.abs(trend.value)}% from last month
                    </p>
                )}
            </CardContent>
        </Card>
    );
}
```

---

### Hybrid Location Strategy

**Where to put wrapper components?**

**Start in features/ directory:**
```
features/
  orders/
    components/
      badges/
        OrderPaymentBadge.tsx
      buttons/
        OrderActionButton.tsx
```

**Move to components/ when used 3+ times:**

If `OrderPaymentBadge` is used in:
- Orders page
- Invoice page
- Payment history page

**Then move to shared:**
```
components/
  domain/
    badges/
      OrderPaymentBadge.tsx
```

**Update imports:**
```typescript
// Before (feature-specific)
import { OrderPaymentBadge } from '@/features/orders/components/badges/OrderPaymentBadge';

// After (shared)
import { OrderPaymentBadge } from '@/components/domain/badges/OrderPaymentBadge';
```

**Why this pattern?**
- Avoids premature abstraction
- Components start domain-specific
- Natural evolution to shared components
- Easy to refactor when needed

---

## Component Structure & Organization

### Recommended File Structure

```typescript
/**
 * Component description
 * What it does, when to use it
 */

// 1. CLIENT DIRECTIVE (if needed)
'use client';

// 2. IMPORTS
// React
import { useState, useCallback, useEffect } from 'react';

// ShadCN Components
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Card } from '@/components/ui/card';

// Custom Components
import { OrderPaymentBadge } from './badges/OrderPaymentBadge';

// Hooks
import { useSearchOrders } from '../hooks/useSearchOrders';

// Types
import type { Order } from '@/types/order';

// 3. PROPS INTERFACE
interface OrdersTableProps {
    /** Initial orders to display */
    orders: Order[];
    /** Callback when order is selected */
    onOrderSelect?: (orderId: string) => void;
}

// 4. COMPONENT
export function OrdersTable({ orders, onOrderSelect }: OrdersTableProps) {
    // Hooks first
    const [selectedId, setSelectedId] = useState<string | null>(null);
    const { searchOrders, results, isSearching } = useSearchOrders();

    // Event handlers (useCallback if passed to children)
    const handleSelect = useCallback((id: string) => {
        setSelectedId(id);
        onOrderSelect?.(id);
    }, [onOrderSelect]);

    // Effects
    useEffect(() => {
        // Setup
        return () => {
            // Cleanup
        };
    }, []);

    // Render
    return (
        <div className="space-y-4">
            {/* Component JSX */}
        </div>
    );
}
```

---

## Component Composition Patterns

### Compound Components

```typescript
// components/ui/card-group/CardGroup.tsx
interface CardGroupProps {
    children: React.ReactNode;
}

export function CardGroup({ children }: CardGroupProps) {
    return <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">{children}</div>;
}

// Subcomponents
CardGroup.Item = function CardGroupItem({ children }: { children: React.ReactNode }) {
    return <Card className="p-4">{children}</Card>;
};

// Usage
<CardGroup>
    <CardGroup.Item>Card 1</CardGroup.Item>
    <CardGroup.Item>Card 2</CardGroup.Item>
    <CardGroup.Item>Card 3</CardGroup.Item>
</CardGroup>
```

### Render Props (Rare, but useful)

```typescript
interface DataProviderProps<T> {
    data: T[];
    children: (data: T[], loading: boolean) => React.ReactNode;
}

export function DataProvider<T>({ data, children }: DataProviderProps<T>) {
    const [loading, setLoading] = useState(false);

    return <>{children(data, loading)}</>;
}

// Usage
<DataProvider data={orders}>
    {(data, loading) => (
        loading ? <LoadingSkeleton /> : <OrdersList orders={data} />
    )}
</DataProvider>
```

---

## Styling with Tailwind

### Inline Classes (Preferred)

```typescript
export function OrderCard({ order }: OrderCardProps) {
    return (
        <Card className="p-6 hover:shadow-lg transition-shadow">
            <h3 className="text-xl font-semibold mb-2">{order.title}</h3>
            <p className="text-muted-foreground">{order.description}</p>
        </Card>
    );
}
```

### Conditional Classes with cn() Helper

```typescript
import { cn } from '@/lib/utils';

export function OrderBadge({ status }: OrderBadgeProps) {
    return (
        <Badge
            className={cn(
                'font-medium',
                status === 'paid' && 'bg-green-500 text-white',
                status === 'unpaid' && 'bg-red-500 text-white',
                status === 'pending' && 'bg-yellow-500 text-white'
            )}
        >
            {status}
        </Badge>
    );
}
```

### Extract to const for Reusability

```typescript
const cardStyles = {
    base: 'rounded-lg border bg-card text-card-foreground shadow-sm',
    hover: 'hover:shadow-md transition-shadow',
    variants: {
        default: 'border-border',
        primary: 'border-primary',
        destructive: 'border-destructive',
    },
};

export function CustomCard({ variant = 'default', children }: CustomCardProps) {
    return (
        <div className={cn(cardStyles.base, cardStyles.hover, cardStyles.variants[variant])}>
            {children}
        </div>
    );
}
```

---

## Data Flow Patterns

### Container/Presentation Pattern

**Container (Server Component):**
Fetches data, handles business logic, passes to presentational components.

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

**Presentation (Client Component):**
Receives data as props, handles UI and interactions.

```typescript
// features/orders/components/OrdersTable.tsx
'use client';

interface OrdersTableProps {
    orders: Order[];
}

export function OrdersTable({ orders }: OrdersTableProps) {
    // UI logic only
    return <Table>...</Table>;
}
```

### Props Down, Events Up

```typescript
// Parent
function OrdersPage() {
    const handleOrderSelect = (id: string) => {
        console.log('Selected:', id);
    };

    return (
        <OrdersTable
            orders={orders}                    // Props down
            onOrderSelect={handleOrderSelect}  // Events up
        />
    );
}

// Child
interface OrdersTableProps {
    orders: Order[];
    onOrderSelect: (id: string) => void;
}

export function OrdersTable({ orders, onOrderSelect }: OrdersTableProps) {
    return (
        <div onClick={() => onOrderSelect(orders[0].id)}>
            {/* Content */}
        </div>
    );
}
```

---

## Summary

**Modern Component Recipe:**

1. **Default to Server Components** - async, fetch data directly
2. **Use 'use client' sparingly** - only for interactivity
3. **ShadCN Wrapper Pattern** - avoid large component imports
4. **Hybrid Location** - features/ first, components/ at 3+ uses
5. **Tailwind for Styling** - inline classes, cn() for conditionals
6. **Container/Presentation** - page.tsx fetches, components present
7. **Props down, events up** - clear data flow

**See Also:**
- [data-fetching.md](data-fetching.md) - Server Actions and data patterns
- [file-organization.md](file-organization.md) - Directory structure
- [shadcn-patterns.md](shadcn-patterns.md) - Advanced ShadCN patterns
- [complete-examples.md](complete-examples.md) - Full working examples
