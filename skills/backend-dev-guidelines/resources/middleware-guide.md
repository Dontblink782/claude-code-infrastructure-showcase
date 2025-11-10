# Middleware Guide - Next.js + Supabase Auth

Complete guide to middleware in Next.js with Supabase authentication. Covers token refresh, route protection, authorization patterns, and context management.

## Table of Contents

- [Understanding Next.js Middleware](#understanding-nextjs-middleware)
- [Supabase Auth Middleware Pattern](#supabase-auth-middleware-pattern)
- [Route Protection in Middleware](#route-protection-in-middleware)
- [Authorization in Server Actions](#authorization-in-server-actions)
- [Audit Context with AsyncLocalStorage](#audit-context-with-asynclocalstorage)
- [What NOT to Do](#what-not-to-do)
- [Common Patterns](#common-patterns)

---

## Understanding Next.js Middleware

### What It Is

Next.js middleware is code that runs **before a request is completed**. It executes at the edge (close to users) with limited capabilities.

**Key characteristics:**
- Runs once per request
- Executes before routes are matched
- Runs in Edge Runtime (not Node.js)
- Limited to lightweight operations

### When It Runs

```
Request Flow:
1. next.config.js (headers, redirects)
2. → Middleware (you are here!)
3. → Filesystem routes (_next/static, public/)
4. → Dynamic routes (pages, API routes)
5. → Fallback (404)
```

### File Location

```
project-root/
├── middleware.ts          # ← Middleware file (root level)
├── app/
│   ├── page.tsx
│   └── dashboard/
└── utils/
    └── supabase/
        └── middleware.ts  # ← Helper function
```

### Basic Structure

```typescript
// middleware.ts (project root)
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  // Your logic here
  return NextResponse.next()
}

export const config = {
  matcher: [
    '/((?!_next/static|_next/image|favicon.ico).*)'
  ]
}
```

### Matcher Configuration

The `matcher` controls which paths trigger middleware.

**Common patterns:**
```typescript
// Match specific paths
export const config = {
  matcher: ['/dashboard/:path*', '/api/:path*']
}

// Match everything except static assets
export const config = {
  matcher: [
    '/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)'
  ]
}

// Multiple specific paths
export const config = {
  matcher: ['/about/:path*', '/dashboard/:path*', '/api/:path*']
}
```

### Edge Runtime Limitations

Middleware runs in **Edge Runtime**, not Node.js. This means:

**❌ Cannot use:**
- Prisma (requires Node.js runtime)
- Heavy npm packages (file system, native modules)
- Long-running operations (10-50ms timeout)
- Complex business logic

**✅ Can use:**
- Supabase client (edge-compatible)
- Cookie manipulation
- Header manipulation
- URL redirects/rewrites
- Lightweight auth checks

**Rule:** Middleware is for **routing decisions**, not business logic.

---

## Supabase Auth Middleware Pattern

The primary use of middleware in Next.js + Supabase is **auth token refresh**. This ensures users stay logged in as they navigate.

### The updateSession Pattern

**File:** `utils/supabase/middleware.ts`

```typescript
import { createServerClient } from '@supabase/ssr'
import { NextResponse, type NextRequest } from 'next/server'

export async function updateSession(request: NextRequest) {
  let supabaseResponse = NextResponse.next({
    request
  })

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll()
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value, options }) => {
            request.cookies.set(name, value)
          })
          supabaseResponse = NextResponse.next({
            request
          })
          cookiesToSet.forEach(({ name, value, options }) => {
            supabaseResponse.cookies.set(name, value, options)
          })
        }
      }
    }
  )

  // IMPORTANT: Avoid writing any logic between createServerClient and
  // supabase.auth.getUser(). A mistake here could make it difficult to debug
  // issues with users being randomly logged out.

  const {
    data: { user }
  } = await supabase.auth.getUser()

  // IMPORTANT: You *must* return the supabaseResponse object as it is.
  // If you're creating a new response object with NextResponse.redirect or
  // NextResponse.rewrite, copy over the cookies from the supabaseResponse.

  return supabaseResponse
}
```

**What this does:**
1. Extracts auth cookies from request
2. Creates Supabase client with cookie handlers
3. Calls `getUser()` - **this refreshes expired tokens**
4. Syncs refreshed tokens to response cookies
5. Returns response with updated cookies

### Using in middleware.ts

**File:** `middleware.ts` (project root)

```typescript
import { type NextRequest } from 'next/server'
import { updateSession } from '@/utils/supabase/middleware'

export async function middleware(request: NextRequest) {
  return await updateSession(request)
}

export const config = {
  matcher: [
    /*
     * Match all request paths except:
     * - _next/static (static files)
     * - _next/image (image optimization files)
     * - favicon.ico (favicon file)
     * - images and other media files
     */
    '/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)'
  ]
}
```

### Critical Rules

**🚨 Rule 1: No logic between createServerClient and getUser()**

```typescript
// ❌ BAD - Logic between client creation and getUser()
const supabase = createServerClient(...)
const someData = await fetchSomething()  // DON'T DO THIS
const user = await supabase.auth.getUser()

// ✅ GOOD - Direct call
const supabase = createServerClient(...)
const user = await supabase.auth.getUser()
```

**🚨 Rule 2: Must return response with cookies**

```typescript
// ❌ BAD - Creating new response loses cookies
return NextResponse.redirect(new URL('/dashboard'))

// ✅ GOOD - Copy cookies when redirecting
const redirectResponse = NextResponse.redirect(new URL('/dashboard'))
supabaseResponse.cookies.getAll().forEach(cookie => {
  redirectResponse.cookies.set(cookie)
})
return redirectResponse
```

### Why This Pattern?

**Token refresh happens automatically:**
- User's access token expires (default 1 hour)
- `getUser()` detects expiration
- Uses refresh token to get new access token
- Updates cookies with new tokens
- User stays logged in seamlessly

**Without middleware:**
- User would be logged out after 1 hour
- Would need to manually refresh on every request
- Poor UX

---

## Route Protection in Middleware

Middleware can redirect unauthenticated users to login. This improves UX but **is not a security boundary**.

### Basic Protection Pattern

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
          // ... cookie sync logic
        }
      }
    }
  )

  const {
    data: { user }
  } = await supabase.auth.getUser()

  // Redirect to login if not authenticated
  if (
    !user &&
    !request.nextUrl.pathname.startsWith('/login') &&
    !request.nextUrl.pathname.startsWith('/signup')
  ) {
    const url = request.nextUrl.clone()
    url.pathname = '/login'
    return NextResponse.redirect(url)
  }

  return supabaseResponse
}
```

### Protecting Specific Routes

```typescript
// middleware.ts
import { type NextRequest } from 'next/server'
import { updateSession } from '@/utils/supabase/middleware'

export async function middleware(request: NextRequest) {
  return await updateSession(request)
}

export const config = {
  matcher: [
    // Only run middleware on protected routes
    '/dashboard/:path*',
    '/api/:path*',
    '/settings/:path*'
  ]
}
```

### Public Routes with Protected Sections

```typescript
// utils/supabase/middleware.ts
export async function updateSession(request: NextRequest) {
  // ... setup code

  const { data: { user } } = await supabase.auth.getUser()

  // Define protected paths
  const protectedPaths = ['/dashboard', '/settings', '/api']
  const isProtectedPath = protectedPaths.some(path =>
    request.nextUrl.pathname.startsWith(path)
  )

  // Redirect only if accessing protected path without auth
  if (!user && isProtectedPath) {
    const url = request.nextUrl.clone()
    url.pathname = '/login'
    url.searchParams.set('redirectTo', request.nextUrl.pathname)
    return NextResponse.redirect(url)
  }

  return supabaseResponse
}
```

### Redirect After Login

```typescript
// app/login/page.tsx
'use client'
import { useSearchParams } from 'next/navigation'

export default function LoginPage() {
  const searchParams = useSearchParams()
  const redirectTo = searchParams.get('redirectTo') || '/dashboard'

  async function handleLogin(formData: FormData) {
    // ... login logic
    window.location.href = redirectTo
  }

  return <form action={handleLogin}>...</form>
}
```

### Decision Tree: When to Protect Where

**Protect in Middleware when:**
- ✅ Want to redirect to login (better UX)
- ✅ Want to prevent page flicker (redirect before render)
- ✅ Checking only if user is logged in (simple check)

**Protect in Server Actions when:**
- ✅ Actual security enforcement (required!)
- ✅ Need to check permissions/roles
- ✅ Need to access database (Prisma)
- ✅ Complex authorization logic

**⚠️ Critical:** Middleware protection is UX, not security. **Always check auth in server actions too.**

---

## Authorization in Server Actions

Middleware redirects users to login. Server actions enforce actual authorization.

### The Two-Layer Pattern

```
Layer 1: Middleware (UX)
└─ Redirect if no auth token
   ↓
Layer 2: Server Action (Security)
└─ Verify user + check permissions
```

Both layers are required:
- **Middleware:** Fast redirect (UX)
- **Server Action:** Security boundary (required)

### getCurrentUser Helper

**File:** `utils/supabase/server.ts`

```typescript
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
          cookiesToSet.forEach(({ name, value, options }) => {
            cookieStore.set(name, value, options)
          })
        }
      }
    }
  )
}

export async function getCurrentUser() {
  const supabase = await createServerSupabaseClient()
  const {
    data: { user },
    error
  } = await supabase.auth.getUser()

  if (error || !user) {
    return null
  }

  return user
}
```

### Basic Authorization in Server Actions

```typescript
// actions/posts.ts
'use server'
import { getCurrentUser } from '@/utils/supabase/server'
import { prisma } from '@/lib/prisma'
import { revalidatePath } from 'next/cache'

export async function createPost(formData: FormData) {
  // 1. Auth check (required!)
  const user = await getCurrentUser()
  if (!user) {
    throw new Error('Unauthorized')
  }

  // 2. Validation
  const title = formData.get('title') as string
  const content = formData.get('content') as string

  if (!title || !content) {
    throw new Error('Title and content required')
  }

  // 3. Business logic with Prisma
  const post = await prisma.post.create({
    data: {
      title,
      content,
      userId: user.id
    }
  })

  // 4. Revalidate
  revalidatePath('/posts')

  return post
}
```

### Complex Authorization: Role Checks

```typescript
// actions/staff.ts
'use server'
import { getCurrentUser } from '@/utils/supabase/server'
import { prisma } from '@/lib/prisma'

export async function addStaffMember(
  organizationId: string,
  email: string
) {
  // 1. Auth check
  const user = await getCurrentUser()
  if (!user) throw new Error('Unauthorized')

  // 2. Permission check (requires Prisma)
  const membership = await prisma.organizationMember.findFirst({
    where: {
      userId: user.id,
      organizationId,
      role: 'ADMIN'
    }
  })

  if (!membership) {
    throw new Error('Only admins can add staff')
  }

  // 3. Business logic...
  return await prisma.organizationMember.create({
    data: {
      organizationId,
      email,
      role: 'MEMBER',
      invitedBy: user.id
    }
  })
}
```

### Resource Ownership Check

```typescript
// actions/posts.ts
'use server'
export async function deletePost(postId: string) {
  const user = await getCurrentUser()
  if (!user) throw new Error('Unauthorized')

  // Check ownership
  const post = await prisma.post.findUnique({
    where: { id: postId }
  })

  if (!post) {
    throw new Error('Post not found')
  }

  if (post.userId !== user.id) {
    throw new Error('You can only delete your own posts')
  }

  await prisma.post.delete({
    where: { id: postId }
  })

  revalidatePath('/posts')
}
```

### Why Not Just Middleware?

**❌ Middleware alone is not enough:**
- Middleware runs at edge (no Prisma access)
- Can't check database permissions/roles
- Easy to bypass (client could call API directly)
- Not a security boundary

**✅ Server actions are the security boundary:**
- Run on server with full Node.js access
- Can query database for permissions
- Can't be bypassed
- Real authorization enforcement

**Example of the gap:**
```typescript
// Middleware: "Is user logged in?"
const user = await supabase.auth.getUser()
if (!user) redirect('/login')  // ✅ Good UX

// Server Action: "Can user do THIS specific thing?"
const membership = await prisma.organizationMember.findFirst({
  where: { userId: user.id, organizationId, role: 'ADMIN' }
})
if (!membership) throw new Error('Forbidden')  // ✅ Real security
```

---

## Audit Context with AsyncLocalStorage

For complex apps, you may want to access the current user deep in your call stack without passing `userId` through every function.

### When to Use This

**✅ Use AsyncLocalStorage when:**
- Deep call stacks (action → service → repository)
- Audit logging needed in multiple places
- Want to avoid passing `userId` through 10+ functions
- Complex multi-layer architecture

**❌ Don't use when:**
- Simple app with 1-2 layers
- Can easily pass `userId` as parameter
- Adds unnecessary complexity

**Most apps don't need this** - just pass `userId` directly.

### AsyncLocalStorage Pattern

**File:** `lib/audit-context.ts`

```typescript
import { AsyncLocalStorage } from 'async_hooks'

export interface AuditContext {
  userId: string
  userName?: string
  email?: string
  timestamp: Date
  requestId: string
}

export const auditContextStorage = new AsyncLocalStorage<AuditContext>()

// Getter for current context
export function getAuditContext(): AuditContext | null {
  return auditContextStorage.getStore() || null
}

// Require context (throw if missing)
export function requireAuditContext(): AuditContext {
  const context = auditContextStorage.getStore()
  if (!context) {
    throw new Error('No audit context available')
  }
  return context
}
```

### Setting Context in Server Actions

```typescript
// actions/posts.ts
'use server'
import { getCurrentUser } from '@/utils/supabase/server'
import { auditContextStorage } from '@/lib/audit-context'
import { createPost as createPostService } from '@/lib/services/posts'
import { v4 as uuidv4 } from 'uuid'

export async function createPost(formData: FormData) {
  const user = await getCurrentUser()
  if (!user) throw new Error('Unauthorized')

  // Set context for this request
  return auditContextStorage.run(
    {
      userId: user.id,
      userName: user.user_metadata?.name,
      email: user.email,
      timestamp: new Date(),
      requestId: uuidv4()
    },
    async () => {
      // Everything in here has access to context
      return await createPostService(formData)
    }
  )
}
```

### Accessing Context Deep in Call Stack

```typescript
// lib/services/posts.ts
import { getAuditContext } from '@/lib/audit-context'
import { prisma } from '@/lib/prisma'
import { logToAuditService } from '@/lib/audit-log'

export async function createPostService(formData: FormData) {
  // Access context without it being passed as parameter
  const context = getAuditContext()

  const post = await prisma.post.create({
    data: {
      title: formData.get('title') as string,
      content: formData.get('content') as string,
      userId: context?.userId!
    }
  })

  // Audit log automatically has context
  await logToAuditService({
    action: 'POST_CREATED',
    resourceId: post.id,
    // userId, requestId automatically from context
  })

  return post
}

// lib/audit-log.ts
import { requireAuditContext } from '@/lib/audit-context'
import { prisma } from '@/lib/prisma'

export async function logToAuditService(data: {
  action: string
  resourceId: string
}) {
  const context = requireAuditContext()

  return await prisma.auditLog.create({
    data: {
      action: data.action,
      resourceId: data.resourceId,
      userId: context.userId,
      userName: context.userName,
      timestamp: context.timestamp,
      requestId: context.requestId
    }
  })
}
```

### Benefits

**With AsyncLocalStorage:**
```typescript
// Action sets context once
export async function createPost(formData: FormData) {
  return auditContextStorage.run(context, () => {
    return createPostService(formData)
      └─> calls updateRelatedService()
          └─> calls logAudit()  // ✅ Has context!
  })
}
```

**Without AsyncLocalStorage:**
```typescript
// Must pass through every layer
export async function createPost(formData: FormData) {
  const userId = ...
  return createPostService(formData, userId)
    └─> calls updateRelatedService(data, userId)  // 😫 Pass through
        └─> calls logAudit(action, userId)        // 😫 Pass through
}
```

### Important Notes

**This is NOT middleware:**
- AsyncLocalStorage works in server actions
- Separate from Next.js middleware.ts
- Different concept than Express middleware
- Context scoped to single request

**Type safety:**
```typescript
// Unsafe - might be null
const context = getAuditContext()
console.log(context?.userId)

// Safe - throws if missing
const context = requireAuditContext()
console.log(context.userId)  // ✅ Never null
```

---

## What NOT to Do

### ❌ Don't Use Prisma in Middleware

```typescript
// middleware.ts

// ❌ BAD - Prisma doesn't work in edge runtime
export async function middleware(request: NextRequest) {
  const user = await prisma.user.findUnique(...)  // ERROR!
  return NextResponse.next()
}

// ✅ GOOD - Use Prisma in server actions
// actions/users.ts
'use server'
export async function getUser(id: string) {
  return await prisma.user.findUnique({ where: { id } })
}
```

### ❌ Don't Add Logic Between createServerClient and getUser

```typescript
// ❌ BAD - Can cause random logouts
const supabase = createServerClient(...)
const someData = await fetchSomething()
await doSomeOtherThing()
const user = await supabase.auth.getUser()

// ✅ GOOD - Immediate call
const supabase = createServerClient(...)
const user = await supabase.auth.getUser()
```

### ❌ Don't Create Multiple Middleware Files

```typescript
// ❌ BAD - Next.js only supports one middleware.ts
// middleware.ts
// auth-middleware.ts
// logging-middleware.ts

// ✅ GOOD - Single middleware.ts, helpers in utils/
// middleware.ts (one file)
import { updateSession } from '@/utils/supabase/middleware'
```

### ❌ Don't Use Middleware for Validation

```typescript
// middleware.ts

// ❌ BAD - Validation in middleware
export async function middleware(request: NextRequest) {
  const body = await request.json()
  if (!body.email.includes('@')) {
    return NextResponse.json({ error: 'Invalid email' }, { status: 400 })
  }
}

// ✅ GOOD - Validation in server action
// actions/users.ts
'use server'
import { z } from 'zod'

const userSchema = z.object({
  email: z.string().email()
})

export async function createUser(formData: FormData) {
  const validated = userSchema.parse({
    email: formData.get('email')
  })
  // ...
}
```

### ❌ Don't Rely Only on Middleware for Security

```typescript
// ❌ BAD - Only middleware protection
// middleware.ts
if (!user) redirect('/login')

// actions/posts.ts (no auth check!)
export async function deletePost(id: string) {
  await prisma.post.delete({ where: { id } })  // Anyone can call this!
}

// ✅ GOOD - Both layers
// middleware.ts
if (!user) redirect('/login')  // UX

// actions/posts.ts
export async function deletePost(id: string) {
  const user = await getCurrentUser()
  if (!user) throw new Error('Unauthorized')  // Security
  await prisma.post.delete({ where: { id } })
}
```

### ❌ Don't Forget to Copy Cookies When Redirecting

```typescript
// middleware.ts

// ❌ BAD - Loses auth cookies
const supabaseResponse = await updateSession(request)
return NextResponse.redirect(new URL('/dashboard'))

// ✅ GOOD - Preserve cookies
const supabaseResponse = await updateSession(request)
const redirectResponse = NextResponse.redirect(new URL('/dashboard'))
supabaseResponse.cookies.getAll().forEach(cookie => {
  redirectResponse.cookies.set(cookie.name, cookie.value, cookie)
})
return redirectResponse
```

---

## Common Patterns

### Pattern 1: Protecting Dashboard Routes

```typescript
// middleware.ts
import { updateSession } from '@/utils/supabase/middleware'

export async function middleware(request: NextRequest) {
  return await updateSession(request)
}

export const config = {
  matcher: [
    '/dashboard/:path*',
    '/settings/:path*',
    '/profile/:path*'
  ]
}

// utils/supabase/middleware.ts
export async function updateSession(request: NextRequest) {
  // ... setup

  const { data: { user } } = await supabase.auth.getUser()

  if (!user && !request.nextUrl.pathname.startsWith('/login')) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  return supabaseResponse
}
```

### Pattern 2: Protecting API Routes

```typescript
// middleware.ts
export const config = {
  matcher: [
    '/api/:path*',  // Protect all API routes
    '/dashboard/:path*'
  ]
}

// app/api/posts/route.ts
import { getCurrentUser } from '@/utils/supabase/server'

export async function POST(request: Request) {
  const user = await getCurrentUser()
  if (!user) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 })
  }

  // ... handle request
}
```

### Pattern 3: Public Routes with Auth State

```typescript
// middleware.ts - runs on all routes
export const config = {
  matcher: [
    '/((?!_next/static|_next/image|favicon.ico).*)'
  ]
}

// utils/supabase/middleware.ts
export async function updateSession(request: NextRequest) {
  // ... setup

  const { data: { user } } = await supabase.auth.getUser()

  // No redirect - just refresh token
  // Routes can check user state themselves
  return supabaseResponse
}

// app/page.tsx (public page that shows user if logged in)
import { getCurrentUser } from '@/utils/supabase/server'

export default async function HomePage() {
  const user = await getCurrentUser()

  return (
    <div>
      <h1>Welcome</h1>
      {user ? (
        <p>Hello {user.email}</p>
      ) : (
        <a href="/login">Login</a>
      )}
    </div>
  )
}
```

### Pattern 4: Multi-Tenant Authorization

```typescript
// actions/organizations.ts
'use server'
import { getCurrentUser } from '@/utils/supabase/server'
import { prisma } from '@/lib/prisma'

export async function getOrganizationData(organizationId: string) {
  const user = await getCurrentUser()
  if (!user) throw new Error('Unauthorized')

  // Check user has access to this organization
  const membership = await prisma.organizationMember.findFirst({
    where: {
      userId: user.id,
      organizationId,
      status: 'ACTIVE'
    }
  })

  if (!membership) {
    throw new Error('No access to this organization')
  }

  // User has access - fetch data
  return await prisma.organization.findUnique({
    where: { id: organizationId },
    include: {
      members: true,
      projects: true
    }
  })
}
```

### Pattern 5: Role-Based Access Control

```typescript
// lib/auth-helpers.ts
import { getCurrentUser } from '@/utils/supabase/server'
import { prisma } from '@/lib/prisma'

export async function requireRole(
  organizationId: string,
  allowedRoles: string[]
) {
  const user = await getCurrentUser()
  if (!user) throw new Error('Unauthorized')

  const membership = await prisma.organizationMember.findFirst({
    where: {
      userId: user.id,
      organizationId,
      role: { in: allowedRoles }
    }
  })

  if (!membership) {
    throw new Error(`Must be ${allowedRoles.join(' or ')}`)
  }

  return { user, membership }
}

// actions/staff.ts
'use server'
import { requireRole } from '@/lib/auth-helpers'

export async function deleteStaffMember(
  organizationId: string,
  memberId: string
) {
  // Only admins can delete staff
  await requireRole(organizationId, ['ADMIN'])

  return await prisma.organizationMember.delete({
    where: { id: memberId }
  })
}
```

### Pattern 6: Redirect After Login

```typescript
// middleware.ts
export async function updateSession(request: NextRequest) {
  // ... setup

  const { data: { user } } = await supabase.auth.getUser()

  // Redirect to login with return URL
  if (!user && isProtectedPath(request.nextUrl.pathname)) {
    const url = request.nextUrl.clone()
    url.pathname = '/login'
    url.searchParams.set('redirectTo', request.nextUrl.pathname)
    return NextResponse.redirect(url)
  }

  return supabaseResponse
}

// app/login/actions.ts
'use server'
import { redirect } from 'next/navigation'
import { createServerSupabaseClient } from '@/utils/supabase/server'

export async function login(formData: FormData) {
  const supabase = await createServerSupabaseClient()
  const redirectTo = formData.get('redirectTo') as string || '/dashboard'

  const { error } = await supabase.auth.signInWithPassword({
    email: formData.get('email') as string,
    password: formData.get('password') as string
  })

  if (error) throw error

  redirect(redirectTo)
}
```

---

**Related Files:**
- [architecture-overview.md](architecture-overview.md) - Two-path architecture (RLS vs Server Actions)
- [database-patterns.md](database-patterns.md) - Prisma usage in server actions
- [validation-patterns.md](validation-patterns.md) - Zod validation in server actions
- [async-and-errors.md](async-and-errors.md) - Error handling patterns
