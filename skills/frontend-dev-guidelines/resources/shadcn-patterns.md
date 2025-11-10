# ShadCN UI Patterns

Deep dive into ShadCN UI composition patterns, wrapper components, and best practices for Next.js 15 App Router applications.

---

## Philosophy: Composition Over Configuration

ShadCN differs from traditional component libraries (like MUI):

**Traditional Libraries (MUI):**
```typescript
// ❌ Large prop APIs, configuration-heavy
<Button
    variant="contained"
    color="primary"
    startIcon={<Icon />}
    disabled={loading}
    onClick={handleClick}
    sx={{ mt: 2, px: 4 }}
>
    Submit
</Button>
```

**ShadCN Approach:**
```typescript
// ✅ Composition, smaller primitives
<Button
    variant="default"
    disabled={loading}
    onClick={handleClick}
    className="mt-2 px-4"
>
    <Icon className="mr-2 h-4 w-4" />
    Submit
</Button>
```

**Key Differences:**
- ShadCN components are **copied into your project** (not npm dependencies)
- You **own the code** - customize directly
- **Smaller API surface** - compose instead of configure
- **Tailwind-first** - no runtime CSS-in-JS

---

## The Wrapper Pattern Problem

### Why Wrappers?

**Without wrappers (repetitive):**

```typescript
// ❌ Repeated in 15 files
import { Badge } from '@/components/ui/badge';

<Badge className="bg-green-500 text-white">Paid</Badge>
<Badge className="bg-red-500 text-white">Unpaid</Badge>
<Badge className="bg-yellow-500 text-white">Pending</Badge>
```

**Problems:**
- Duplicated styling logic
- Hard to maintain consistency
- No type safety for status values
- Large import trees

**With wrappers (DRY):**

```typescript
// ✅ Single source of truth
import { OrderPaymentBadge } from '@/features/orders/components/badges/OrderPaymentBadge';

<OrderPaymentBadge status="paid" />
<OrderPaymentBadge status="unpaid" />
<OrderPaymentBadge status="pending" />
```

**Benefits:**
- Type-safe status prop
- Single place to change styling
- Domain-specific naming
- Cleaner imports

---

## Wrapper Pattern Architecture

### Basic Wrapper Structure

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

**Pattern Elements:**
1. **Type-safe props** - union type for status
2. **Config object** - maps status to styling
3. **Single ShadCN import** - wrapped once
4. **Domain naming** - OrderPaymentBadge vs generic Badge

---

## Badge Wrapper Patterns

### Status Badge (Most Common)

```typescript
// features/orders/components/badges/OrderStatusBadge.tsx
import { Badge } from '@/components/ui/badge';
import { cn } from '@/lib/utils';

interface OrderStatusBadgeProps {
    status: 'pending' | 'processing' | 'completed' | 'cancelled';
    className?: string;
}

export function OrderStatusBadge({ status, className }: OrderStatusBadgeProps) {
    const variants = {
        pending: 'bg-yellow-100 text-yellow-800 hover:bg-yellow-200 border-yellow-300',
        processing: 'bg-blue-100 text-blue-800 hover:bg-blue-200 border-blue-300',
        completed: 'bg-green-100 text-green-800 hover:bg-green-200 border-green-300',
        cancelled: 'bg-red-100 text-red-800 hover:bg-red-200 border-red-300',
    };

    return (
        <Badge
            variant="outline"
            className={cn(variants[status], className)}
        >
            {status.charAt(0).toUpperCase() + status.slice(1)}
        </Badge>
    );
}
```

### Badge with Icon

```typescript
// features/orders/components/badges/PaymentMethodBadge.tsx
import { Badge } from '@/components/ui/badge';
import { CreditCard, Wallet, Building2 } from 'lucide-react';

interface PaymentMethodBadgeProps {
    method: 'card' | 'paypal' | 'bank_transfer';
}

export function PaymentMethodBadge({ method }: PaymentMethodBadgeProps) {
    const config = {
        card: {
            label: 'Credit Card',
            icon: CreditCard,
            className: 'bg-purple-100 text-purple-800',
        },
        paypal: {
            label: 'PayPal',
            icon: Wallet,
            className: 'bg-blue-100 text-blue-800',
        },
        bank_transfer: {
            label: 'Bank Transfer',
            icon: Building2,
            className: 'bg-gray-100 text-gray-800',
        },
    };

    const { label, icon: Icon, className } = config[method];

    return (
        <Badge variant="outline" className={className}>
            <Icon className="mr-1 h-3 w-3" />
            {label}
        </Badge>
    );
}
```

---

## Button Wrapper Patterns

### Action Button with Icons

```typescript
// features/orders/components/buttons/OrderActionButton.tsx
import { Button } from '@/components/ui/button';
import { Check, X, Clock, Ban } from 'lucide-react';
import type { LucideIcon } from 'lucide-react';

interface OrderActionButtonProps {
    action: 'approve' | 'reject' | 'pending' | 'cancel';
    onClick: () => void;
    disabled?: boolean;
    loading?: boolean;
}

export function OrderActionButton({
    action,
    onClick,
    disabled,
    loading,
}: OrderActionButtonProps) {
    const config: Record<string, {
        label: string;
        icon: LucideIcon;
        variant: 'default' | 'destructive' | 'outline' | 'secondary';
        className?: string;
    }> = {
        approve: {
            label: 'Approve',
            icon: Check,
            variant: 'default',
            className: 'bg-green-600 hover:bg-green-700 text-white',
        },
        reject: {
            label: 'Reject',
            icon: X,
            variant: 'destructive',
        },
        pending: {
            label: 'Mark Pending',
            icon: Clock,
            variant: 'outline',
        },
        cancel: {
            label: 'Cancel Order',
            icon: Ban,
            variant: 'secondary',
        },
    };

    const { label, icon: Icon, variant, className } = config[action];

    return (
        <Button
            variant={variant}
            onClick={onClick}
            disabled={disabled || loading}
            className={className}
        >
            <Icon className="mr-2 h-4 w-4" />
            {loading ? 'Processing...' : label}
        </Button>
    );
}
```

### Confirmation Button (with built-in state)

```typescript
// features/orders/components/buttons/DeleteOrderButton.tsx
'use client';

import { useState, useTransition } from 'react';
import { Button } from '@/components/ui/button';
import {
    AlertDialog,
    AlertDialogAction,
    AlertDialogCancel,
    AlertDialogContent,
    AlertDialogDescription,
    AlertDialogFooter,
    AlertDialogHeader,
    AlertDialogTitle,
    AlertDialogTrigger,
} from '@/components/ui/alert-dialog';
import { Trash2 } from 'lucide-react';
import { toast } from 'sonner';
import { deleteOrder } from '@/actions/order-actions';

interface DeleteOrderButtonProps {
    orderId: string;
    orderNumber: string;
}

export function DeleteOrderButton({ orderId, orderNumber }: DeleteOrderButtonProps) {
    const [open, setOpen] = useState(false);
    const [isPending, startTransition] = useTransition();

    function handleDelete() {
        startTransition(async () => {
            const result = await deleteOrder(orderId);

            if (result.success) {
                toast.success(`Order ${orderNumber} deleted`);
                setOpen(false);
            } else {
                toast.error(result.errors?._form?.[0] || 'Failed to delete');
            }
        });
    }

    return (
        <AlertDialog open={open} onOpenChange={setOpen}>
            <AlertDialogTrigger asChild>
                <Button variant="destructive" size="sm">
                    <Trash2 className="mr-2 h-4 w-4" />
                    Delete
                </Button>
            </AlertDialogTrigger>
            <AlertDialogContent>
                <AlertDialogHeader>
                    <AlertDialogTitle>Delete Order</AlertDialogTitle>
                    <AlertDialogDescription>
                        Are you sure you want to delete order {orderNumber}? This action cannot be undone.
                    </AlertDialogDescription>
                </AlertDialogHeader>
                <AlertDialogFooter>
                    <AlertDialogCancel>Cancel</AlertDialogCancel>
                    <AlertDialogAction
                        onClick={handleDelete}
                        disabled={isPending}
                        className="bg-red-600 hover:bg-red-700"
                    >
                        {isPending ? 'Deleting...' : 'Delete'}
                    </AlertDialogAction>
                </AlertDialogFooter>
            </AlertDialogContent>
        </AlertDialog>
    );
}
```

---

## Card Wrapper Patterns

### Stat Card

```typescript
// features/dashboard/components/cards/StatCard.tsx
import { Card, CardHeader, CardTitle, CardContent } from '@/components/ui/card';
import { LucideIcon } from 'lucide-react';
import { cn } from '@/lib/utils';

interface StatCardProps {
    title: string;
    value: string | number;
    icon: LucideIcon;
    trend?: {
        value: number;
        isPositive: boolean;
    };
    className?: string;
}

export function StatCard({ title, value, icon: Icon, trend, className }: StatCardProps) {
    return (
        <Card className={cn('hover:shadow-md transition-shadow', className)}>
            <CardHeader className="flex flex-row items-center justify-between pb-2 space-y-0">
                <CardTitle className="text-sm font-medium text-muted-foreground">
                    {title}
                </CardTitle>
                <Icon className="h-4 w-4 text-muted-foreground" />
            </CardHeader>
            <CardContent>
                <div className="text-2xl font-bold">{value}</div>
                {trend && (
                    <p className={cn(
                        'text-xs mt-1',
                        trend.isPositive ? 'text-green-600' : 'text-red-600'
                    )}>
                        {trend.isPositive ? '↑' : '↓'} {Math.abs(trend.value)}% from last month
                    </p>
                )}
            </CardContent>
        </Card>
    );
}
```

### List Item Card

```typescript
// features/orders/components/cards/OrderCard.tsx
import { Card, CardHeader, CardTitle, CardDescription, CardContent } from '@/components/ui/card';
import { OrderStatusBadge } from '../badges/OrderStatusBadge';
import { formatDate } from '@/lib/utils';
import type { Order } from '@/types/order';

interface OrderCardProps {
    order: Order;
    onClick?: () => void;
}

export function OrderCard({ order, onClick }: OrderCardProps) {
    return (
        <Card
            className="cursor-pointer hover:shadow-lg transition-shadow"
            onClick={onClick}
        >
            <CardHeader>
                <div className="flex items-center justify-between">
                    <CardTitle className="text-lg">
                        Order #{order.orderNumber}
                    </CardTitle>
                    <OrderStatusBadge status={order.status} />
                </div>
                <CardDescription>
                    {formatDate(order.createdAt)}
                </CardDescription>
            </CardHeader>
            <CardContent>
                <div className="space-y-2">
                    <div className="flex justify-between text-sm">
                        <span className="text-muted-foreground">Customer:</span>
                        <span className="font-medium">{order.customerName}</span>
                    </div>
                    <div className="flex justify-between text-sm">
                        <span className="text-muted-foreground">Total:</span>
                        <span className="font-bold">${order.total.toFixed(2)}</span>
                    </div>
                </div>
            </CardContent>
        </Card>
    );
}
```

---

## Dialog/Modal Wrapper Patterns

### Form Dialog

```typescript
// features/users/components/dialogs/AddUserDialog.tsx
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
import { toast } from 'sonner';
import { createUser } from '@/actions/user-actions';

export function AddUserDialog() {
    const [open, setOpen] = useState(false);
    const [isPending, startTransition] = useTransition();
    const [errors, setErrors] = useState<Record<string, string[]>>({});

    async function handleSubmit(formData: FormData) {
        startTransition(async () => {
            const result = await createUser(formData);

            if (result.success) {
                toast.success('User created successfully');
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
                            <p className="text-sm text-red-500 mt-1">{errors.name[0]}</p>
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
                            <p className="text-sm text-red-500 mt-1">{errors.email[0]}</p>
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

## When to Extract vs Inline

### Extract to Wrapper When:

**✅ Used 3+ times across features**
```typescript
// OrderPaymentBadge used in:
// - Orders list page
// - Order detail page
// - Invoice page
// - Payment history
// → EXTRACT
```

**✅ Complex configuration logic**
```typescript
// Badge with 5+ status values + icons + colors
// → EXTRACT (too complex to inline)
```

**✅ Domain-specific naming improves clarity**
```typescript
// OrderStatusBadge is clearer than:
// <Badge className={getStatusColor(status)}>{status}</Badge>
```

### Keep Inline When:

**❌ Used only 1-2 times**
```typescript
// One-off badge in single component
// → INLINE (premature abstraction)
<Badge className="bg-blue-500">Special</Badge>
```

**❌ Simple, no logic**
```typescript
// Just styling, no config
<Button variant="outline">Click</Button>
```

**❌ Highly context-specific**
```typescript
// Needs props from parent context
// → INLINE or use children pattern
```

---

## Advanced Patterns

### Polymorphic Wrapper (with `asChild`)

```typescript
// components/domain/links/OrderLink.tsx
import { Button } from '@/components/ui/button';
import { ExternalLink } from 'lucide-react';
import Link from 'next/link';

interface OrderLinkProps {
    orderId: string;
    children?: React.ReactNode;
}

export function OrderLink({ orderId, children }: OrderLinkProps) {
    return (
        <Button variant="link" size="sm" asChild>
            <Link href={`/orders/${orderId}`}>
                {children || `Order #${orderId}`}
                <ExternalLink className="ml-1 h-3 w-3" />
            </Link>
        </Button>
    );
}
```

### Compound Wrapper Pattern

```typescript
// features/dashboard/components/metric-cards/MetricGroup.tsx
import { Card } from '@/components/ui/card';

export function MetricGroup({ children }: { children: React.ReactNode }) {
    return (
        <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
            {children}
        </div>
    );
}

MetricGroup.Card = function MetricCard({
    title,
    value,
    icon: Icon,
}: {
    title: string;
    value: string;
    icon: React.ComponentType<{ className?: string }>;
}) {
    return (
        <Card className="p-6">
            <div className="flex items-center justify-between">
                <div>
                    <p className="text-sm text-muted-foreground">{title}</p>
                    <p className="text-3xl font-bold mt-2">{value}</p>
                </div>
                <Icon className="h-8 w-8 text-muted-foreground" />
            </div>
        </Card>
    );
};

// Usage:
// <MetricGroup>
//   <MetricGroup.Card title="Revenue" value="$12,345" icon={DollarSign} />
//   <MetricGroup.Card title="Orders" value="456" icon={ShoppingCart} />
// </MetricGroup>
```

---

## Testing Wrapper Components

```typescript
// features/orders/components/badges/OrderStatusBadge.test.tsx
import { render, screen } from '@testing-library/react';
import { OrderStatusBadge } from './OrderStatusBadge';

describe('OrderStatusBadge', () => {
    it('renders pending status', () => {
        render(<OrderStatusBadge status="pending" />);
        expect(screen.getByText('Pending')).toBeInTheDocument();
    });

    it('applies correct styling for completed', () => {
        const { container } = render(<OrderStatusBadge status="completed" />);
        const badge = container.firstChild;
        expect(badge).toHaveClass('bg-green-100', 'text-green-800');
    });

    it('accepts custom className', () => {
        const { container } = render(
            <OrderStatusBadge status="pending" className="ml-4" />
        );
        expect(container.firstChild).toHaveClass('ml-4');
    });
});
```

---

## Summary

**ShadCN Wrapper Pattern Checklist:**

1. **Identify repetition** - same Badge/Button/Card used 3+ times
2. **Extract wrapper** - domain-specific component
3. **Start in features/** - avoid premature abstraction
4. **Move to components/ at 3+ uses** - when truly shared
5. **Keep config objects** - single source of truth
6. **Type-safe props** - union types for variants
7. **Allow customization** - className prop escape hatch
8. **Test thoroughly** - ensure all variants render correctly

**Benefits:**
- ✅ Single source of truth for styling
- ✅ Type-safe domain-specific props
- ✅ Cleaner component trees
- ✅ Easier to maintain consistency
- ✅ Reduced bundle size (fewer imports)

**See Also:**
- [component-patterns.md](component-patterns.md) - Server/Client Components
- [styling-guide.md](styling-guide.md) - Tailwind patterns
- [complete-examples.md](complete-examples.md) - Full examples
