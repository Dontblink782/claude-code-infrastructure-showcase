# Complete Examples - Next.js + Supabase + Prisma

Real-world examples showing the two-path architecture in action.

## Table of Contents

- [Example 1: Simple CRUD with RLS](#example-1-simple-crud-with-rls)
- [Example 2: Complex Transaction](#example-2-complex-transaction)
- [Example 3: Mixed Approach](#example-3-mixed-approach)
- [Example 4: Complete Auth Flow](#example-4-complete-auth-flow)
- [Example 5: Dashboard with Performance](#example-5-dashboard-with-performance)

---

## Example 1: Simple CRUD with RLS

**Scenario:** Blog post management where anyone can read, but only authenticated users can create/edit their own posts.

**Why RLS:** Simple CRUD, straightforward ownership rules, no complex business logic.

### Complete File Structure

```
app/
├── posts/
│   ├── page.tsx           # Server component - fetches posts
│   └── [id]/
│       └── page.tsx       # Individual post page
├── actions/
│   └── posts.ts           # Server actions for mutations
└── components/
    ├── PostsList.tsx      # Client component - displays posts
    └── CreatePostForm.tsx # Client component - create form

supabase/
└── migrations/
    └── 001_posts_rls.sql  # RLS policies
```

### 1. RLS Policies (SQL)

```sql
-- supabase/migrations/001_posts_rls.sql

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

### 2. Server Action (Data Fetching)

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

### 3. Server Component Page

```typescript
// app/posts/page.tsx
import { getPosts } from '@/actions/posts'
import { PostsList } from '@/components/PostsList'
import { CreatePostForm } from '@/components/CreatePostForm'

export default async function PostsPage() {
  const posts = await getPosts()

  return (
    <div>
      <h1>Blog Posts</h1>
      <CreatePostForm />
      <PostsList initialPosts={posts} />
    </div>
  )
}
```

### 4. Client Component (Form)

```typescript
// components/CreatePostForm.tsx
'use client'
import { createPost } from '@/actions/posts'
import { useState, useTransition } from 'react'

export function CreatePostForm() {
  const [error, setError] = useState<string | null>(null)
  const [isPending, startTransition] = useTransition()

  async function handleSubmit(formData: FormData) {
    setError(null)

    startTransition(async () => {
      try {
        await createPost(formData)
      } catch (err) {
        setError(err instanceof Error ? err.message : 'Failed to create post')
      }
    })
  }

  return (
    <form action={handleSubmit}>
      {error && <div className="error">{error}</div>}

      <input name="title" placeholder="Title" required />
      <textarea name="content" placeholder="Content" required />

      <button type="submit" disabled={isPending}>
        {isPending ? 'Creating...' : 'Create Post'}
      </button>
    </form>
  )
}
```

### Request Flow

```
1. User visits /posts
   ↓
2. PostsPage server component renders
   ↓
3. getPosts() server action called
   ↓
4. Supabase client queries posts table
   ↓
5. RLS "viewable by everyone" policy allows read
   ↓
6. Posts returned and rendered
   ↓
7. User submits form
   ↓
8. createPost() server action called
   ↓
9. RLS "users can create" checks auth.uid()
   ↓
10. If authenticated → INSERT succeeds
    If not → Error returned
```

### Key Takeaways

- **Use RLS when:** Simple CRUD, ownership-based auth, no complex logic
- **RLS runs at database level:** Fast, secure, automatic filtering
- **No Prisma needed:** Direct Supabase client calls
- **Real-time capable:** Can add subscriptions for live updates

---

## Example 2: Complex Transaction

**Scenario:** Multi-tenant SaaS where admins can add staff members to their organization with role checks, business rules, and audit logging.

**Why Server Actions:** Complex authorization, transactions, business rules, audit logs.

### Complete File Structure

```
app/
├── dashboard/
│   └── [orgId]/
│       └── team/
│           └── page.tsx       # Team management page
├── actions/
│   └── staff.ts               # Staff management logic
└── lib/
    └── prisma.ts              # Prisma client
```

### 1. Server Action with Transaction

```typescript
// actions/staff.ts
'use server'
import { getCurrentUser } from '@/utils/supabase/server'
import { prisma } from '@/lib/prisma'
import { revalidatePath } from 'next/cache'
import { z } from 'zod'

const addStaffSchema = z.object({
  email: z.string().email(),
  role: z.enum(['ADMIN', 'MANAGER', 'MEMBER']),
  organizationId: z.string().uuid()
})

export async function addStaffMember(formData: FormData) {
  // 1. Auth check
  const user = await getCurrentUser()
  if (!user) {
    return { success: false, error: 'Unauthorized' }
  }

  // 2. Validate input
  const result = addStaffSchema.safeParse({
    email: formData.get('email'),
    role: formData.get('role'),
    organizationId: formData.get('organizationId')
  })

  if (!result.success) {
    return { success: false, error: 'Invalid input' }
  }

  const { email, role, organizationId } = result.data

  try {
    // 3. Transaction: Multiple operations that must succeed together
    const member = await prisma.$transaction(async (tx) => {
      // Check admin permission
      const membership = await tx.organizationMember.findFirst({
        where: {
          userId: user.id,
          organizationId,
          role: 'ADMIN'
        }
      })

      if (!membership) {
        throw new Error('Only admins can add staff')
      }

      // Check organization limits
      const org = await tx.organization.findUnique({
        where: { id: organizationId },
        include: { _count: { select: { members: true } } }
      })

      if (!org) {
        throw new Error('Organization not found')
      }

      if (org._count.members >= org.maxMembers) {
        throw new Error('Staff limit reached. Upgrade your plan.')
      }

      // Check if user already exists
      let staffUser = await tx.user.findUnique({
        where: { email }
      })

      // Create user if doesn't exist
      if (!staffUser) {
        staffUser = await tx.user.create({
          data: { email }
        })
      }

      // Create organization membership
      const newMember = await tx.organizationMember.create({
        data: {
          userId: staffUser.id,
          organizationId,
          role,
          invitedBy: user.id
        }
      })

      // Audit log
      await tx.auditLog.create({
        data: {
          action: 'STAFF_ADDED',
          userId: user.id,
          organizationId,
          metadata: {
            staffUserId: staffUser.id,
            role
          }
        }
      })

      return newMember
    })

    // 4. Revalidate page
    revalidatePath(`/dashboard/${organizationId}/team`)

    return { success: true, member }
  } catch (error) {
    return {
      success: false,
      error: error instanceof Error ? error.message : 'Failed to add staff'
    }
  }
}
```

### 2. Server Component Page

```typescript
// app/dashboard/[orgId]/team/page.tsx
import { getCurrentUser } from '@/utils/supabase/server'
import { prisma } from '@/lib/prisma'
import { AddStaffForm } from '@/components/AddStaffForm'
import { StaffList } from '@/components/StaffList'
import { notFound } from 'next/navigation'

export default async function TeamPage({
  params
}: {
  params: { orgId: string }
}) {
  const user = await getCurrentUser()
  if (!user) notFound()

  // Check user has access to this org
  const membership = await prisma.organizationMember.findFirst({
    where: {
      userId: user.id,
      organizationId: params.orgId
    }
  })

  if (!membership) notFound()

  // Fetch team members
  const members = await prisma.organizationMember.findMany({
    where: { organizationId: params.orgId },
    include: {
      user: {
        select: { email: true }
      }
    }
  })

  const isAdmin = membership.role === 'ADMIN'

  return (
    <div>
      <h1>Team Members</h1>
      {isAdmin && <AddStaffForm organizationId={params.orgId} />}
      <StaffList members={members} isAdmin={isAdmin} />
    </div>
  )
}
```

### 3. Client Component (Form)

```typescript
// components/AddStaffForm.tsx
'use client'
import { addStaffMember } from '@/actions/staff'
import { useState } from 'react'

export function AddStaffForm({ organizationId }: { organizationId: string }) {
  const [error, setError] = useState<string | null>(null)
  const [loading, setLoading] = useState(false)

  async function handleSubmit(formData: FormData) {
    setError(null)
    setLoading(true)

    formData.append('organizationId', organizationId)

    const result = await addStaffMember(formData)

    if (!result.success) {
      setError(result.error)
    }

    setLoading(false)
  }

  return (
    <form action={handleSubmit}>
      {error && <div className="error">{error}</div>}

      <input name="email" type="email" placeholder="Email" required />

      <select name="role" required>
        <option value="MEMBER">Member</option>
        <option value="MANAGER">Manager</option>
        <option value="ADMIN">Admin</option>
      </select>

      <button type="submit" disabled={loading}>
        {loading ? 'Adding...' : 'Add Staff Member'}
      </button>
    </form>
  )
}
```

### Request Flow

```
1. Admin visits /dashboard/123/team
   ↓
2. TeamPage checks auth (getCurrentUser)
   ↓
3. Verifies org membership (Prisma)
   ↓
4. Fetches team members (Prisma)
   ↓
5. Renders form (if admin)
   ↓
6. Admin submits form
   ↓
7. addStaffMember server action:
   - Gets current user (auth layer)
   - Validates input (Zod)
   - Starts transaction:
     * Check admin role
     * Check org limits
     * Create/find user
     * Create membership
     * Log action
   - All or nothing!
   ↓
8. Returns result to form
```

### Key Takeaways

- **Use server actions when:** Complex authorization, transactions, business rules
- **Transactions ensure:** All operations succeed or all fail (atomic)
- **Auth in two layers:** Middleware (UX) + server action (security)
- **Return error objects:** Better UX than throwing errors

---

## Example 3: Mixed Approach

**Scenario:** E-commerce product catalog where browsing is public (RLS), but inventory management requires complex logic (server actions).

**Why Mixed:** Leverage RLS for fast reads, Prisma for complex writes.

### 1. RLS Policies (Reads)

```sql
-- Enable RLS
ALTER TABLE products ENABLE ROW LEVEL SECURITY;

-- Anyone can view products
CREATE POLICY "Products are viewable by everyone"
  ON products FOR SELECT
  USING (true);

-- No direct writes through Supabase client
-- All writes go through server actions
```

### 2. Server Action (Product Browsing - RLS)

```typescript
// actions/products.ts
'use server'
import { createServerClient } from '@/utils/supabase/server'

export async function getProducts(category?: string) {
  const supabase = await createServerClient()

  let query = supabase
    .from('products')
    .select('*')
    .eq('isActive', true)

  if (category) {
    query = query.eq('category', category)
  }

  const { data, error } = await query

  if (error) throw error
  return data
}
```

### 3. Server Action (Inventory Update - Prisma)

```typescript
// actions/inventory.ts
'use server'
import { getCurrentUser } from '@/utils/supabase/server'
import { prisma } from '@/lib/prisma'
import { revalidatePath } from 'next/cache'

export async function updateInventory(
  productId: string,
  quantity: number,
  reason: string
) {
  // Auth check
  const user = await getCurrentUser()
  if (!user) {
    return { success: false, error: 'Unauthorized' }
  }

  // Role check - only warehouse staff can update inventory
  const staff = await prisma.staff.findFirst({
    where: {
      userId: user.id,
      role: { in: ['ADMIN', 'WAREHOUSE'] }
    }
  })

  if (!staff) {
    return { success: false, error: 'Insufficient permissions' }
  }

  try {
    // Transaction: Update inventory + log change
    const result = await prisma.$transaction(async (tx) => {
      // Get current inventory
      const product = await tx.product.findUnique({
        where: { id: productId }
      })

      if (!product) {
        throw new Error('Product not found')
      }

      // Business rule: Prevent negative inventory
      const newQuantity = product.inventory + quantity
      if (newQuantity < 0) {
        throw new Error('Insufficient inventory')
      }

      // Update inventory
      const updated = await tx.product.update({
        where: { id: productId },
        data: { inventory: newQuantity }
      })

      // Log inventory change
      await tx.inventoryLog.create({
        data: {
          productId,
          quantity,
          previousQuantity: product.inventory,
          newQuantity,
          reason,
          userId: user.id
        }
      })

      return updated
    })

    revalidatePath('/admin/inventory')
    return { success: true, product: result }
  } catch (error) {
    return {
      success: false,
      error: error instanceof Error ? error.message : 'Failed to update'
    }
  }
}
```

### 4. Product Page (RLS Path)

```typescript
// app/products/page.tsx
import { getProducts } from '@/actions/products'
import { ProductGrid } from '@/components/ProductGrid'

export default async function ProductsPage({
  searchParams
}: {
  searchParams: { category?: string }
}) {
  // Fast RLS-powered read
  const products = await getProducts(searchParams.category)

  return (
    <div>
      <h1>Products</h1>
      <ProductGrid products={products} />
    </div>
  )
}
```

### Request Flow

```
READ PATH (RLS - Fast):
1. User visits /products
   ↓
2. getProducts() uses Supabase client
   ↓
3. RLS allows public SELECT
   ↓
4. Fast database-level filtering
   ↓
5. Products rendered

WRITE PATH (Server Action - Complex):
1. Warehouse staff updates inventory
   ↓
2. updateInventory() server action
   ↓
3. getCurrentUser() + role check (Prisma)
   ↓
4. Transaction:
   - Fetch current inventory
   - Validate business rules
   - Update quantity
   - Log change
   ↓
5. Return result
```

### Key Takeaways

- **Mix approaches:** RLS for reads, server actions for writes
- **Best of both worlds:** Speed + flexibility
- **Decision rule:** Simple reads → RLS, Complex writes → server actions
- **No trade-offs:** Use the right tool for each operation

---

## Example 4: Complete Auth Flow

**Scenario:** Protecting dashboard routes with middleware token refresh and server action auth enforcement.

### Complete File Structure

```
middleware.ts             # Root middleware
utils/supabase/
├── middleware.ts         # Token refresh helper
└── server.ts             # Server client + getCurrentUser
app/
├── dashboard/
│   ├── page.tsx          # Protected page
│   └── error.tsx         # Error boundary
└── login/
    └── page.tsx          # Login page
```

### 1. Middleware (Token Refresh)

```typescript
// middleware.ts (project root)
import { type NextRequest } from 'next/server'
import { updateSession } from '@/utils/supabase/middleware'

export async function middleware(request: NextRequest) {
  return await updateSession(request)
}

export const config = {
  matcher: [
    '/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)'
  ]
}
```

### 2. Update Session Helper

```typescript
// utils/supabase/middleware.ts
import { createServerClient } from '@supabase/ssr'
import { NextResponse, type NextRequest } from 'next/server'

export async function updateSession(request: NextRequest) {
  let supabaseResponse = NextResponse.next({ request })

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll()
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value }) =>
            request.cookies.set(name, value)
          )
          supabaseResponse = NextResponse.next({ request })
          cookiesToSet.forEach(({ name, value, options }) =>
            supabaseResponse.cookies.set(name, value, options)
          )
        }
      }
    }
  )

  // This refreshes expired tokens automatically
  const { data: { user } } = await supabase.auth.getUser()

  // Protect dashboard routes
  if (
    !user &&
    request.nextUrl.pathname.startsWith('/dashboard')
  ) {
    const url = request.nextUrl.clone()
    url.pathname = '/login'
    url.searchParams.set('redirectTo', request.nextUrl.pathname)
    return NextResponse.redirect(url)
  }

  return supabaseResponse
}
```

### 3. Server Client Helper

```typescript
// utils/supabase/server.ts
import { createServerClient } from '@supabase/ssr'
import { cookies } from 'next/headers'

export async function createServerSupabaseClient() {
  const cookieStore = await cookies()

  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll()
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value, options }) =>
            cookieStore.set(name, value, options)
          )
        }
      }
    }
  )
}

export async function getCurrentUser() {
  const supabase = await createServerSupabaseClient()
  const { data: { user }, error } = await supabase.auth.getUser()

  if (error || !user) return null
  return user
}
```

### 4. Protected Dashboard Page

```typescript
// app/dashboard/page.tsx
import { getCurrentUser } from '@/utils/supabase/server'
import { prisma } from '@/lib/prisma'
import { notFound } from 'next/navigation'

export default async function DashboardPage() {
  // Security boundary - always check in server components/actions
  const user = await getCurrentUser()
  if (!user) notFound()

  // Fetch user data
  const userData = await prisma.user.findUnique({
    where: { id: user.id },
    include: {
      organizations: true,
      posts: {
        take: 5,
        orderBy: { createdAt: 'desc' }
      }
    }
  })

  return (
    <div>
      <h1>Welcome, {user.email}</h1>
      <div>
        <h2>Your Organizations</h2>
        {userData?.organizations.map(org => (
          <div key={org.id}>{org.name}</div>
        ))}
      </div>
      <div>
        <h2>Recent Posts</h2>
        {userData?.posts.map(post => (
          <div key={post.id}>{post.title}</div>
        ))}
      </div>
    </div>
  )
}
```

### 5. Server Action with Auth

```typescript
// actions/dashboard.ts
'use server'
import { getCurrentUser } from '@/utils/supabase/server'
import { prisma } from '@/lib/prisma'

export async function updateProfile(formData: FormData) {
  // Always check auth in server actions
  const user = await getCurrentUser()
  if (!user) {
    return { success: false, error: 'Unauthorized' }
  }

  try {
    const updated = await prisma.user.update({
      where: { id: user.id },
      data: {
        name: formData.get('name') as string,
        bio: formData.get('bio') as string
      }
    })

    return { success: true, user: updated }
  } catch (error) {
    return { success: false, error: 'Failed to update profile' }
  }
}
```

### Request Flow

```
1. User visits /dashboard
   ↓
2. Middleware runs (Edge Runtime)
   - Creates Supabase client
   - Calls getUser() → refreshes token if expired
   - No user? → Redirect to /login
   - Has user? → Continue
   ↓
3. DashboardPage renders (Server Component)
   - Calls getCurrentUser() again (security boundary!)
   - No user? → 404
   - Has user? → Fetch data with Prisma
   ↓
4. User submits form
   ↓
5. updateProfile server action
   - getCurrentUser() once more (can't trust middleware!)
   - No user? → Error
   - Has user? → Update with Prisma
```

### Key Takeaways

- **Two-layer auth:** Middleware (UX) + server action (security)
- **Middleware redirects:** Better UX, prevents page flicker
- **Always check in actions:** Middleware can be bypassed
- **Token refresh automatic:** getUser() handles expired tokens

---

## Example 5: Dashboard with Performance

**Scenario:** Dashboard page that fetches multiple data sources efficiently with error handling.

### Complete File Structure

```
app/
├── dashboard/
│   ├── page.tsx          # Main dashboard
│   ├── error.tsx         # Error boundary
│   └── loading.tsx       # Loading state
└── components/
    ├── StatsSection.tsx  # Server component
    ├── PostsSection.tsx  # Server component
    └── EventsSection.tsx # Server component
```

### 1. Dashboard Page (Parallel Queries)

```typescript
// app/dashboard/page.tsx
import { getCurrentUser } from '@/utils/supabase/server'
import { Suspense } from 'react'
import { StatsSection } from '@/components/StatsSection'
import { PostsSection } from '@/components/PostsSection'
import { EventsSection } from '@/components/EventsSection'
import { LoadingSpinner } from '@/components/LoadingSpinner'
import { notFound } from 'next/navigation'

export default async function DashboardPage() {
  const user = await getCurrentUser()
  if (!user) notFound()

  return (
    <div>
      <h1>Dashboard</h1>

      {/* Each Suspense boundary streams independently */}
      <Suspense fallback={<LoadingSpinner />}>
        <StatsSection userId={user.id} />
      </Suspense>

      <Suspense fallback={<LoadingSpinner />}>
        <PostsSection userId={user.id} />
      </Suspense>

      <Suspense fallback={<LoadingSpinner />}>
        <EventsSection userId={user.id} />
      </Suspense>
    </div>
  )
}
```

### 2. Stats Section (Parallel Prisma Queries)

```typescript
// components/StatsSection.tsx
import { prisma } from '@/lib/prisma'

export async function StatsSection({ userId }: { userId: string }) {
  // Parallel queries - all run at once
  const [postCount, commentCount, likeCount, followers] = await Promise.all([
    prisma.post.count({ where: { userId } }),
    prisma.comment.count({ where: { userId } }),
    prisma.like.count({ where: { post: { userId } } }),
    prisma.follow.count({ where: { followingId: userId } })
  ])

  return (
    <div className="stats">
      <div className="stat">
        <h3>Posts</h3>
        <p>{postCount}</p>
      </div>
      <div className="stat">
        <h3>Comments</h3>
        <p>{commentCount}</p>
      </div>
      <div className="stat">
        <h3>Likes</h3>
        <p>{likeCount}</p>
      </div>
      <div className="stat">
        <h3>Followers</h3>
        <p>{followers}</p>
      </div>
    </div>
  )
}
```

### 3. Error Boundary

```typescript
// app/dashboard/error.tsx
'use client'
import { useEffect } from 'react'

export default function DashboardError({
  error,
  reset
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  useEffect(() => {
    console.error('Dashboard error:', error)
  }, [error])

  return (
    <div className="error-container">
      <h2>Something went wrong loading your dashboard</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  )
}
```

### 4. Server Action (With Error Handling)

```typescript
// actions/dashboard.ts
'use server'
import { getCurrentUser } from '@/utils/supabase/server'
import { prisma } from '@/lib/prisma'

export async function getDashboardData(userId: string) {
  const user = await getCurrentUser()
  if (!user) {
    return { success: false, error: 'Unauthorized' }
  }

  try {
    // Parallel queries for performance
    const [stats, recentPosts, upcomingEvents] = await Promise.all([
      prisma.user.findUnique({
        where: { id: userId },
        select: {
          _count: {
            select: {
              posts: true,
              comments: true,
              followers: true
            }
          }
        }
      }),
      prisma.post.findMany({
        where: { userId },
        take: 5,
        orderBy: { createdAt: 'desc' },
        include: {
          _count: {
            select: { comments: true, likes: true }
          }
        }
      }),
      prisma.event.findMany({
        where: {
          participants: {
            some: { userId }
          },
          startDate: {
            gte: new Date()
          }
        },
        take: 3,
        orderBy: { startDate: 'asc' }
      })
    ])

    return {
      success: true,
      data: {
        stats: stats?._count,
        recentPosts,
        upcomingEvents
      }
    }
  } catch (error) {
    console.error('Dashboard fetch error:', error)
    return {
      success: false,
      error: 'Failed to load dashboard data'
    }
  }
}
```

### Performance Comparison

```typescript
// ❌ BAD - Sequential (slow)
const posts = await prisma.post.count()      // Wait 100ms
const comments = await prisma.comment.count() // Wait 150ms
const likes = await prisma.like.count()       // Wait 80ms
// Total: 330ms

// ✅ GOOD - Parallel (fast)
const [posts, comments, likes] = await Promise.all([
  prisma.post.count(),
  prisma.comment.count(),
  prisma.like.count()
])
// Total: 150ms (slowest query)
```

### Request Flow

```
1. User visits /dashboard
   ↓
2. DashboardPage renders
   - Shows layout immediately
   - Each Suspense boundary shown with LoadingSpinner
   ↓
3. StatsSection starts fetching (parallel queries)
   ↓
4. PostsSection starts fetching (parallel)
   ↓
5. EventsSection starts fetching (parallel)
   ↓
6. Each section streams to client as it completes
   - First one done: 150ms → renders
   - Second one done: 200ms → renders
   - Third one done: 250ms → renders
   ↓
7. If any error → caught by error.tsx boundary
```

### Key Takeaways

- **Use Promise.all:** Run independent queries in parallel
- **Use Suspense:** Stream sections as they load (better UX)
- **Error boundaries:** Catch and handle errors gracefully
- **Return error objects:** In server actions for form handling
- **Performance matters:** 330ms → 150ms with one change

---

**Related Files:**
- [architecture-overview.md](architecture-overview.md) - Two-path architecture decision tree
- [async-and-errors.md](async-and-errors.md) - Error handling patterns
- [database-patterns.md](database-patterns.md) - Prisma optimization
- [middleware-guide.md](middleware-guide.md) - Auth middleware patterns
