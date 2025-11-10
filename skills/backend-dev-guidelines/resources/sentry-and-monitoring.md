# Sentry Integration and Monitoring - Next.js

Complete guide to error tracking and performance monitoring with Sentry v8 in Next.js applications.

## Table of Contents

- [Core Principles](#core-principles)
- [Next.js Instrumentation](#nextjs-instrumentation)
- [Server Action Error Handling](#server-action-error-handling)
- [Error Boundaries](#error-boundaries)
- [API Route Error Handling](#api-route-error-handling)
- [Middleware Error Tracking](#middleware-error-tracking)
- [Performance Monitoring](#performance-monitoring)
- [Error Context Best Practices](#error-context-best-practices)
- [Common Mistakes](#common-mistakes)

---

## Core Principles

**MANDATORY**: All errors MUST be captured to Sentry. No exceptions.

### Key Rules

1. **Capture all errors** - Never silently fail
2. **Add context** - Tags, user info, breadcrumbs
3. **Protect PII** - Scrub sensitive data before sending
4. **Monitor performance** - Track slow operations
5. **Use scopes** - Isolate error context

### When to Capture Errors

```typescript
✅ Server action failures
✅ Database query errors
✅ Business logic failures
✅ Auth failures (except expected JWT expiry)
✅ External API failures
✅ Unexpected exceptions

❌ Expected validation errors (return to user instead)
❌ 404s for valid not-found cases
❌ Health check failures
```

---

## Next.js Instrumentation

### File Structure

```
project-root/
├── instrumentation.ts           # Sentry init (root level)
├── sentry.client.config.ts      # Client-side config
├── sentry.server.config.ts      # Server-side config
└── sentry.edge.config.ts        # Edge runtime config
```

### 1. Server Instrumentation

**File:** `instrumentation.ts` (project root)

```typescript
// instrumentation.ts
export async function register() {
  if (process.env.NEXT_RUNTIME === 'nodejs') {
    await import('./sentry.server.config')
  }

  if (process.env.NEXT_RUNTIME === 'edge') {
    await import('./sentry.edge.config')
  }
}
```

### 2. Server Config

**File:** `sentry.server.config.ts`

```typescript
import * as Sentry from '@sentry/nextjs'

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,

  // Performance monitoring
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
  profilesSampleRate: 0.1,

  integrations: [
    Sentry.extraErrorDataIntegration({ depth: 5 }),
    Sentry.prismaIntegration(),
    Sentry.httpIntegration(),
  ],

  beforeSend(event, hint) {
    // Filter health checks
    if (event.request?.url?.includes('/api/health')) {
      return null
    }

    // Scrub sensitive headers
    if (event.request?.headers) {
      delete event.request.headers['authorization']
      delete event.request.headers['cookie']
    }

    // Mask emails for PII protection
    if (event.user?.email) {
      event.user.email = event.user.email.replace(
        /^(.{2}).*(@.*)$/,
        '$1***$2'
      )
    }

    return event
  },

  ignoreErrors: [
    /^Invalid JWT/,
    /^JWT expired/,
    'NetworkError',
    'AbortError',
  ],
})

// Set service context
Sentry.setTags({
  service: 'app',
  runtime: 'nodejs',
})
```

### 3. Client Config

**File:** `sentry.client.config.ts`

```typescript
import * as Sentry from '@sentry/nextjs'

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  environment: process.env.NODE_ENV,

  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,

  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,

  integrations: [
    Sentry.replayIntegration({
      maskAllText: true,
      blockAllMedia: true,
    }),
  ],

  beforeSend(event, hint) {
    // Don't send CSP violations
    if (event.exception?.values?.[0]?.value?.includes('CSP')) {
      return null
    }

    return event
  },
})
```

### 4. Edge Config

**File:** `sentry.edge.config.ts`

```typescript
import * as Sentry from '@sentry/nextjs'

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 0.1,
})
```

---

## Server Action Error Handling

### Basic Pattern

```typescript
// actions/posts.ts
'use server'
import * as Sentry from '@sentry/nextjs'
import { getCurrentUser } from '@/utils/supabase/server'
import { prisma } from '@/lib/prisma'

export async function createPost(formData: FormData) {
  try {
    const user = await getCurrentUser()
    if (!user) {
      return { success: false, error: 'Unauthorized' }
    }

    const post = await prisma.post.create({
      data: {
        title: formData.get('title') as string,
        content: formData.get('content') as string,
        userId: user.id
      }
    })

    return { success: true, post }
  } catch (error) {
    // Capture to Sentry with context
    Sentry.captureException(error, {
      tags: {
        action: 'createPost',
        userId: user?.id
      },
      extra: {
        title: formData.get('title'),
        hasContent: !!formData.get('content')
      }
    })

    return {
      success: false,
      error: 'Failed to create post'
    }
  }
}
```

### Transaction Error Handling

```typescript
// actions/staff.ts
'use server'
import * as Sentry from '@sentry/nextjs'
import { getCurrentUser } from '@/utils/supabase/server'
import { prisma } from '@/lib/prisma'

export async function addStaffMember(
  organizationId: string,
  email: string
) {
  const user = await getCurrentUser()
  if (!user) {
    return { success: false, error: 'Unauthorized' }
  }

  return await Sentry.startSpan(
    {
      name: 'addStaffMember',
      op: 'db.transaction',
      attributes: {
        organizationId,
        inviterUserId: user.id
      }
    },
    async () => {
      try {
        const result = await prisma.$transaction(async (tx) => {
          // Business logic with multiple operations
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

          // Create member
          return await tx.organizationMember.create({
            data: {
              organizationId,
              email,
              invitedBy: user.id
            }
          })
        })

        return { success: true, member: result }
      } catch (error) {
        // Capture transaction failures
        Sentry.captureException(error, {
          tags: {
            action: 'addStaffMember',
            errorType: 'transaction_failed'
          },
          contexts: {
            organization: {
              id: organizationId
            },
            inviter: {
              id: user.id,
              email: user.email
            }
          }
        })

        return {
          success: false,
          error: error instanceof Error ? error.message : 'Failed to add staff'
        }
      }
    }
  )
}
```

### Prisma Error Handling

```typescript
// actions/users.ts
'use server'
import * as Sentry from '@sentry/nextjs'
import { Prisma } from '@prisma/client'
import { prisma } from '@/lib/prisma'

export async function createUser(email: string, name: string) {
  try {
    return await prisma.user.create({
      data: { email, name }
    })
  } catch (error) {
    // Handle Prisma-specific errors
    if (error instanceof Prisma.PrismaClientKnownRequestError) {
      // Unique constraint violation
      if (error.code === 'P2002') {
        Sentry.captureException(error, {
          level: 'warning',
          tags: {
            errorCode: 'P2002',
            errorType: 'duplicate_entry'
          },
          extra: {
            field: error.meta?.target,
            email
          }
        })

        return {
          success: false,
          error: 'Email already exists'
        }
      }

      // Record not found
      if (error.code === 'P2025') {
        Sentry.captureException(error, {
          level: 'warning',
          tags: {
            errorCode: 'P2025',
            errorType: 'not_found'
          }
        })

        return {
          success: false,
          error: 'Record not found'
        }
      }
    }

    // Unknown error - capture with high severity
    Sentry.captureException(error, {
      level: 'error',
      tags: {
        action: 'createUser',
        errorType: 'unknown'
      }
    })

    return {
      success: false,
      error: 'An unexpected error occurred'
    }
  }
}
```

---

## Error Boundaries

### Route Error Boundary

**File:** `app/dashboard/error.tsx`

```typescript
'use client'
import * as Sentry from '@sentry/nextjs'
import { useEffect } from 'react'

export default function DashboardError({
  error,
  reset
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  useEffect(() => {
    // Capture to Sentry with component context
    Sentry.captureException(error, {
      tags: {
        component: 'DashboardError',
        route: '/dashboard',
        digest: error.digest
      },
      contexts: {
        react: {
          componentStack: error.stack
        }
      }
    })
  }, [error])

  return (
    <div className="error-container">
      <h2>Something went wrong!</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  )
}
```

### Global Error Boundary

**File:** `app/global-error.tsx`

```typescript
'use client'
import * as Sentry from '@sentry/nextjs'
import { useEffect } from 'react'

export default function GlobalError({
  error,
  reset
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  useEffect(() => {
    // Critical error - capture with high severity
    Sentry.captureException(error, {
      level: 'fatal',
      tags: {
        component: 'GlobalError',
        digest: error.digest
      }
    })
  }, [error])

  return (
    <html>
      <body>
        <h1>Application Error</h1>
        <p>Something went very wrong</p>
        <button onClick={reset}>Try again</button>
      </body>
    </html>
  )
}
```

### Custom Error Boundary Component

```typescript
// components/ErrorBoundary.tsx
'use client'
import * as Sentry from '@sentry/nextjs'
import { Component, ReactNode } from 'react'

interface Props {
  children: ReactNode
  fallback?: ReactNode
  context?: string
}

interface State {
  hasError: boolean
  error?: Error
}

export class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props)
    this.state = { hasError: false }
  }

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error }
  }

  componentDidCatch(error: Error, errorInfo: any) {
    Sentry.withScope((scope) => {
      scope.setTag('boundary', this.props.context || 'custom')
      scope.setContext('react', {
        componentStack: errorInfo.componentStack
      })

      Sentry.captureException(error)
    })
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || (
        <div className="error">
          <h2>Something went wrong</h2>
          <button onClick={() => this.setState({ hasError: false })}>
            Try again
          </button>
        </div>
      )
    }

    return this.props.children
  }
}

// Usage
<ErrorBoundary context="chart" fallback={<div>Chart failed to load</div>}>
  <ComplexChart data={data} />
</ErrorBoundary>
```

---

## API Route Error Handling

### GET Route

```typescript
// app/api/posts/route.ts
import * as Sentry from '@sentry/nextjs'
import { NextRequest } from 'next/server'
import { getCurrentUser } from '@/utils/supabase/server'
import { prisma } from '@/lib/prisma'

export async function GET(request: NextRequest) {
  return await Sentry.startSpan(
    {
      name: 'GET /api/posts',
      op: 'http.server',
      attributes: {
        'http.method': 'GET',
        'http.route': '/api/posts'
      }
    },
    async () => {
      try {
        const user = await getCurrentUser()
        if (!user) {
          return Response.json(
            { error: 'Unauthorized' },
            { status: 401 }
          )
        }

        const posts = await prisma.post.findMany({
          where: { userId: user.id },
          take: 20
        })

        return Response.json({ posts })
      } catch (error) {
        Sentry.captureException(error, {
          tags: {
            route: '/api/posts',
            method: 'GET'
          }
        })

        return Response.json(
          { error: 'Failed to fetch posts' },
          { status: 500 }
        )
      }
    }
  )
}
```

### POST Route with Validation

```typescript
// app/api/posts/route.ts
import * as Sentry from '@sentry/nextjs'
import { NextRequest } from 'next/server'
import { z } from 'zod'

const createPostSchema = z.object({
  title: z.string().min(1).max(200),
  content: z.string().min(1)
})

export async function POST(request: NextRequest) {
  return await Sentry.startSpan(
    {
      name: 'POST /api/posts',
      op: 'http.server'
    },
    async () => {
      try {
        const user = await getCurrentUser()
        if (!user) {
          return Response.json({ error: 'Unauthorized' }, { status: 401 })
        }

        const body = await request.json()

        // Validation errors - don't send to Sentry
        const result = createPostSchema.safeParse(body)
        if (!result.success) {
          return Response.json(
            { error: 'Invalid input', details: result.error.flatten() },
            { status: 400 }
          )
        }

        const post = await prisma.post.create({
          data: {
            title: result.data.title,
            content: result.data.content,
            userId: user.id
          }
        })

        return Response.json({ post }, { status: 201 })
      } catch (error) {
        Sentry.captureException(error, {
          tags: {
            route: '/api/posts',
            method: 'POST',
            userId: user?.id
          },
          extra: {
            bodyKeys: Object.keys(body || {})
          }
        })

        return Response.json(
          { error: 'Failed to create post' },
          { status: 500 }
        )
      }
    }
  )
}
```

---

## Middleware Error Tracking

### Middleware with Error Tracking

```typescript
// middleware.ts
import * as Sentry from '@sentry/nextjs'
import { type NextRequest, NextResponse } from 'next/server'
import { updateSession } from '@/utils/supabase/middleware'

export async function middleware(request: NextRequest) {
  return await Sentry.startSpan(
    {
      name: 'middleware',
      op: 'middleware',
      attributes: {
        'http.method': request.method,
        'http.url': request.url,
        'http.pathname': request.nextUrl.pathname
      }
    },
    async () => {
      try {
        // Token refresh
        const response = await updateSession(request)

        return response
      } catch (error) {
        // Capture middleware errors
        Sentry.captureException(error, {
          tags: {
            middleware: 'auth',
            pathname: request.nextUrl.pathname
          },
          extra: {
            url: request.url,
            method: request.method
          }
        })

        // Let the error propagate or return error response
        return NextResponse.json(
          { error: 'Middleware error' },
          { status: 500 }
        )
      }
    }
  )
}

export const config = {
  matcher: [
    '/((?!_next/static|_next/image|favicon.ico).*)'
  ]
}
```

### Auth Middleware Error Context

```typescript
// utils/supabase/middleware.ts
import * as Sentry from '@sentry/nextjs'
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
          // ... cookie logic
        }
      }
    }
  )

  try {
    const { data: { user }, error } = await supabase.auth.getUser()

    if (error) {
      // Don't capture expected JWT expiry
      if (!error.message.includes('JWT expired')) {
        Sentry.captureException(error, {
          tags: {
            auth: 'token_refresh',
            pathname: request.nextUrl.pathname
          }
        })
      }
    }

    return supabaseResponse
  } catch (error) {
    Sentry.captureException(error, {
      level: 'error',
      tags: {
        auth: 'middleware_crash',
        pathname: request.nextUrl.pathname
      }
    })

    throw error
  }
}
```

---

## Performance Monitoring

### Server Action Performance

```typescript
// actions/dashboard.ts
'use server'
import * as Sentry from '@sentry/nextjs'
import { prisma } from '@/lib/prisma'
import { getCurrentUser } from '@/utils/supabase/server'

export async function getDashboardData() {
  return await Sentry.startSpan(
    {
      name: 'getDashboardData',
      op: 'function.server_action',
      attributes: {
        action: 'getDashboardData'
      }
    },
    async (span) => {
      const user = await getCurrentUser()
      if (!user) {
        return { success: false, error: 'Unauthorized' }
      }

      // Add user context to span
      span.setAttributes({
        userId: user.id
      })

      try {
        // Track parallel queries
        const [posts, comments, stats] = await Promise.all([
          Sentry.startSpan(
            { name: 'fetchPosts', op: 'db.query' },
            () => prisma.post.findMany({ where: { userId: user.id } })
          ),
          Sentry.startSpan(
            { name: 'fetchComments', op: 'db.query' },
            () => prisma.comment.findMany({ where: { userId: user.id } })
          ),
          Sentry.startSpan(
            { name: 'fetchStats', op: 'db.query' },
            () => prisma.user.findUnique({
              where: { id: user.id },
              select: {
                _count: {
                  select: { posts: true, comments: true }
                }
              }
            })
          )
        ])

        return {
          success: true,
          data: { posts, comments, stats }
        }
      } catch (error) {
        Sentry.captureException(error)
        return { success: false, error: 'Failed to load dashboard' }
      }
    }
  )
}
```

### Prisma Query Performance

```typescript
// lib/prisma.ts
import { PrismaClient } from '@prisma/client'
import * as Sentry from '@sentry/nextjs'

const prismaClientSingleton = () => {
  return new PrismaClient({
    log: [
      {
        emit: 'event',
        level: 'query'
      },
      {
        emit: 'event',
        level: 'error'
      }
    ]
  })
}

const prisma = prismaClientSingleton()

// Monitor slow queries
prisma.$on('query' as any, (e: any) => {
  if (e.duration > 100) {
    Sentry.captureMessage(`Slow query detected: ${e.duration}ms`, {
      level: 'warning',
      tags: {
        query: 'slow',
        duration: e.duration
      },
      extra: {
        query: e.query,
        params: e.params
      }
    })
  }
})

// Monitor query errors
prisma.$on('error' as any, (e: any) => {
  Sentry.captureException(new Error(e.message), {
    tags: {
      prisma: 'error',
      target: e.target
    }
  })
})

export { prisma }
```

### Database Transaction Performance

```typescript
// actions/staff.ts
'use server'
import * as Sentry from '@sentry/nextjs'

export async function complexOperation() {
  return await Sentry.startSpan(
    {
      name: 'complexOperation',
      op: 'function'
    },
    async () => {
      return await Sentry.startSpan(
        {
          name: 'transaction',
          op: 'db.transaction'
        },
        async (span) => {
          const result = await prisma.$transaction(async (tx) => {
            // Multiple operations tracked as single transaction
            const step1 = await tx.user.create({ data: { ... } })
            const step2 = await tx.profile.create({ data: { ... } })
            const step3 = await tx.auditLog.create({ data: { ... } })

            return { step1, step2, step3 }
          })

          span.setAttributes({
            operationCount: 3,
            success: true
          })

          return result
        }
      )
    }
  )
}
```

---

## Error Context Best Practices

### Rich Context Example

```typescript
// actions/posts.ts
'use server'
import * as Sentry from '@sentry/nextjs'

export async function complexOperation(postId: string) {
  const user = await getCurrentUser()
  if (!user) throw new Error('Unauthorized')

  // Use scope for rich context
  return await Sentry.withScope(async (scope) => {
    // User context
    scope.setUser({
      id: user.id,
      email: user.email
    })

    // Tags for filtering in Sentry UI
    scope.setTag('action', 'complexOperation')
    scope.setTag('postId', postId)
    scope.setTag('environment', process.env.NODE_ENV)

    // Structured context
    scope.setContext('operation', {
      type: 'post.update',
      postId,
      timestamp: new Date().toISOString()
    })

    // Breadcrumbs for timeline
    scope.addBreadcrumb({
      category: 'operation',
      message: 'Starting post update',
      level: 'info',
      data: { postId }
    })

    try {
      const post = await prisma.post.update({
        where: { id: postId },
        data: { ... }
      })

      scope.addBreadcrumb({
        category: 'operation',
        message: 'Post updated successfully',
        level: 'info'
      })

      return { success: true, post }
    } catch (error) {
      scope.addBreadcrumb({
        category: 'error',
        message: 'Post update failed',
        level: 'error',
        data: { error: error.message }
      })

      Sentry.captureException(error)

      return { success: false, error: 'Failed to update post' }
    }
  })
}
```

### Breadcrumbs for User Actions

```typescript
// components/PostForm.tsx
'use client'
import * as Sentry from '@sentry/nextjs'
import { useState } from 'react'

export function PostForm() {
  const [title, setTitle] = useState('')

  function handleTitleChange(e: React.ChangeEvent<HTMLInputElement>) {
    setTitle(e.target.value)

    // Track user interactions
    Sentry.addBreadcrumb({
      category: 'ui.input',
      message: 'User updated post title',
      level: 'info',
      data: {
        titleLength: e.target.value.length
      }
    })
  }

  async function handleSubmit(formData: FormData) {
    Sentry.addBreadcrumb({
      category: 'ui.submit',
      message: 'User submitted post form',
      level: 'info',
      data: {
        hasTitle: !!formData.get('title'),
        hasContent: !!formData.get('content')
      }
    })

    try {
      const result = await createPost(formData)

      if (!result.success) {
        Sentry.captureMessage('Form submission failed', {
          level: 'warning',
          extra: { error: result.error }
        })
      }
    } catch (error) {
      Sentry.captureException(error)
    }
  }

  return (
    <form action={handleSubmit}>
      <input
        name="title"
        value={title}
        onChange={handleTitleChange}
      />
      <button type="submit">Submit</button>
    </form>
  )
}
```

---

## Common Mistakes

### ❌ Swallowing Errors

```typescript
// ❌ BAD - Silent failure
export async function createPost(data: FormData) {
  try {
    await riskyOperation()
  } catch (error) {
    // Silent - no logging, no Sentry, nothing!
  }
}

// ✅ GOOD - Capture and handle
export async function createPost(data: FormData) {
  try {
    await riskyOperation()
  } catch (error) {
    Sentry.captureException(error, {
      tags: { action: 'createPost' }
    })
    return { success: false, error: 'Failed to create post' }
  }
}
```

### ❌ Generic Error Messages

```typescript
// ❌ BAD - No context
throw new Error('Error occurred')

// ✅ GOOD - Descriptive with context
throw new Error(`Failed to create post: ${title} (userId: ${userId})`)
```

### ❌ Exposing Sensitive Data

```typescript
// ❌ BAD - PII in logs
Sentry.captureException(error, {
  extra: {
    password: user.password,  // NEVER
    ssn: user.ssn,            // NEVER
    creditCard: payment.cc    // NEVER
  }
})

// ✅ GOOD - Scrubbed data
Sentry.captureException(error, {
  extra: {
    userId: user.id,
    hasPassword: !!user.password,
    paymentMethod: payment.type // 'card', not number
  }
})
```

### ❌ Missing Async Error Handling

```typescript
// ❌ BAD - Unhandled promise
async function bad() {
  fetchData().then(data => processResult(data))
  // If fetchData fails, unhandled rejection!
}

// ✅ GOOD - Proper async/await with try-catch
async function good() {
  try {
    const data = await fetchData()
    await processResult(data)
  } catch (error) {
    Sentry.captureException(error)
    throw error
  }
}
```

### ❌ Not Using Scopes

```typescript
// ❌ BAD - No context isolation
Sentry.setTag('userId', user1.id)
await operation1()  // Has user1 context

Sentry.setTag('userId', user2.id)
await operation2()  // Overwrites! Now both have user2

// ✅ GOOD - Use scopes for isolation
await Sentry.withScope(async (scope) => {
  scope.setTag('userId', user1.id)
  await operation1()  // Only this has user1 context
})

await Sentry.withScope(async (scope) => {
  scope.setTag('userId', user2.id)
  await operation2()  // Only this has user2 context
})
```

### ❌ Capturing Expected Errors

```typescript
// ❌ BAD - Clutters Sentry with noise
export async function getPost(id: string) {
  const post = await prisma.post.findUnique({ where: { id } })

  if (!post) {
    Sentry.captureException(new Error('Post not found'))  // Don't do this!
    return null
  }

  return post
}

// ✅ GOOD - Only capture unexpected errors
export async function getPost(id: string) {
  try {
    const post = await prisma.post.findUnique({ where: { id } })

    // Not found is expected - just return null
    if (!post) return null

    return post
  } catch (error) {
    // Unexpected database error - capture this
    Sentry.captureException(error, {
      tags: { action: 'getPost', postId: id }
    })
    throw error
  }
}
```

### ❌ Not Adding Performance Spans

```typescript
// ❌ BAD - No performance tracking
export async function getDashboard() {
  const posts = await prisma.post.findMany()
  const comments = await prisma.comment.findMany()
  return { posts, comments }
}

// ✅ GOOD - Track performance
export async function getDashboard() {
  return await Sentry.startSpan(
    { name: 'getDashboard', op: 'function' },
    async () => {
      const [posts, comments] = await Promise.all([
        Sentry.startSpan(
          { name: 'fetchPosts', op: 'db.query' },
          () => prisma.post.findMany()
        ),
        Sentry.startSpan(
          { name: 'fetchComments', op: 'db.query' },
          () => prisma.comment.findMany()
        )
      ])

      return { posts, comments }
    }
  )
}
```

---

**Related Files:**
- [architecture-overview.md](architecture-overview.md) - Two-path architecture
- [async-and-errors.md](async-and-errors.md) - Error handling patterns
- [database-patterns.md](database-patterns.md) - Prisma error handling
- [middleware-guide.md](middleware-guide.md) - Auth middleware patterns
