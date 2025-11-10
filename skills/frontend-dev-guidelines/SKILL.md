---
name: frontend-dev-guidelines
description: Frontend development guidelines for Next.js 15 App Router applications with React/TypeScript. Modern patterns including Server Components, Server Actions, ShadCN UI, file organization with app router structure, data fetching with native Next.js, and performance optimization. Use when creating components, pages, features, fetching data, styling, routing, or working with Next.js frontend code.
---

# Frontend Development Guidelines

Comprehensive guide for Next.js 15 App Router development with Server Components, Server Actions, ShadCN UI, and TypeScript best practices.

---

## Quick Start Checklists

### New Component

- [ ] Server Component (default) or Client Component ('use client')?
- [ ] Server: async function, fetch data directly
- [ ] Client: Receive data as props, handle interactions
- [ ] ShadCN UI components for common patterns
- [ ] Domain wrappers in features/ (move to components/ at 3+ uses)
- [ ] Tailwind classes with cn() utility
- [ ] Named export (not default)

### New Feature

- [ ] `features/{feature-name}/` with components/, hooks/, types/
- [ ] Route in `app/{feature-name}/page.tsx`
- [ ] Server Actions in `actions/{feature-name}-actions.ts`
- [ ] Suspense boundaries for streaming
- [ ] ShadCN wrappers for domain UI
- [ ] Export public API from `index.ts`

---

## Common Imports

```typescript
// React & Next.js
import { Suspense } from 'react';
import { redirect, notFound } from 'next/navigation';
import { revalidatePath, revalidateTag } from 'next/cache';

// Client Hooks
import { useState, useTransition } from 'react';
import { useRouter } from 'next/navigation';

// ShadCN UI
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Card, CardContent } from '@/components/ui/card';
import { Dialog, DialogContent, DialogTrigger } from '@/components/ui/dialog';

// Server Actions & Auth
import { createOrder } from '@/actions/order-actions';
import { auth } from '@/lib/auth';
import { useSession } from 'next-auth/react';

// Utilities
import { cn } from '@/lib/utils';
import { toast } from 'sonner';
import { z } from 'zod';
```

---

## Topic Guides

### 🎨 [Component Patterns](resources/component-patterns.md)

Server Components by default, Client Components for interactivity. ShadCN wrapper pattern for domain logic. Hybrid location strategy.

**Key Topics:** Server/Client boundary, 'use client' rules, ShadCN wrappers, Container/Presentation pattern

---

### 📊 [Data Fetching](resources/data-fetching.md)

async Server Components fetch with native fetch API. Server Actions for mutations with Zod validation.

**Key Topics:** Server Components, Server Actions, revalidation, Client hooks for search/filter

---

### 📁 [File Organization](resources/file-organization.md)

App Router structure: app/, actions/, features/, components/. Special files: page.tsx, layout.tsx, loading.tsx, error.tsx.

**Key Topics:** Next.js structure, Route groups, actions/ directory, features/ organization, 3+ rule

---

### 🎨 [Styling Guide](resources/styling-guide.md)

Tailwind CSS with ShadCN components. cn() utility for conditional classes. CSS variables for theming.

**Key Topics:** Tailwind patterns, ShadCN wrappers, Composition over configuration

---

### 🛣️ [Routing Guide](resources/routing-guide.md)

Folder-based routing with app/. Dynamic routes with [slug]. Route groups with (name).

**Key Topics:** Special files, Dynamic routes, Parallel routes, Route groups

---

### ⏳ [Loading & Error States](resources/loading-and-error-states.md)

**CRITICAL:** No early returns. Use Suspense boundaries, loading.tsx, and error.tsx.

**Key Topics:** Suspense pattern, error.tsx, Skeleton components, Prevent CLS

---

### ⚡ [Performance](resources/performance.md)

Server Components (zero JS). Static rendering. Streaming with Suspense. Image/Font optimization.

**Key Topics:** Server Components, Streaming, useMemo/useCallback, Debouncing

---

### 📘 [TypeScript Standards](resources/typescript-standards.md)

Strict mode, no `any`. No React.FC. Server Action typing with FormData.

**Key Topics:** Type safety, Server Action types, Validation schemas

---

### 🔧 [Common Patterns](resources/common-patterns.md)

Auth (auth()/useSession), Forms with Server Actions, ShadCN Dialogs, Search hooks, Mutations with revalidation.

**Key Topics:** Authentication, Forms, Dialogs, Search/filter, Mutations, Toasts

---

### 📚 [Complete Examples](resources/complete-examples.md)

Full working examples: Server Component pages, feature structures, Server Actions, Client Components, forms, parallel fetching.

**Key Topics:** Complete Orders feature, Server Actions CRUD, Search hooks, ShadCN wrappers

---

## Core Principles

1. **Server Components by Default** - 'use client' only when needed
2. **Server Actions for Mutations** - No API routes in frontend
3. **Suspense for Loading** - Not early returns (prevents CLS)
4. **ShadCN UI** - Wrappers for domain logic
5. **Tailwind Styling** - cn() utility for conditional classes
6. **Hybrid Location** - features/ → components/ at 3+ uses
7. **Streaming** - Progressive loading with Suspense

---

## File Structure Reference

```
app/
  orders/
    page.tsx              # Route - Server Component
    [orderId]/page.tsx    # Dynamic route
    loading.tsx           # Loading UI
    error.tsx             # Error boundary

actions/
  order-actions.ts        # Server Actions

features/
  orders/
    components/
      order-list.tsx      # Client (interactive)
      order-detail.tsx    # Server Component
    hooks/
      use-search-orders.ts
    types/
      index.ts
    index.ts              # Public exports

components/
  ui/                     # ShadCN (auto-generated)
  domain/                 # Wrappers (3+ uses)
```

---

## Quick Templates

**Server Component:**
```typescript
// app/orders/[orderId]/page.tsx
interface PageProps {
    params: { orderId: string };
}

async function getOrder(orderId: string) {
    const res = await fetch(`${process.env.API_URL}/orders/${orderId}`, {
        next: { revalidate: 60 },
    });
    if (!res.ok) throw new Error('Failed to fetch');
    return res.json();
}

export default async function OrderPage({ params }: PageProps) {
    const order = await getOrder(params.orderId);
    return <OrderDetail order={order} />;
}
```

**Client Component:**
```typescript
// features/orders/components/order-list.tsx
'use client';

import { useState } from 'react';
import type { Order } from '../types';

interface OrderListProps {
    orders: Order[];
}

export function OrderList({ orders }: OrderListProps) {
    const [search, setSearch] = useState('');
    // Handle user interaction
    return <div>{/* UI */}</div>;
}
```

**Server Action:**
```typescript
// actions/order-actions.ts
'use server';

import { z } from 'zod';
import { revalidatePath } from 'next/cache';

const schema = z.object({
    name: z.string().min(1),
});

export async function createOrder(formData: FormData) {
    const parsed = schema.safeParse({
        name: formData.get('name'),
    });

    if (!parsed.success) {
        return { success: false, errors: parsed.error.flatten().fieldErrors };
    }

    // Call API
    revalidatePath('/orders');
    return { success: true };
}
```

---

## Navigation

| Need to... | Read |
|------------|------|
| Create component | [component-patterns.md](resources/component-patterns.md) |
| Fetch data | [data-fetching.md](resources/data-fetching.md) |
| Organize files | [file-organization.md](resources/file-organization.md) |
| Style components | [styling-guide.md](resources/styling-guide.md) |
| Set up routes | [routing-guide.md](resources/routing-guide.md) |
| Handle loading/errors | [loading-and-error-states.md](resources/loading-and-error-states.md) |
| Optimize performance | [performance.md](resources/performance.md) |
| TypeScript types | [typescript-standards.md](resources/typescript-standards.md) |
| Forms/Auth/Search | [common-patterns.md](resources/common-patterns.md) |
| See examples | [complete-examples.md](resources/complete-examples.md) |

---

**Skill Status**: Modular structure with progressive loading for optimal context management
