# Performance Optimization

Modern performance patterns for Next.js 15 App Router. Learn Server Component caching, static vs dynamic rendering, image optimization, and client component optimization.

---

## Server Component Caching

### Automatic Request Memoization

**Next.js automatically deduplicates requests** in Server Components:

```typescript
// app/dashboard/page.tsx
export default async function DashboardPage() {
    // All three components call getUser() with same ID
    // Next.js only makes ONE request - others are cached
    return (
        <>
            <UserProfile />   {/* getUser('123') */}
            <UserStats />     {/* getUser('123') - cached! */}
            <UserActivity />  {/* getUser('123') - cached! */}
        </>
    );
}

// actions/user.ts
export async function getUser(id: string) {
    // Called 3 times but executes once per request
    return await prisma.user.findUnique({ where: { id } });
}
```

**How it works:**
- Same fetch/function calls in single request tree are automatically memoized
- No configuration needed
- Resets per-request (not across requests)

---

## Static vs Dynamic Rendering

### Static Rendering (Default - Fastest)

**Pre-rendered at build time** - HTML cached on CDN:

```typescript
// app/about/page.tsx
export default async function AboutPage() {
    // No dynamic data - rendered once at build
    return (
        <div className="container py-8">
            <h1 className="text-3xl font-bold">About Us</h1>
            <p>Company information...</p>
        </div>
    );
}
```

**Benefits:**
- Instant page loads (served from CDN)
- No server computation per request
- Best Core Web Vitals scores
- Works offline (cached)

**Good for:**
- Marketing pages
- Blog posts
- Documentation
- Product pages with stable data

### Incremental Static Regeneration (ISR)

**Static with periodic updates:**

```typescript
// app/products/page.tsx
export const revalidate = 60; // Revalidate every 60 seconds

export default async function ProductsPage() {
    const products = await getProducts();

    return <ProductsList products={products} />;
}
```

**How it works:**
1. First request - serves static HTML
2. After 60 seconds - next request triggers rebuild in background
3. Subsequent requests get new version
4. Best of both worlds - fast + fresh data

### Dynamic Rendering

**Rendered per-request** - for user-specific content:

```typescript
// app/dashboard/page.tsx
import { cookies } from 'next/headers';

export default async function DashboardPage() {
    // Using cookies() forces dynamic rendering
    const userId = cookies().get('userId');

    const stats = await getStats(userId);

    return <DashboardDisplay stats={stats} />;
}
```

**Triggers dynamic rendering:**
- `cookies()`
- `headers()`
- `searchParams`
- `fetch()` with `cache: 'no-store'`

**Force dynamic:**
```typescript
export const dynamic = 'force-dynamic';
```

---

## Image Optimization

### next/image Component

**Automatic optimization** - use `<Image>` instead of `<img>`:

```typescript
import Image from 'next/image';

export function ProductCard({ product }: { product: Product }) {
    return (
        <div className="rounded-lg border bg-card p-4">
            {/* ✅ CORRECT - Optimized */}
            <Image
                src={product.imageUrl}
                alt={product.name}
                width={400}
                height={300}
                className="rounded-md"
            />

            {/* ❌ AVOID - Not optimized */}
            <img src={product.imageUrl} alt={product.name} />
        </div>
    );
}
```

**Benefits:**
- Automatic WebP/AVIF conversion
- Responsive images (srcset)
- Lazy loading by default
- Prevents layout shift (with width/height)
- Size optimization

### Priority Loading (Above Fold)

```typescript
<Image
    src="/hero.jpg"
    alt="Hero"
    width={1200}
    height={600}
    priority  // Load immediately (above the fold)
/>
```

### Fill Mode (Unknown Dimensions)

```typescript
<div className="relative h-96 w-full">
    <Image
        src={product.imageUrl}
        alt={product.name}
        fill
        className="object-cover"
    />
</div>
```

---

## Font Optimization

### next/font

**Automatic font optimization** - no layout shift:

```typescript
// app/layout.tsx
import { Inter, Roboto_Mono } from 'next/font/google';

const inter = Inter({
    subsets: ['latin'],
    display: 'swap',
});

const robotoMono = Roboto_Mono({
    subsets: ['latin'],
    weight: ['400', '700'],
    display: 'swap',
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
    return (
        <html lang="en" className={inter.className}>
            <body>{children}</body>
        </html>
    );
}
```

**Benefits:**
- Self-hosted (no external requests)
- Zero layout shift
- Preloaded automatically
- Subset optimization

---

## Code Splitting (Automatic)

### Route-Based Splitting

**Automatic** - Each route is a separate chunk:

```typescript
// app/orders/page.tsx
export default function OrdersPage() {
    // Only loads when user visits /orders
    return <OrdersList />;
}

// app/products/page.tsx
export default function ProductsPage() {
    // Only loads when user visits /products
    return <ProductsList />;
}
```

**No manual splitting needed** - Next.js does it automatically.

---

## Client Component Optimization

### useMemo for Expensive Computations

```typescript
'use client';

import { useMemo } from 'react';

export function OrdersTable({ orders }: { orders: Order[] }) {
    const [searchTerm, setSearchTerm] = useState('');
    const [sortBy, setSortBy] = useState('date');

    // ✅ CORRECT - Memoized expensive computation
    const filteredAndSortedOrders = useMemo(() => {
        return orders
            .filter(order =>
                order.customerName.toLowerCase().includes(searchTerm.toLowerCase())
            )
            .sort((a, b) => {
                if (sortBy === 'date') return b.createdAt - a.createdAt;
                if (sortBy === 'amount') return b.amount - a.amount;
                return 0;
            });
    }, [orders, searchTerm, sortBy]);

    return (
        <div>
            <input
                value={searchTerm}
                onChange={(e) => setSearchTerm(e.target.value)}
                placeholder="Search..."
            />
            <table>
                {filteredAndSortedOrders.map(order => (
                    <OrderRow key={order.id} order={order} />
                ))}
            </table>
        </div>
    );
}
```

**When to use useMemo:**
- Filtering/sorting large arrays
- Complex calculations
- Transforming data structures

**When NOT to:**
- Simple operations (string concat)
- Small arrays (<100 items)

### useCallback for Stable Functions

```typescript
'use client';

import { useCallback } from 'react';

export function OrdersList({ orders }: { orders: Order[] }) {
    const handleOrderClick = useCallback((orderId: string) => {
        console.log('Clicked:', orderId);
        // Navigate or show details
    }, []);

    return (
        <div>
            {orders.map(order => (
                <OrderCard
                    key={order.id}
                    order={order}
                    onClick={handleOrderClick}  // Stable reference
                />
            ))}
        </div>
    );
}
```

**When to use useCallback:**
- Functions passed to child components
- Functions in dependency arrays
- Event handlers in lists

---

## Streaming and Suspense

### Progressive Rendering

**Load fast content first, slow content streams in:**

```typescript
// app/dashboard/page.tsx
import { Suspense } from 'react';
import { Skeleton } from '@/components/ui/skeleton';

export default function DashboardPage() {
    return (
        <div className="container py-8 space-y-8">
            {/* Fast - loads immediately */}
            <h1 className="text-3xl font-bold">Dashboard</h1>

            {/* Fast query - appears first */}
            <Suspense fallback={<Skeleton className="h-32" />}>
                <QuickStats />
            </Suspense>

            {/* Slow query - doesn't block page */}
            <Suspense fallback={<Skeleton className="h-96" />}>
                <HeavyAnalytics />
            </Suspense>
        </div>
    );
}

async function QuickStats() {
    const stats = await getQuickStats(); // 50ms query
    return <StatsDisplay stats={stats} />;
}

async function HeavyAnalytics() {
    const analytics = await getAnalytics(); // 2000ms query - streams separately!
    return <AnalyticsChart data={analytics} />;
}
```

**Benefits:**
- User sees content sooner
- Slow queries don't block page
- Better perceived performance

---

## Parallel Data Fetching

### Promise.all() for Independent Data

```typescript
// app/orders/[id]/page.tsx
export default async function OrderDetailPage({ params }: { params: { id: string } }) {
    // ✅ CORRECT - Fetch in parallel
    const [order, customer, items] = await Promise.all([
        getOrder(params.id),      // 100ms
        getCustomer(params.id),   // 150ms
        getOrderItems(params.id), // 200ms
    ]);
    // Total time: 200ms (slowest query)

    return (
        <div>
            <OrderHeader order={order} />
            <CustomerInfo customer={customer} />
            <ItemsList items={items} />
        </div>
    );
}

// ❌ AVOID - Sequential (slow)
// const order = await getOrder(params.id);      // 100ms
// const customer = await getCustomer(params.id);   // 150ms
// const items = await getOrderItems(params.id); // 200ms
// Total time: 450ms (sum of all!)
```

---

## Debounced Search

```typescript
'use client';

import { useState } from 'react';
import { useDebounce } from '@/hooks/use-debounce';

export function OrdersSearch() {
    const [searchTerm, setSearchTerm] = useState('');
    const debouncedSearch = useDebounce(searchTerm, 300);

    // This only runs 300ms after user stops typing
    const results = useSearchOrders(debouncedSearch);

    return (
        <input
            value={searchTerm}
            onChange={(e) => setSearchTerm(e.target.value)}
            placeholder="Search orders..."
        />
    );
}
```

**Optimal debounce times:**
- Search: 300-500ms
- Auto-save: 1000ms
- Validation: 100-200ms

---

## Memory Leak Prevention

### Cleanup in useEffect

```typescript
'use client';

import { useEffect } from 'react';

export function MyComponent() {
    useEffect(() => {
        // Setup
        const interval = setInterval(() => {
            console.log('Tick');
        }, 1000);

        // ✅ CLEANUP - Prevent memory leak
        return () => {
            clearInterval(interval);
        };
    }, []);

    useEffect(() => {
        const handleResize = () => console.log('Resized');

        window.addEventListener('resize', handleResize);

        // ✅ CLEANUP
        return () => {
            window.removeEventListener('resize', handleResize);
        };
    }, []);

    return <div>Content</div>;
}
```

---

## Best Practices Summary

### DO:

**✅ Use Server Components by default**
```typescript
// Fastest - no client JS
export default async function Page() {
    const data = await getData();
    return <Display data={data} />;
}
```

**✅ Use next/image for images**
```typescript
<Image src="/photo.jpg" width={400} height={300} alt="Photo" />
```

**✅ Use ISR for semi-static pages**
```typescript
export const revalidate = 60; // Revalidate every 60s
```

**✅ Stream with Suspense**
```typescript
<Suspense fallback={<Skeleton />}>
    <SlowComponent />
</Suspense>
```

**✅ Parallel data fetching**
```typescript
const [data1, data2] = await Promise.all([getData1(), getData2()]);
```

### DON'T:

**❌ Don't fetch sequentially**
```typescript
// BAD - 450ms total
const data1 = await getData1(); // 150ms
const data2 = await getData2(); // 150ms
const data3 = await getData3(); // 150ms

// GOOD - 150ms total
const [data1, data2, data3] = await Promise.all([
    getData1(),
    getData2(),
    getData3(),
]);
```

**❌ Don't use regular <img>**
```typescript
// BAD
<img src="/photo.jpg" />

// GOOD
<Image src="/photo.jpg" width={400} height={300} alt="Photo" />
```

**❌ Don't make everything dynamic**
```typescript
// BAD - slower
export const dynamic = 'force-dynamic';

// GOOD - use ISR when possible
export const revalidate = 60;
```

---

## Summary

**Next.js 15 Performance Checklist:**

1. **Server Components** - Default to server, use client sparingly
2. **Static Rendering** - Use ISR for semi-static pages
3. **Image Optimization** - Always use next/image
4. **Font Optimization** - Use next/font
5. **Streaming** - Suspense for progressive rendering
6. **Parallel Fetching** - Promise.all() for independent data
7. **Code Splitting** - Automatic per-route
8. **Memoization** - useMemo/useCallback in Client Components
9. **Debouncing** - 300-500ms for search
10. **Cleanup** - Always cleanup effects

**Performance Hierarchy (fastest → slowest):**
1. Static (build time) - CDN cached
2. ISR (revalidate: 60) - Mostly static
3. Server Component (dynamic) - Server render per-request
4. Client Component - Browser render + JavaScript

**See Also:**
- [server-components.md](server-components.md) - Server Component patterns
- [data-fetching.md](data-fetching.md) - Fetching strategies
- [loading-and-error-states.md](loading-and-error-states.md) - Streaming
- [complete-examples.md](complete-examples.md) - Real examples
