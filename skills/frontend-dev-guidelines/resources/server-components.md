# Server Components

Deep dive into Next.js 15 Server Components - the default rendering paradigm for App Router. Learn when to use them, data fetching patterns, streaming, and optimization strategies.

---

## What Are Server Components?

**Server Components** are React components that render exclusively on the server. They are the **default** in Next.js 15 App Router - every component is a Server Component unless you add `'use client'`.

**Key Characteristics:**
- Render on server only (never sent to client)
- Can be `async` functions
- Direct database/API access
- Zero JavaScript sent to browser
- Cannot use hooks (useState, useEffect)
- Cannot use browser APIs
- Automatic code splitting

**Mental Model:**
```
Traditional React (Client Components):
  Component JS → Browser → Render → Interactive

Server Components:
  Component → Server Render → HTML → Browser (no component JS!)
```

---

## Why Server Components?

### Performance Benefits

**Smaller bundle sizes:**
```typescript
// ❌ Client Component - sends Chart.js (100KB) to browser
'use client';
import Chart from 'chart.js';

export function SalesChart() {
    return <Chart data={...} />;
}

// ✅ Server Component - Chart.js stays on server
import Chart from 'chart.js';

export async function SalesChart() {
    const data = await getChartData();
    return <Chart data={data} />; // Renders to HTML on server
}
```

**Faster data fetching:**
```typescript
// ❌ Client Component - waterfall (HTML → JS → fetch)
'use client';
useEffect(() => {
    fetch('/api/orders').then(setOrders);
}, []);

// ✅ Server Component - parallel with page load
export async function OrdersPage() {
    const orders = await prisma.order.findMany(); // Direct DB access
    return <OrdersList orders={orders} />;
}
```

**Security:**
```typescript
// ✅ Server Component - secrets never exposed
export async function UserProfile() {
    const user = await fetch('https://api.example.com/user', {
        headers: {
            Authorization: `Bearer ${process.env.SECRET_API_KEY}`, // Safe!
        },
    });
    return <Profile user={user} />;
}
```

---

## When to Use Server Components

### Use Server Components For:

**✅ Data Fetching**
```typescript
export async function OrdersPage() {
    const orders = await prisma.order.findMany();
    return <OrdersList orders={orders} />;
}
```

**✅ Backend Resource Access**
```typescript
export async function ConfigPage() {
    const config = await fs.readFile('./config.json', 'utf-8');
    return <ConfigDisplay config={JSON.parse(config)} />;
}
```

**✅ Keeping Secrets Safe**
```typescript
export async function PaymentPage() {
    const stripeKey = process.env.STRIPE_SECRET_KEY; // Never exposed
    const paymentIntent = await stripe.paymentIntents.create({ /* ... */ });
    return <PaymentForm clientSecret={paymentIntent.client_secret} />;
}
```

**✅ Reducing Bundle Size**
```typescript
// Heavy libraries (markdown, syntax highlighting) stay on server
import { marked } from 'marked';
import { highlight } from 'highlight.js';

export async function BlogPost({ slug }: { slug: string }) {
    const post = await getPost(slug);
    const html = marked(post.content);
    return <div dangerouslySetInnerHTML={{ __html: html }} />;
}
```

**✅ Static/SEO Content**
```typescript
export async function ProductPage({ id }: { id: string }) {
    const product = await getProduct(id);
    return (
        <div>
            <h1>{product.name}</h1>
            <p>{product.description}</p>
            {/* Fully rendered HTML - great for SEO */}
        </div>
    );
}
```

---

## Data Fetching in Server Components

### Direct Database Access

```typescript
// app/orders/page.tsx
import { prisma } from '@/lib/db';

export default async function OrdersPage() {
    // Direct Prisma query - no API route needed
    const orders = await prisma.order.findMany({
        where: { status: 'active' },
        include: { customer: true },
        orderBy: { createdAt: 'desc' },
    });

    return (
        <div className="container">
            <h1>Active Orders</h1>
            <OrdersList orders={orders} />
        </div>
    );
}
```

### Using Server Actions

```typescript
// actions/orders.ts
'use server';

import { prisma } from '@/lib/db';

export async function getOrders() {
    return await prisma.order.findMany({
        where: { status: 'active' },
        include: { customer: true },
    });
}

// app/orders/page.tsx
import { getOrders } from '@/actions/orders';

export default async function OrdersPage() {
    const orders = await getOrders();
    return <OrdersList orders={orders} />;
}
```

**When to use each:**
- **Direct in component**: Simple queries, single use
- **Server Action**: Reusable logic, complex queries, multiple pages

---

## Parallel Data Fetching

### Promise.all() for Independent Data

```typescript
// app/dashboard/page.tsx
import { getStats } from '@/actions/stats';
import { getOrders } from '@/actions/orders';
import { getCustomers } from '@/actions/customers';

export default async function DashboardPage() {
    // Fetch all in parallel - total time = slowest query
    const [stats, recentOrders, topCustomers] = await Promise.all([
        getStats(),
        getOrders({ limit: 5 }),
        getCustomers({ limit: 5 }),
    ]);

    return (
        <div className="grid grid-cols-3 gap-4">
            <StatsCard data={stats} />
            <RecentOrders orders={recentOrders} />
            <TopCustomers customers={topCustomers} />
        </div>
    );
}
```

**Benefits:**
- All queries run concurrently
- Page renders when all data ready
- Type-safe with destructuring

---

## Streaming with Suspense

### Progressive Rendering

**Problem:** Slow queries block entire page

**Solution:** Stream components independently

```typescript
// app/dashboard/page.tsx
import { Suspense } from 'react';
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

// Each section is async Server Component
async function StatsSection() {
    const stats = await getStats(); // Fast query
    return <StatsCard data={stats} />;
}

async function OrdersSection() {
    const orders = await getOrders(); // Medium query
    return <OrdersList orders={orders} />;
}

async function CustomersSection() {
    const customers = await getCustomers(); // Slow query
    return <CustomersList customers={customers} />;
}
```

**How it works:**
1. Page shell renders immediately
2. Fast sections (stats) appear first
3. Slower sections (customers) stream in as ready
4. Better perceived performance

### Nested Suspense Boundaries

```typescript
export default function OrderPage({ params }: { params: { id: string } }) {
    return (
        <div className="space-y-8">
            {/* Order header loads first */}
            <Suspense fallback={<Skeleton className="h-24" />}>
                <OrderHeader orderId={params.id} />
            </Suspense>

            <div className="grid grid-cols-2 gap-4">
                {/* Items and timeline load independently */}
                <Suspense fallback={<Skeleton className="h-96" />}>
                    <OrderItems orderId={params.id} />
                </Suspense>

                <Suspense fallback={<Skeleton className="h-96" />}>
                    <OrderTimeline orderId={params.id} />
                </Suspense>
            </div>
        </div>
    );
}
```

---

## Error Handling

### error.tsx File Convention

**Next.js automatically wraps pages in error boundaries:**

```typescript
// app/orders/error.tsx
'use client'; // Error components must be Client Components

import { useEffect } from 'react';
import { Button } from '@/components/ui/button';

export default function OrdersError({
    error,
    reset,
}: {
    error: Error & { digest?: string };
    reset: () => void;
}) {
    useEffect(() => {
        console.error('Orders page error:', error);
    }, [error]);

    return (
        <div className="flex flex-col items-center justify-center min-h-screen">
            <h2 className="text-2xl font-bold mb-4">Something went wrong!</h2>
            <p className="text-muted-foreground mb-4">{error.message}</p>
            <Button onClick={reset}>Try again</Button>
        </div>
    );
}
```

### try/catch in Server Components

```typescript
export async function OrdersPage() {
    try {
        const orders = await prisma.order.findMany();
        return <OrdersList orders={orders} />;
    } catch (error) {
        console.error('Failed to fetch orders:', error);
        return (
            <div className="p-4 border border-red-500 rounded">
                <p>Failed to load orders. Please try again later.</p>
            </div>
        );
    }
}
```

### notFound() for 404s

```typescript
import { notFound } from 'next/navigation';

export async function OrderPage({ params }: { params: { id: string } }) {
    const order = await prisma.order.findUnique({
        where: { id: params.id },
    });

    if (!order) {
        notFound(); // Triggers not-found.tsx
    }

    return <OrderDetails order={order} />;
}

// app/orders/[id]/not-found.tsx
export default function OrderNotFound() {
    return (
        <div className="flex flex-col items-center justify-center min-h-screen">
            <h2 className="text-2xl font-bold">Order Not Found</h2>
            <p className="text-muted-foreground">The order you're looking for doesn't exist.</p>
        </div>
    );
}
```

---

## Loading States

### loading.tsx File Convention

```typescript
// app/orders/loading.tsx
import { Skeleton } from '@/components/ui/skeleton';

export default function OrdersLoading() {
    return (
        <div className="container space-y-4">
            <Skeleton className="h-8 w-48" /> {/* Title */}
            <Skeleton className="h-96 w-full" /> {/* Table */}
        </div>
    );
}
```

**Automatic behavior:**
- Shown while page.tsx async function resolves
- Wrapped in Suspense boundary automatically
- No manual Suspense needed

---

## Static vs Dynamic Rendering

### Static Rendering (Default)

**Page pre-rendered at build time:**

```typescript
// app/about/page.tsx
export default async function AboutPage() {
    // No dynamic data - rendered once at build
    return (
        <div>
            <h1>About Us</h1>
            <p>We are a company...</p>
        </div>
    );
}
```

**Good for:**
- Marketing pages
- Blog posts
- Documentation
- Anything that doesn't change per-request

### Dynamic Rendering

**Page rendered on each request:**

```typescript
// app/dashboard/page.tsx
import { cookies } from 'next/headers';

export default async function DashboardPage() {
    // Using cookies forces dynamic rendering
    const userId = cookies().get('userId');

    const stats = await getStats(userId);

    return <StatsDisplay stats={stats} />;
}
```

**Triggers dynamic rendering:**
- `cookies()`
- `headers()`
- `searchParams`
- `fetch()` with `cache: 'no-store'`

### Force Dynamic

```typescript
// app/dashboard/page.tsx
export const dynamic = 'force-dynamic'; // Opt out of static

export default async function DashboardPage() {
    const data = await getRealTimeData();
    return <DataDisplay data={data} />;
}
```

### Revalidate (ISR)

```typescript
// app/products/page.tsx
export const revalidate = 60; // Revalidate every 60 seconds

export default async function ProductsPage() {
    const products = await getProducts();
    return <ProductsList products={products} />;
}
```

---

## Metadata and SEO

### Static Metadata

```typescript
// app/orders/page.tsx
import type { Metadata } from 'next';

export const metadata: Metadata = {
    title: 'Orders',
    description: 'View and manage your orders',
};

export default async function OrdersPage() {
    const orders = await getOrders();
    return <OrdersList orders={orders} />;
}
```

### Dynamic Metadata

```typescript
// app/orders/[id]/page.tsx
import type { Metadata } from 'next';

export async function generateMetadata({
    params,
}: {
    params: { id: string };
}): Promise<Metadata> {
    const order = await getOrder(params.id);

    return {
        title: `Order ${order.orderNumber}`,
        description: `Order details for ${order.customerName}`,
    };
}

export default async function OrderPage({ params }: { params: { id: string } }) {
    const order = await getOrder(params.id);
    return <OrderDetails order={order} />;
}
```

---

## Server/Client Composition Patterns

### Server Wraps Client

```typescript
// app/dashboard/page.tsx (Server Component)
import { getStats } from '@/actions/stats';
import { StatsChart } from '@/features/dashboard/components/StatsChart'; // Client

export default async function DashboardPage() {
    const stats = await getStats();

    return (
        <div>
            <h1>Dashboard</h1>
            {/* Server Component passes data to Client Component */}
            <StatsChart data={stats} />
        </div>
    );
}
```

### Client with Server Children (via children prop)

```typescript
// features/layout/Sidebar.tsx (Client Component)
'use client';

import { useState } from 'react';

export function Sidebar({ children }: { children: React.ReactNode }) {
    const [collapsed, setCollapsed] = useState(false);

    return (
        <div className={collapsed ? 'w-16' : 'w-64'}>
            <button onClick={() => setCollapsed(!collapsed)}>Toggle</button>
            {/* children can still be Server Components! */}
            {children}
        </div>
    );
}

// app/layout.tsx (Server Component)
import { Sidebar } from '@/features/layout/Sidebar';

export default function RootLayout({ children }: { children: React.ReactNode }) {
    return (
        <html>
            <body>
                <Sidebar>
                    {/* This is still a Server Component tree */}
                    {children}
                </Sidebar>
            </body>
        </html>
    );
}
```

### Passing Server Components as Props

```typescript
// Client Component accepts Server Component as prop
'use client';

export function TabLayout({
    content,
    sidebar,
}: {
    content: React.ReactNode;
    sidebar: React.ReactNode;
}) {
    const [activeTab, setActiveTab] = useState('content');

    return (
        <div>
            <Tabs value={activeTab} onValueChange={setActiveTab}>
                <TabsList>
                    <TabsTrigger value="content">Content</TabsTrigger>
                    <TabsTrigger value="sidebar">Sidebar</TabsTrigger>
                </TabsList>
            </Tabs>
            {activeTab === 'content' ? content : sidebar}
        </div>
    );
}

// Server Component usage
export default async function Page() {
    const content = await getContent();
    const sidebar = await getSidebar();

    return (
        <TabLayout
            content={<ContentSection data={content} />}
            sidebar={<SidebarSection data={sidebar} />}
        />
    );
}
```

---

## Common Patterns

### List Page Pattern

```typescript
// app/orders/page.tsx
export default async function OrdersPage({
    searchParams,
}: {
    searchParams: { status?: string; page?: string };
}) {
    const status = searchParams.status || 'all';
    const page = Number(searchParams.page) || 1;

    const orders = await prisma.order.findMany({
        where: status !== 'all' ? { status } : undefined,
        skip: (page - 1) * 20,
        take: 20,
    });

    const total = await prisma.order.count({
        where: status !== 'all' ? { status } : undefined,
    });

    return (
        <div>
            <OrdersFilter /> {/* Client Component for filters */}
            <OrdersList orders={orders} /> {/* Display list */}
            <Pagination page={page} total={total} /> {/* Client Component */}
        </div>
    );
}
```

### Detail Page Pattern

```typescript
// app/orders/[id]/page.tsx
import { notFound } from 'next/navigation';

export default async function OrderDetailPage({
    params,
}: {
    params: { id: string };
}) {
    const order = await prisma.order.findUnique({
        where: { id: params.id },
        include: { items: true, customer: true },
    });

    if (!order) {
        notFound();
    }

    return (
        <div className="space-y-6">
            <OrderHeader order={order} />
            <OrderItems items={order.items} />
            <OrderCustomer customer={order.customer} />
            <OrderActions orderId={order.id} /> {/* Client Component for buttons */}
        </div>
    );
}
```

---

## Summary

**Server Components Best Practices:**

1. **Default to Server Components** - Use Client only when needed
2. **Async for data fetching** - Direct DB access, no API routes
3. **Parallel fetching** - Promise.all() for independent data
4. **Streaming** - Suspense for progressive rendering
5. **Error boundaries** - error.tsx and try/catch
6. **Loading states** - loading.tsx or Suspense fallbacks
7. **SEO-friendly** - generateMetadata() for dynamic pages
8. **Composition** - Pass Server Components as children/props to Client

**Server Component Recipe:**

```typescript
// app/page.tsx
import { Suspense } from 'react';
import type { Metadata } from 'next';

export const metadata: Metadata = {
    title: 'Page Title',
};

export default async function Page() {
    // Parallel fetch
    const [data1, data2] = await Promise.all([
        getData1(),
        getData2(),
    ]);

    return (
        <div>
            <DataDisplay data={data1} />

            <Suspense fallback={<Skeleton />}>
                <SlowSection data={data2} />
            </Suspense>
        </div>
    );
}
```

**See Also:**
- [data-fetching.md](data-fetching.md) - Fetching strategies
- [component-patterns.md](component-patterns.md) - Client/Server boundaries
- [server-actions.md](server-actions.md) - Mutations
- [complete-examples.md](complete-examples.md) - Full examples
