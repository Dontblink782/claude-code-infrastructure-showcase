# Architecture Overview - Next.js + Supabase + Prisma

Complete guide to the two-path architecture pattern for Next.js applications with Supabase auth/RLS and Prisma for complex business logic.

## Table of Contents

- [Two-Path Architecture](#two-path-architecture)
- [Request Lifecycle](#request-lifecycle)
- [Directory Structure](#directory-structure)
- [Separation of Concerns](#separation-of-concerns)

---

## Two-Path Architecture

Next.js + Supabase + Prisma follows a **pragmatic two-path pattern** that balances simplicity with power.

### The Two Paths

```
PATH 1: Simple CRUD (Supabase RLS)
┌─────────────────────────────────────┐
│    Client Component/Page            │
│    - useEffect, event handlers      │
│    - Supabase browser client        │
└───────────────┬─────────────────────┘
                ↓
┌─────────────────────────────────────┐
│    Supabase RLS Policies            │
│    - Auth checks (user logged in?)  │
│    - Ownership checks (user's data?)│
│    - Public reads, auth writes      │
└───────────────┬─────────────────────┘
                ↓
┌─────────────────────────────────────┐
│         Database                    │
└─────────────────────────────────────┘

PATH 2: Complex Logic (Server Actions + Prisma)
┌─────────────────────────────────────┐
│    Client Component/Page            │
│    - Form actions, useTransition    │
│    - Calls server action            │
└───────────────┬─────────────────────┘
                ↓
┌─────────────────────────────────────┐
│    Server Action (actions/*.ts)     │
│    - Get auth context               │
│    - Input validation               │
│    - Call Prisma (or lib helpers)   │
│    - Business logic                 │
│    - Transactions                   │
└───────────────┬─────────────────────┘
                ↓
┌─────────────────────────────────────┐
│    Prisma Client                    │
│    - Complex queries                │
│    - Transactions                   │
│    - Business operations            │
└───────────────┬─────────────────────┘
                ↓
┌─────────────────────────────────────┐
│         Database                    │
└─────────────────────────────────────┘
```

### When to Use Each Path

**Use Supabase RLS (Path 1) when:**
- ✅ Simple CRUD operations (list posts, update profile)
- ✅ Authorization is straightforward (user owns resource)
- ✅ No complex business logic needed
- ✅ Real-time subscriptions needed

**Use Server Actions + Prisma (Path 2) when:**
- ✅ Complex business logic (multitenant staff management)
- ✅ Multiple related operations (transactions)
- ✅ Third-party integrations (Stripe, email)
- ✅ Complex authorization (team permissions, roles)
- ✅ Need server-side validation

### Why This Architecture?

**Simplicity:**
- No over-engineering
- No thin wrapper functions
- Direct: client → RLS or action → Prisma
- Minimal abstraction layers

**Developer Experience:**
- Solo/duo dev friendly
- Easy to reason about
- Fast iteration
- Less boilerplate

**Performance:**
- RLS policies run at database level (fast)
- Server actions only when needed
- No unnecessary network hops

**Maintainability:**
- Clear decision: simple → RLS, complex → action
- Easy to locate logic (actions/*.ts or RLS)
- No hunting through layers

---

## Request Lifecycle

### Flow 1: RLS Path (Simple Post Creation)

**Scenario:** User creates a blog post

```typescript
1. Client Component renders form
   ↓
2. User submits → event handler fires
   ↓
3. Client gets Supabase browser client
   const supabase = createBrowserClient()
   ↓
4. Client calls Supabase directly
   await supabase.from('posts').insert({ title, content })
   ↓
5. RLS Policy executes (database level)
   - Check: auth.uid() IS NOT NULL (user logged in?)
   - Check: auth.uid() = user_id (user owns this?)
   ↓
6. If policy passes → INSERT succeeds
   If policy fails → Error returned to client
   ↓
7. Client updates UI (optimistic or on success)
```

**Key points:**
- No server action needed
- Auth checked at database level (RLS)
- Direct client → database flow
- Fast, simple, secure

### Flow 2: Server Action Path (Complex Staff Management)

**Scenario:** Multitenant app - admin adds staff member

```typescript
1. Client Component renders form with server action
   <form action={addStaffMember}>
   ↓
2. User submits → Next.js invokes server action
   ↓
3. Server Action: actions/staff.ts
   export async function addStaffMember(formData: FormData) {
   ↓
4. Get auth context
   const user = await getCurrentUser()
   if (!user) throw new Error('Unauthorized')
   ↓
5. Extract and validate input
   const email = formData.get('email')
   const validated = addStaffSchema.parse({ email, ... })
   ↓
6. Complex business logic
   - Check if user is admin of organization
   - Check if staff limit reached
   - Check if email already exists
   ↓
7. Prisma transaction (multiple operations)
   await prisma.$transaction([
     prisma.user.create({ ... }),
     prisma.organizationMember.create({ ... }),
     prisma.auditLog.create({ ... })
   ])
   ↓
8. Additional actions
   - Send invitation email
   - Log to monitoring
   ↓
9. Return result to client (redirect or revalidatePath)
```

**Key points:**
- Server action for complex logic
- Auth via getCurrentUser() helper
- Prisma for transactions
- Can integrate third-party services
- Server-side validation

---

## Directory Structure

### Recommended Structure

```
project-root/
├── app/                           # Next.js App Router
│   ├── (auth)/                    # Route groups
│   │   ├── login/
│   │   └── signup/
│   ├── dashboard/
│   │   └── page.tsx
│   └── api/                       # API routes (if needed)
│       └── webhooks/
│           └── stripe/
│               └── route.ts
├── actions/                       # Server actions (flat)
│   ├── posts.ts                   # Post-related actions
│   ├── comments.ts                # Comment-related actions
│   ├── staff.ts                   # Staff management
│   └── billing.ts                 # Billing/subscription
├── lib/
│   ├── db/                        # Reusable Prisma helpers
│   │   ├── posts.ts               # Only if 2x+ reuse
│   │   └── transactions.ts        # Complex queries
│   ├── stripe/                    # Stripe helpers
│   │   ├── payments.ts            # createPaymentIntent, etc
│   │   └── connect.ts             # Stripe Connect helpers
│   ├── email/                     # Email helpers
│   │   └── resend.ts              # sendWelcome, sendInvite
│   └── validations/               # Zod schemas
│       ├── post.ts
│       └── user.ts
├── utils/supabase/                # Supabase clients (official)
│   ├── client.ts                  # Browser client
│   ├── server.ts                  # Server client
│   └── middleware.ts              # Token refresh
├── middleware.ts                  # Next.js middleware
├── components/                    # React components
└── types/                         # TypeScript types
    └── database.ts                # Prisma types
```

### Core Directories Explained

**`actions/` - Server Actions**
- Flat structure (no nested dirs unless massive app)
- One file per feature/domain
- Contains business logic + Prisma calls
- Examples: `actions/posts.ts`, `actions/stripe.ts`

**`lib/` - Reusable Utilities**
- **Only extract when used 2+ times**
- Avoid thin wrappers (no `lib/stripe/createIntent.ts` that just wraps Stripe)
- Good: `lib/stripe/payments.ts` with 5+ functions
- Good: `lib/db/transactions.ts` for complex reusable queries
- Bad: `lib/db/posts.ts` with single `getPosts()` wrapper

**`utils/supabase/` - Auth & Database Client**
- Official Supabase structure (don't change)
- `client.ts` - Client components
- `server.ts` - Server actions/components
- `middleware.ts` - Token refresh logic

**`middleware.ts` - Next.js Middleware**
- Auth token refresh (Supabase pattern)
- Route protection
- Redirect logic

**`lib/validations/` - Zod Schemas**
- Shared validation schemas
- Reusable across actions
- Type inference

### What NOT to Create

**❌ Thin Wrappers:**
```typescript
// BAD - pointless wrapper
export async function getPost(id: string) {
  return prisma.post.findUnique({ where: { id } })
}
```

**❌ Over-abstracted Layers:**
```typescript
// BAD - unnecessary repository layer
class PostRepository {
  async find(id: string) { return prisma.post.findUnique(...) }
}
class PostService {
  constructor(private repo: PostRepository) {}
  async get(id: string) { return this.repo.find(id) }
}
```

**✅ Direct Approach:**
```typescript
// GOOD - direct in server action
export async function getPost(id: string) {
  const user = await getCurrentUser()
  if (!user) throw new Error('Unauthorized')

  return prisma.post.findUnique({
    where: { id },
    include: { comments: true }
  })
}
```

### When to Extract to lib/

**Extract when:**
- Used in 2+ actions
- Complex logic (20+ lines)
- Needs testing isolation
- Third-party integration (Stripe, email)

**Keep in actions/ when:**
- Single use case
- Simple query (<10 lines)
- Action-specific logic

---

## Separation of Concerns

### Decision Tree: RLS vs Server Actions

**Ask these questions:**

1. **Is this simple CRUD with straightforward auth?**
   - User owns resource? → RLS
   - Public read? → RLS
   - Logged-in users only? → RLS

2. **Does it need complex business logic?**
   - Multiple related operations? → Server Action
   - Transactions? → Server Action
   - Third-party APIs? → Server Action

3. **Does it need real-time updates?**
   - Yes? → RLS (Supabase real-time)
   - No? → Either path works

### Example 1: Blog Post CRUD (RLS Path)

**RLS Policy (SQL):**
```sql
-- Enable RLS
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;

-- Anyone can read posts
CREATE POLICY "Posts are viewable by everyone"
  ON posts FOR SELECT
  USING (true);

-- Authenticated users can create posts
CREATE POLICY "Users can create posts"
  ON posts FOR INSERT
  WITH CHECK (auth.uid() IS NOT NULL);

-- Users can update their own posts
CREATE POLICY "Users can update own posts"
  ON posts FOR UPDATE
  USING (auth.uid() = user_id);

-- Users can delete their own posts
CREATE POLICY "Users can delete own posts"
  ON posts FOR DELETE
  USING (auth.uid() = user_id);
```

**1. Server Action (data fetching logic):**
```typescript
// actions/posts.ts
'use server'
import { createServerClient } from '@/utils/supabase/server'
import { revalidatePath } from 'next/cache'

export async function getPosts() {
  const supabase = await createServerClient()

  const { data: posts, error } = await supabase
    .from('posts')
    .select('*')
    .order('created_at', { ascending: false })

  if (error) throw error
  return posts
}

export async function createPost(formData: FormData) {
  const supabase = await createServerClient()

  const { data, error } = await supabase
    .from('posts')
    .insert({
      title: formData.get('title') as string,
      content: formData.get('content') as string
    })
    .select()
    .single()

  if (error) throw error

  revalidatePath('/posts')
  return data
}
```

**2. Server Component Page (calls server action):**
```typescript
// app/posts/page.tsx
import { getPosts } from '@/actions/posts'
import { PostsList } from '@/components/PostsList'
import { CreatePostForm } from '@/components/CreatePostForm'

export default async function PostsPage() {
  const posts = await getPosts()

  return (
    <div>
      <CreatePostForm />
      <PostsList initialPosts={posts} />
    </div>
  )
}
```

**3. Presentational component (uses server action):**
```typescript
// components/CreatePostForm.tsx
'use client'
import { createPost } from '@/actions/posts'
import { useTransition } from 'react'

export function CreatePostForm() {
  const [isPending, startTransition] = useTransition()

  async function handleSubmit(formData: FormData) {
    startTransition(async () => {
      await createPost(formData)
      // Server action handles revalidation
    })
  }

  return (
    <form action={handleSubmit}>
      <input name="title" required />
      <textarea name="content" required />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Creating...' : 'Create Post'}
      </button>
    </form>
  )
}
```

**2B. Parallel Data Fetching (Avoid Waterfalls):**

**❌ Bad - Sequential (slow):**
```typescript
// app/dashboard/page.tsx
export default async function DashboardPage() {
  const posts = await getPosts()      // Wait for posts
  const events = await getEvents()    // Then wait for events
  const comments = await getComments() // Then wait for comments

  // Total time = posts + events + comments (waterfall!)
  return <div>...</div>
}
```

**✅ Good - Parallel with Promise.all:**
```typescript
// app/dashboard/page.tsx
import { getPosts } from '@/actions/posts'
import { getEvents } from '@/actions/events'
import { getComments } from '@/actions/comments'
import { PostsList } from '@/components/PostsList'
import { EventsList } from '@/components/EventsList'
import { CommentsList } from '@/components/CommentsList'

export default async function DashboardPage() {
  // Fetch all in parallel - total time = slowest query
  const [posts, events, comments] = await Promise.all([
    getPosts(),
    getEvents(),
    getComments()
  ])

  return (
    <div>
      <PostsList posts={posts} />
      <EventsList events={events} />
      <CommentsList comments={comments} />
    </div>
  )
}
```

**✅ Better - Suspense boundaries for streaming:**
```typescript
// app/dashboard/page.tsx
import { Suspense } from 'react'
import { PostsSection } from '@/components/PostsSection'
import { EventsSection } from '@/components/EventsSection'
import { CommentsSection } from '@/components/CommentsSection'
import { LoadingSpinner } from '@/components/LoadingSpinner'

export default function DashboardPage() {
  return (
    <div>
      {/* Each section streams independently as data arrives */}
      <Suspense fallback={<LoadingSpinner />}>
        <PostsSection />
      </Suspense>

      <Suspense fallback={<LoadingSpinner />}>
        <EventsSection />
      </Suspense>

      <Suspense fallback={<LoadingSpinner />}>
        <CommentsSection />
      </Suspense>
    </div>
  )
}

// components/PostsSection.tsx (Server Component)
import { getPosts } from '@/actions/posts'
import { PostsList } from './PostsList'

export async function PostsSection() {
  const posts = await getPosts()
  return <PostsList posts={posts} />
}

// Same pattern for EventsSection, CommentsSection...
```

**When to use which:**

- **Promise.all**: Data needed at once, render together (dashboard stats)
- **Suspense boundaries**: Independent sections, stream as ready (better UX)
- **Sequential**: Only when data depends on previous result

**4. Hook for client-side mutations (alternative pattern):**
```typescript
// hooks/useCreatePost.ts
'use client'
import { createBrowserClient } from '@/utils/supabase/client'
import { useState } from 'react'

export function useCreatePost() {
  const [loading, setLoading] = useState(false)
  const supabase = createBrowserClient()

  async function createPost(title: string, content: string) {
    setLoading(true)
    try {
      // RLS handles auth check
      const { data, error } = await supabase
        .from('posts')
        .insert({ title, content })
        .select()
        .single()

      if (error) throw error
      return { data, error: null }
    } catch (error) {
      return { data: null, error }
    } finally {
      setLoading(false)
    }
  }

  return { createPost, loading }
}
```

**3. Presentational component:**
```typescript
// components/CreatePostForm.tsx
'use client'
import { useCreatePost } from '@/hooks/useCreatePost'
import { useRouter } from 'next/navigation'

export function CreatePostForm() {
  const { createPost, loading } = useCreatePost()
  const router = useRouter()

  async function handleSubmit(e: FormEvent) {
    e.preventDefault()
    const formData = new FormData(e.currentTarget)

    const { data, error } = await createPost(
      formData.get('title') as string,
      formData.get('content') as string
    )

    if (error) {
      // Handle error
      return
    }

    router.refresh() // Revalidate server component data
  }

  return (
    <form onSubmit={handleSubmit}>
      <input name="title" required />
      <textarea name="content" required />
      <button type="submit" disabled={loading}>
        {loading ? 'Creating...' : 'Create Post'}
      </button>
    </form>
  )
}
```

**Why RLS here?**
- Simple CRUD
- Straightforward auth (user owns post)
- No complex business logic
- Needs real-time (can subscribe to changes)

### Example 1B: Advanced RLS (Multi-tenant with Staff Access)

**Scenario:** Multi-tenant SaaS where staff members need access to merchant data. Read-only RLS policies with helper functions for complex authorization.

**RLS Policies with Helper Functions:**
```sql
-- ============================================
-- ENABLE RLS ON ALL TABLES
-- ============================================

ALTER TABLE public.merchants ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.staff ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.customers ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.orders ENABLE ROW LEVEL SECURITY;

-- ============================================
-- HELPER FUNCTIONS FOR RLS
-- ============================================

-- Check if user has active staff access to a merchant
CREATE OR REPLACE FUNCTION has_merchant_access(target_merchant_id UUID)
RETURNS BOOLEAN AS $$
BEGIN
    RETURN EXISTS (
        SELECT 1
        FROM public.staff
        WHERE staff.merchant_id = target_merchant_id
        AND staff.user_id = auth.uid()
        AND staff.status = 'active'
    );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Check if user has staff access with specific role(s)
CREATE OR REPLACE FUNCTION has_merchant_role(target_merchant_id UUID, required_roles staff_role[])
RETURNS BOOLEAN AS $$
BEGIN
    RETURN EXISTS (
        SELECT 1
        FROM public.staff
        WHERE staff.merchant_id = target_merchant_id
        AND staff.user_id = auth.uid()
        AND staff.status = 'active'
        AND staff.role = ANY(required_roles)
    );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- ============================================
-- MERCHANTS TABLE RLS POLICIES (READ-ONLY)
-- ============================================

-- Users can read merchants where they have active staff access
CREATE POLICY "Users can read their merchants"
    ON public.merchants
    FOR SELECT
    USING (
        has_merchant_access(id)
    );

-- ============================================
-- STAFF TABLE RLS POLICIES (READ-ONLY)
-- ============================================

-- Users can read their own staff record
CREATE POLICY "Users can read their own staff record"
    ON public.staff
    FOR SELECT
    USING (
        user_id = auth.uid()
    );

-- Admins and managers can read all staff in their merchant
CREATE POLICY "Admins and managers can read staff"
    ON public.staff
    FOR SELECT
    USING (
        has_merchant_role(merchant_id, ARRAY['admin', 'manager']::staff_role[])
    );

-- ============================================
-- CUSTOMERS TABLE RLS POLICIES (READ-ONLY)
-- ============================================

-- Users can read customers from merchants they have access to
CREATE POLICY "Users can read customers from their merchants"
    ON public.customers
    FOR SELECT
    USING (
        has_merchant_access(merchant_id)
    );

-- ============================================
-- ORDERS TABLE RLS POLICIES (READ-ONLY)
-- ============================================

-- Users can read orders from merchants they have access to
CREATE POLICY "Users can read orders from their merchants"
    ON public.orders
    FOR SELECT
    USING (
        has_merchant_access(merchant_id)
    );

-- ============================================
-- GRANT PERMISSIONS (READ-ONLY)
-- ============================================

-- Grant authenticated users SELECT access only (webhooks handle writes)
GRANT SELECT ON public.merchants TO authenticated;
GRANT SELECT ON public.staff TO authenticated;
GRANT SELECT ON public.customers TO authenticated;
GRANT SELECT ON public.orders TO authenticated;
```

**1. Server Action (data fetching logic):**
```typescript
// actions/orders.ts
'use server'
import { createServerClient } from '@/utils/supabase/server'

export async function getOrders() {
  const supabase = await createServerClient()

  // RLS automatically filters to orders from user's merchants
  const { data: orders, error } = await supabase
    .from('orders')
    .select('*, merchant:merchants(name), customer:customers(name)')
    .order('created_at', { ascending: false })
    .limit(50)

  if (error) throw error
  return orders
}
```

**2. Server Component Page (calls server action):**
```typescript
// app/orders/page.tsx
import { getOrders } from '@/actions/orders'
import { OrdersList } from '@/components/OrdersList'
import { OrdersSearch } from '@/components/OrdersSearch'

export default async function OrdersPage() {
  const orders = await getOrders()

  return (
    <div>
      <OrdersSearch />
      <OrdersList initialOrders={orders} />
    </div>
  )
}
```

**3. Hook for dynamic client-side queries:**
```typescript
// hooks/useSearchOrders.ts
'use client'
import { createBrowserClient } from '@/utils/supabase/client'
import { useState } from 'react'

export function useSearchOrders() {
  const [orders, setOrders] = useState([])
  const [loading, setLoading] = useState(false)
  const supabase = createBrowserClient()

  async function searchOrders(query: string) {
    setLoading(true)
    try {
      // RLS automatically filters to user's merchants
      const { data, error } = await supabase
        .from('orders')
        .select('*, merchant:merchants(name), customer:customers(name)')
        .ilike('customer.name', `%${query}%`)
        .order('created_at', { ascending: false })

      if (error) throw error
      setOrders(data)
    } finally {
      setLoading(false)
    }
  }

  return { orders, searchOrders, loading }
}
```

**3. Presentational components:**
```typescript
// components/OrdersList.tsx
'use client'

interface OrdersListProps {
  initialOrders: Order[]
}

export function OrdersList({ initialOrders }: OrdersListProps) {
  return (
    <div>
      {initialOrders.map(order => (
        <OrderCard key={order.id} order={order} />
      ))}
    </div>
  )
}

// components/OrdersSearch.tsx
'use client'
import { useSearchOrders } from '@/hooks/useSearchOrders'
import { useState } from 'react'

export function OrdersSearch() {
  const [query, setQuery] = useState('')
  const { orders, searchOrders, loading } = useSearchOrders()

  async function handleSearch(e: FormEvent) {
    e.preventDefault()
    await searchOrders(query)
  }

  return (
    <form onSubmit={handleSearch}>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search orders..."
      />
      <button type="submit" disabled={loading}>Search</button>

      {orders.length > 0 && (
        <div>
          <h3>Search Results:</h3>
          {orders.map(order => (
            <OrderCard key={order.id} order={order} />
          ))}
        </div>
      )}
    </form>
  )
}
```

**Why RLS here?**
- Read-only access (webhooks handle writes via server)
- Complex multi-tenant isolation (staff → merchant → data)
- Helper functions encapsulate authorization logic
- Role-based access (admin/manager can see all staff)
- Reusable across all tables (has_merchant_access)
- No N+1 queries - Postgres handles filtering efficiently

**Pattern:** Read via RLS, Write via Server Actions/Webhooks
- RLS policies provide secure, efficient reads
- Server actions/webhooks handle complex writes with business logic
- Best of both worlds: security + flexibility

### Example 2: Multitenant Staff (Server Action Path)

**Server Action:**
```typescript
'use server'
import { getCurrentUser } from '@/utils/supabase/server'
import { prisma } from '@/lib/prisma'
import { addStaffSchema } from '@/lib/validations/staff'
import { revalidatePath } from 'next/cache'

export async function addStaffMember(formData: FormData) {
  // 1. Auth
  const user = await getCurrentUser()
  if (!user) throw new Error('Unauthorized')

  // 2. Validate
  const validated = addStaffSchema.parse({
    email: formData.get('email'),
    role: formData.get('role'),
    organizationId: formData.get('organizationId')
  })

  // 3. Complex authorization
  const membership = await prisma.organizationMember.findFirst({
    where: {
      userId: user.id,
      organizationId: validated.organizationId,
      role: 'ADMIN'
    }
  })
  if (!membership) throw new Error('Only admins can add staff')

  // 4. Business rule check
  const org = await prisma.organization.findUnique({
    where: { id: validated.organizationId },
    include: { _count: { select: { members: true } } }
  })
  if (org._count.members >= org.maxMembers) {
    throw new Error('Staff limit reached')
  }

  // 5. Transaction (multiple operations)
  const result = await prisma.$transaction(async (tx) => {
    // Create user if doesn't exist
    let staffUser = await tx.user.findUnique({
      where: { email: validated.email }
    })

    if (!staffUser) {
      staffUser = await tx.user.create({
        data: { email: validated.email }
      })
    }

    // Add to organization
    const member = await tx.organizationMember.create({
      data: {
        userId: staffUser.id,
        organizationId: validated.organizationId,
        role: validated.role,
        invitedBy: user.id
      }
    })

    // Audit log
    await tx.auditLog.create({
      data: {
        action: 'STAFF_ADDED',
        userId: user.id,
        organizationId: validated.organizationId,
        metadata: { staffUserId: staffUser.id }
      }
    })

    return member
  })

  // 6. Side effects
  await sendInvitationEmail(validated.email, org.name)

  // 7. Revalidate
  revalidatePath(`/dashboard/${validated.organizationId}/team`)

  return { success: true, member: result }
}
```

**Why Server Action here?**
- Complex authorization (admin check, membership)
- Business rules (staff limit)
- Transaction (3 related operations)
- Side effects (email)
- Can't be done with RLS alone

### Key Principles

**RLS is for:**
- Row-level authorization (user owns row)
- Simple rules (`auth.uid() = user_id`)
- CRUD operations
- Real-time subscriptions

**Server Actions are for:**
- Complex authorization (roles, permissions)
- Business logic (limits, calculations)
- Multiple operations (transactions)
- Third-party integrations
- Server-side validation

**Both can coexist:**
- Use RLS for reads (fast, real-time)
- Use Server Actions for complex writes

---

**Related Files:**
- [SKILL.md](../SKILL.md) - Main guide
- [validation-patterns.md](validation-patterns.md) - Zod validation (still relevant)
- [database-patterns.md](database-patterns.md) - Prisma patterns (needs Next.js update)
- [async-and-errors.md](async-and-errors.md) - Error handling (still relevant)

**Note:** Other resource files still contain Express patterns and need updating for Next.js + Supabase stack.
