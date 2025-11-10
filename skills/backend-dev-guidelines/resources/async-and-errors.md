# Async Patterns and Error Handling - Next.js + Supabase

Complete guide to async/await patterns and error handling in Next.js with Supabase. Covers server actions, error boundaries, Prisma errors, and client component error states.

## Table of Contents

- [Async/Await Best Practices](#asyncawait-best-practices)
- [Error Handling in Server Actions](#error-handling-in-server-actions)
- [Next.js Error Boundaries](#nextjs-error-boundaries)
- [Client Component Error Handling](#client-component-error-handling)
- [Custom Error Types](#custom-error-types)
- [Prisma Error Handling](#prisma-error-handling)
- [Common Async Pitfalls](#common-async-pitfalls)
- [What NOT to Do](#what-not-to-do)
- [Error Handling Decision Tree](#error-handling-decision-tree)

---

## Async/Await Best Practices

### Always Use Try-Catch

Async functions can throw errors. Always wrap await calls in try-catch blocks.

```typescript
// actions/posts.ts
'use server'
import { prisma } from '@/lib/prisma'

// ❌ BAD - Unhandled errors
export async function getPosts() {
  const posts = await prisma.post.findMany()  // If throws, unhandled!
  return posts
}

// ✅ GOOD - Wrapped in try-catch
export async function getPosts() {
  try {
    const posts = await prisma.post.findMany()
    return { success: true, posts }
  } catch (error) {
    console.error('Failed to fetch posts:', error)
    return { success: false, error: 'Failed to fetch posts' }
  }
}
```

### Avoid .then() Chains

Use async/await instead of promise chains for better readability.

```typescript
// ❌ BAD - Promise chains
function processData(userId: string) {
  return prisma.user.findUnique({ where: { id: userId } })
    .then(user => prisma.post.findMany({ where: { userId: user.id } }))
    .then(posts => posts.map(p => ({ ...p, processed: true })))
    .catch(error => {
      console.error(error)
      throw error
    })
}

// ✅ GOOD - Async/await
async function processData(userId: string) {
  try {
    const user = await prisma.user.findUnique({
      where: { id: userId }
    })

    if (!user) throw new Error('User not found')

    const posts = await prisma.post.findMany({
      where: { userId: user.id }
    })

    return posts.map(p => ({ ...p, processed: true }))
  } catch (error) {
    console.error('Failed to process data:', error)
    throw error
  }
}
```

### Parallel Operations

Run independent operations in parallel to improve performance.

```typescript
// actions/dashboard.ts
'use server'
import { getCurrentUser } from '@/utils/supabase/server'
import { prisma } from '@/lib/prisma'

// ❌ BAD - Sequential (slow)
export async function getDashboardData() {
  const user = await getCurrentUser()
  if (!user) throw new Error('Unauthorized')

  const posts = await prisma.post.findMany()      // Wait 100ms
  const comments = await prisma.comment.findMany() // Wait 150ms
  const likes = await prisma.like.findMany()       // Wait 80ms

  // Total: 330ms
  return { posts, comments, likes }
}

// ✅ GOOD - Parallel (fast)
export async function getDashboardData() {
  const user = await getCurrentUser()
  if (!user) throw new Error('Unauthorized')

  try {
    const [posts, comments, likes] = await Promise.all([
      prisma.post.findMany(),
      prisma.comment.findMany(),
      prisma.like.findMany()
    ])

    // Total: 150ms (slowest query)
    return { success: true, posts, comments, likes }
  } catch (error) {
    console.error('Dashboard fetch failed:', error)
    return { success: false, error: 'Failed to load dashboard' }
  }
}
```

### Promise.all vs Promise.allSettled

**Promise.all** - Fail fast (one error fails all):
```typescript
try {
  const [users, posts, comments] = await Promise.all([
    prisma.user.findMany(),
    prisma.post.findMany(),
    prisma.comment.findMany()
  ])
  // All succeeded
} catch (error) {
  // One failed, all results lost
}
```

**Promise.allSettled** - All operations complete regardless:
```typescript
const results = await Promise.allSettled([
  prisma.user.findMany(),
  prisma.post.findMany(),
  prisma.comment.findMany()
])

const users = results[0].status === 'fulfilled' ? results[0].value : []
const posts = results[1].status === 'fulfilled' ? results[1].value : []
const comments = results[2].status === 'fulfilled' ? results[2].value : []

// Handle partial failures
results.forEach((result, i) => {
  if (result.status === 'rejected') {
    console.error(`Operation ${i} failed:`, result.reason)
  }
})
```

**When to use which:**
- **Promise.all**: All operations must succeed (critical data)
- **Promise.allSettled**: Some operations can fail (optional data)

**See also:** [database-patterns.md](database-patterns.md#parallel-queries) for more parallel query patterns.

---

## Error Handling in Server Actions

Server actions are the primary place for error handling in Next.js. You have two approaches: **throw errors** or **return error objects**.

### Approach 1: Return Error Objects (Recommended)

Return success/error objects to give components full control.

```typescript
// actions/posts.ts
'use server'
import { getCurrentUser } from '@/utils/supabase/server'
import { prisma } from '@/lib/prisma'
import { revalidatePath } from 'next/cache'

export async function createPost(formData: FormData) {
  try {
    // 1. Auth check
    const user = await getCurrentUser()
    if (!user) {
      return { success: false, error: 'Unauthorized' }
    }

    // 2. Validation
    const title = formData.get('title') as string
    const content = formData.get('content') as string

    if (!title || !content) {
      return { success: false, error: 'Title and content required' }
    }

    // 3. Create post
    const post = await prisma.post.create({
      data: {
        title,
        content,
        userId: user.id
      }
    })

    // 4. Revalidate
    revalidatePath('/posts')

    return { success: true, post }
  } catch (error) {
    console.error('Failed to create post:', error)
    return {
      success: false,
      error: error instanceof Error ? error.message : 'Failed to create post'
    }
  }
}
```

**Benefits:**
- Component can show specific error messages
- No error boundaries triggered
- Full control over error handling
- Better UX (stay on page, show inline error)

### Approach 2: Throw Errors

Throw errors to trigger error.tsx boundaries.

```typescript
// actions/posts.ts
'use server'
export async function createPost(formData: FormData) {
  const user = await getCurrentUser()
  if (!user) throw new Error('Unauthorized')

  const title = formData.get('title') as string
  if (!title) throw new Error('Title required')

  const post = await prisma.post.create({
    data: { title, userId: user.id }
  })

  revalidatePath('/posts')
  return post
}
```

**Benefits:**
- Simpler code (no success/error objects)
- Error caught by error.tsx
- Good for catastrophic failures

**Drawbacks:**
- Triggers error boundary (replaces page content)
- Less control over error display
- Worse UX for validation errors

### Decision: When to Use Which?

**Return error objects when:**
- ✅ Form validation errors (show inline)
- ✅ User-fixable errors (wrong password, etc.)
- ✅ Want to stay on current page
- ✅ Need specific error messages per field

**Throw errors when:**
- ✅ Catastrophic failures (database down)
- ✅ Authorization failures (403, 404)
- ✅ Unexpected errors (bugs)
- ✅ Want to show error page (error.tsx)

### Error Handling with revalidatePath

Be careful with revalidatePath and errors.

```typescript
// ❌ BAD - revalidatePath before error check
export async function updatePost(id: string, formData: FormData) {
  revalidatePath('/posts')  // Runs even if update fails!

  const post = await prisma.post.update({
    where: { id },
    data: { title: formData.get('title') as string }
  })

  return post
}

// ✅ GOOD - revalidatePath after success
export async function updatePost(id: string, formData: FormData) {
  try {
    const post = await prisma.post.update({
      where: { id },
      data: { title: formData.get('title') as string }
    })

    revalidatePath('/posts')  // Only if update succeeds
    return { success: true, post }
  } catch (error) {
    return { success: false, error: 'Failed to update post' }
  }
}
```

### Handling Errors from Multiple Actions

```typescript
// actions/posts.ts
'use server'
export async function createPostWithImage(formData: FormData) {
  try {
    const user = await getCurrentUser()
    if (!user) return { success: false, error: 'Unauthorized' }

    // Multiple operations that can fail
    const imageUrl = await uploadImage(formData.get('image'))
    const post = await prisma.post.create({
      data: {
        title: formData.get('title') as string,
        imageUrl,
        userId: user.id
      }
    })

    await sendNotification(user.id, 'Post created')

    revalidatePath('/posts')
    return { success: true, post }
  } catch (error) {
    // Clean up on error
    if (imageUrl) await deleteImage(imageUrl)

    return {
      success: false,
      error: 'Failed to create post with image'
    }
  }
}
```

---

## Next.js Error Boundaries

Next.js provides file-based error boundaries through `error.tsx` and `not-found.tsx` files.

### error.tsx - Route Error Boundary

**File:** `app/dashboard/error.tsx`

```typescript
'use client'
import { useEffect } from 'react'

export default function Error({
  error,
  reset
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  useEffect(() => {
    // Log error to monitoring service
    console.error('Dashboard error:', error)
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

**What it catches:**
- Errors in server components
- Errors thrown by server actions
- Errors during data fetching
- Runtime errors in the route

**What it doesn't catch:**
- Errors in error.tsx itself (use global-error.tsx)
- Errors in layout.tsx above it
- 404 errors (use not-found.tsx)

### global-error.tsx - Root Error Boundary

**File:** `app/global-error.tsx`

```typescript
'use client'

export default function GlobalError({
  error,
  reset
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  return (
    <html>
      <body>
        <h2>Application Error</h2>
        <p>Something went very wrong</p>
        <button onClick={reset}>Try again</button>
      </body>
    </html>
  )
}
```

**Use for:**
- Errors in root layout
- Last-resort error handling
- Catastrophic failures

### not-found.tsx - 404 Errors

**File:** `app/dashboard/not-found.tsx`

```typescript
export default function NotFound() {
  return (
    <div>
      <h2>Not Found</h2>
      <p>Could not find requested resource</p>
      <a href="/dashboard">Return to Dashboard</a>
    </div>
  )
}
```

**Triggering 404:**
```typescript
// app/posts/[id]/page.tsx
import { notFound } from 'next/navigation'
import { prisma } from '@/lib/prisma'

export default async function PostPage({ params }: { params: { id: string } }) {
  const post = await prisma.post.findUnique({
    where: { id: params.id }
  })

  if (!post) {
    notFound()  // Triggers not-found.tsx
  }

  return <div>{post.title}</div>
}
```

### Error Boundary Hierarchy

```
app/
├── global-error.tsx           # Catches everything
├── layout.tsx
├── error.tsx                  # Catches app-level errors
├── page.tsx
└── dashboard/
    ├── layout.tsx
    ├── error.tsx              # Catches dashboard errors (most specific)
    ├── not-found.tsx          # 404s in dashboard
    └── page.tsx
```

**Error resolution order:**
1. Most specific error.tsx (closest to error)
2. Parent error.tsx
3. global-error.tsx

### Resetting Error Boundaries

The `reset()` function re-renders the error boundary.

```typescript
// error.tsx
'use client'
import { useState } from 'react'

export default function Error({ error, reset }) {
  const [attempts, setAttempts] = useState(0)

  function handleReset() {
    setAttempts(a => a + 1)
    reset()
  }

  return (
    <div>
      <h2>Error occurred</h2>
      {attempts > 2 ? (
        <p>Multiple attempts failed. Please refresh the page.</p>
      ) : (
        <button onClick={handleReset}>
          Try again {attempts > 0 && `(${attempts} attempts)`}
        </button>
      )}
    </div>
  )
}
```

---

## Client Component Error Handling

Client components handle errors through state management and user feedback.

### Form Error States

**Pattern 1: useState for errors**

```typescript
// components/CreatePostForm.tsx
'use client'
import { createPost } from '@/actions/posts'
import { useState, useTransition } from 'react'

export function CreatePostForm() {
  const [error, setError] = useState<string | null>(null)
  const [isPending, startTransition] = useTransition()

  async function handleSubmit(formData: FormData) {
    setError(null)  // Clear previous errors

    startTransition(async () => {
      const result = await createPost(formData)

      if (!result.success) {
        setError(result.error)
      } else {
        // Success - clear form or redirect
      }
    })
  }

  return (
    <form action={handleSubmit}>
      {error && (
        <div className="error-message" role="alert">
          {error}
        </div>
      )}

      <input name="title" required />
      <textarea name="content" required />

      <button type="submit" disabled={isPending}>
        {isPending ? 'Creating...' : 'Create Post'}
      </button>
    </form>
  )
}
```

**Pattern 2: Field-specific errors**

```typescript
'use client'
import { useState } from 'react'

interface FormErrors {
  title?: string
  content?: string
  general?: string
}

export function CreatePostForm() {
  const [errors, setErrors] = useState<FormErrors>({})
  const [isPending, startTransition] = useTransition()

  async function handleSubmit(formData: FormData) {
    setErrors({})

    startTransition(async () => {
      const result = await createPost(formData)

      if (!result.success) {
        if (result.errors) {
          setErrors(result.errors)  // Field-specific errors
        } else {
          setErrors({ general: result.error })
        }
      }
    })
  }

  return (
    <form action={handleSubmit}>
      {errors.general && (
        <div className="error-general">{errors.general}</div>
      )}

      <div>
        <input name="title" />
        {errors.title && <span className="error">{errors.title}</span>}
      </div>

      <div>
        <textarea name="content" />
        {errors.content && <span className="error">{errors.content}</span>}
      </div>

      <button type="submit" disabled={isPending}>Submit</button>
    </form>
  )
}
```

### Optimistic Updates with Error Recovery

```typescript
'use client'
import { useState, useOptimistic } from 'react'
import { createPost } from '@/actions/posts'

export function PostsList({ initialPosts }) {
  const [error, setError] = useState<string | null>(null)
  const [optimisticPosts, addOptimisticPost] = useOptimistic(
    initialPosts,
    (state, newPost) => [...state, newPost]
  )

  async function handleCreate(formData: FormData) {
    setError(null)

    // Optimistic update
    const tempPost = {
      id: 'temp-' + Date.now(),
      title: formData.get('title') as string,
      createdAt: new Date()
    }
    addOptimisticPost(tempPost)

    // Actual create
    const result = await createPost(formData)

    if (!result.success) {
      // Rollback happens automatically
      setError(result.error)
    }
  }

  return (
    <div>
      {error && <div className="error">{error}</div>}

      <form action={handleCreate}>
        <input name="title" />
        <button type="submit">Create</button>
      </form>

      {optimisticPosts.map(post => (
        <div key={post.id}>
          {post.title}
          {post.id.startsWith('temp-') && <span>(Saving...)</span>}
        </div>
      ))}
    </div>
  )
}
```

### Loading and Error States

```typescript
'use client'
import { useState, useEffect } from 'react'

export function PostsList() {
  const [posts, setPosts] = useState([])
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<string | null>(null)

  useEffect(() => {
    async function loadPosts() {
      try {
        setLoading(true)
        setError(null)

        const result = await getPosts()

        if (result.success) {
          setPosts(result.posts)
        } else {
          setError(result.error)
        }
      } catch (err) {
        setError('Failed to load posts')
      } finally {
        setLoading(false)
      }
    }

    loadPosts()
  }, [])

  if (loading) return <div>Loading...</div>
  if (error) return <div className="error">{error}</div>

  return (
    <div>
      {posts.map(post => (
        <div key={post.id}>{post.title}</div>
      ))}
    </div>
  )
}
```

### React Error Boundary (Class Component)

For catching errors in client component trees.

```typescript
// components/ErrorBoundary.tsx
'use client'
import { Component, ReactNode } from 'react'

interface Props {
  children: ReactNode
  fallback?: ReactNode
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
    console.error('ErrorBoundary caught:', error, errorInfo)
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
<ErrorBoundary fallback={<div>Chart failed to load</div>}>
  <ComplexChart data={data} />
</ErrorBoundary>
```

---

## Custom Error Types

Define custom error classes for better error handling and user messages.

### Base Error Class

```typescript
// lib/errors.ts
export class AppError extends Error {
  constructor(
    message: string,
    public code: string,
    public statusCode: number,
    public isOperational: boolean = true
  ) {
    super(message)
    this.name = this.constructor.name
    Error.captureStackTrace(this, this.constructor)
  }
}
```

### Specific Error Types

```typescript
// lib/errors.ts
export class ValidationError extends AppError {
  constructor(message: string) {
    super(message, 'VALIDATION_ERROR', 400)
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string) {
    super(`${resource} not found`, 'NOT_FOUND', 404)
  }
}

export class UnauthorizedError extends AppError {
  constructor(message: string = 'Unauthorized') {
    super(message, 'UNAUTHORIZED', 401)
  }
}

export class ForbiddenError extends AppError {
  constructor(message: string = 'Forbidden') {
    super(message, 'FORBIDDEN', 403)
  }
}

export class ConflictError extends AppError {
  constructor(message: string) {
    super(message, 'CONFLICT', 409)
  }
}
```

### Usage in Server Actions

```typescript
// actions/posts.ts
'use server'
import { NotFoundError, UnauthorizedError, ValidationError } from '@/lib/errors'

export async function deletePost(id: string) {
  try {
    const user = await getCurrentUser()
    if (!user) throw new UnauthorizedError()

    const post = await prisma.post.findUnique({ where: { id } })
    if (!post) throw new NotFoundError('Post')

    if (post.userId !== user.id) {
      throw new ForbiddenError('You can only delete your own posts')
    }

    await prisma.post.delete({ where: { id } })
    revalidatePath('/posts')

    return { success: true }
  } catch (error) {
    if (error instanceof AppError) {
      return { success: false, error: error.message, code: error.code }
    }

    console.error('Unexpected error:', error)
    return { success: false, error: 'An unexpected error occurred' }
  }
}
```

### Handling Custom Errors in Components

```typescript
'use client'
export function DeletePostButton({ postId }: { postId: string }) {
  const [error, setError] = useState<string | null>(null)

  async function handleDelete() {
    const result = await deletePost(postId)

    if (!result.success) {
      // Show user-friendly message based on error code
      if (result.code === 'UNAUTHORIZED') {
        setError('Please log in to delete posts')
      } else if (result.code === 'NOT_FOUND') {
        setError('Post not found')
      } else if (result.code === 'FORBIDDEN') {
        setError('You can only delete your own posts')
      } else {
        setError(result.error)
      }
    }
  }

  return (
    <>
      {error && <div className="error">{error}</div>}
      <button onClick={handleDelete}>Delete</button>
    </>
  )
}
```

---

## Prisma Error Handling

Prisma throws specific error types that need proper handling.

### Common Prisma Error Codes

| Code | Meaning | Example |
|------|---------|---------|
| P2002 | Unique constraint violation | Email already exists |
| P2003 | Foreign key constraint failed | Invalid organization ID |
| P2025 | Record not found | User doesn't exist |
| P2014 | Required relation violation | Can't delete user with posts |
| P2034 | Transaction conflict | Concurrent update conflict |

### Handling Prisma Errors

```typescript
// actions/users.ts
'use server'
import { Prisma } from '@prisma/client'
import { ConflictError, NotFoundError, ValidationError } from '@/lib/errors'

export async function createUser(formData: FormData) {
  try {
    const email = formData.get('email') as string
    const name = formData.get('name') as string

    const user = await prisma.user.create({
      data: { email, name }
    })

    return { success: true, user }
  } catch (error) {
    if (error instanceof Prisma.PrismaClientKnownRequestError) {
      // Unique constraint violation
      if (error.code === 'P2002') {
        const field = (error.meta?.target as string[])?.[0] || 'Field'
        throw new ConflictError(`${field} already exists`)
      }

      // Foreign key constraint
      if (error.code === 'P2003') {
        throw new ValidationError('Invalid reference - related record not found')
      }

      // Record not found
      if (error.code === 'P2025') {
        throw new NotFoundError('Record')
      }
    }

    // Unknown error
    console.error('Database error:', error)
    throw new Error('An unexpected database error occurred')
  }
}
```

### Prisma Transaction Errors

```typescript
// actions/staff.ts
'use server'
export async function addStaffMember(orgId: string, email: string) {
  try {
    return await prisma.$transaction(async (tx) => {
      // Business rule check
      const org = await tx.organization.findUnique({
        where: { id: orgId },
        include: { _count: { select: { members: true } } }
      })

      if (!org) {
        throw new NotFoundError('Organization')
      }

      if (org._count.members >= org.maxMembers) {
        throw new ValidationError('Staff limit reached. Upgrade your plan.')
      }

      // Create member
      return await tx.organizationMember.create({
        data: { email, organizationId: orgId }
      })
    })
  } catch (error) {
    // Handle business logic errors
    if (error instanceof AppError) {
      return { success: false, error: error.message }
    }

    // Handle Prisma errors
    if (error instanceof Prisma.PrismaClientKnownRequestError) {
      if (error.code === 'P2002') {
        return { success: false, error: 'User is already a member' }
      }

      if (error.code === 'P2034') {
        return { success: false, error: 'Update conflict. Please try again.' }
      }
    }

    console.error('Transaction error:', error)
    return { success: false, error: 'Failed to add staff member' }
  }
}
```

### Prisma Error Helper

Create a helper to map Prisma errors to user-friendly messages.

```typescript
// lib/prisma-errors.ts
import { Prisma } from '@prisma/client'

export function handlePrismaError(error: unknown): string {
  if (error instanceof Prisma.PrismaClientKnownRequestError) {
    switch (error.code) {
      case 'P2002': {
        const field = (error.meta?.target as string[])?.[0] || 'Field'
        return `${field} already exists`
      }
      case 'P2003':
        return 'Invalid reference'
      case 'P2025':
        return 'Record not found'
      case 'P2014':
        return 'Cannot delete record with existing relations'
      case 'P2034':
        return 'Update conflict occurred'
      default:
        return 'Database operation failed'
    }
  }

  if (error instanceof Prisma.PrismaClientValidationError) {
    return 'Invalid data provided'
  }

  return 'An unexpected error occurred'
}

// Usage
export async function createUser(email: string) {
  try {
    return await prisma.user.create({ data: { email } })
  } catch (error) {
    const message = handlePrismaError(error)
    return { success: false, error: message }
  }
}
```

**See also:** [database-patterns.md](database-patterns.md#error-handling) for more Prisma error patterns.

---

## Common Async Pitfalls

### Fire and Forget (Unhandled Promises)

Calling async functions without await or error handling can lead to silent failures.

```typescript
// ❌ BAD - Fire and forget
export async function createPost(formData: FormData) {
  const user = await getCurrentUser()
  if (!user) throw new Error('Unauthorized')

  const post = await prisma.post.create({
    data: {
      title: formData.get('title') as string,
      userId: user.id
    }
  })

  sendEmailNotification(user.email, post)  // ⚠️ Fires async, errors unhandled!

  return { success: true, post }
}

// ✅ GOOD - Await and handle
export async function createPost(formData: FormData) {
  const user = await getCurrentUser()
  if (!user) throw new Error('Unauthorized')

  const post = await prisma.post.create({
    data: {
      title: formData.get('title') as string,
      userId: user.id
    }
  })

  try {
    await sendEmailNotification(user.email, post)
  } catch (error) {
    console.error('Email notification failed:', error)
    // Don't fail the whole operation if email fails
  }

  return { success: true, post }
}

// ✅ ALTERNATIVE - Intentional background task
export async function createPost(formData: FormData) {
  const user = await getCurrentUser()
  if (!user) throw new Error('Unauthorized')

  const post = await prisma.post.create({
    data: {
      title: formData.get('title') as string,
      userId: user.id
    }
  })

  // Fire and forget with explicit error handling
  sendEmailNotification(user.email, post).catch(error => {
    console.error('Email notification failed:', error)
  })

  return { success: true, post }
}
```

### Sequential Instead of Parallel

```typescript
// ❌ BAD - Sequential (slow)
export async function getDashboard(userId: string) {
  const user = await prisma.user.findUnique({ where: { id: userId } })
  const posts = await prisma.post.findMany({ where: { userId } })
  const comments = await prisma.comment.findMany({ where: { userId } })

  return { user, posts, comments }  // Took: 300ms
}

// ✅ GOOD - Parallel (fast)
export async function getDashboard(userId: string) {
  const [user, posts, comments] = await Promise.all([
    prisma.user.findUnique({ where: { id: userId } }),
    prisma.post.findMany({ where: { userId } }),
    prisma.comment.findMany({ where: { userId } })
  ])

  return { user, posts, comments }  // Took: 100ms
}
```

### Missing Await

```typescript
// ❌ BAD - Missing await
export async function createPost(formData: FormData) {
  const post = prisma.post.create({  // Missing await!
    data: { title: formData.get('title') as string }
  })

  revalidatePath('/posts')  // Runs before post is created!
  return post  // Returns a Promise, not the post!
}

// ✅ GOOD - Proper await
export async function createPost(formData: FormData) {
  const post = await prisma.post.create({
    data: { title: formData.get('title') as string }
  })

  revalidatePath('/posts')  // Runs after post is created
  return post  // Returns the post
}
```

### Unhandled Rejections

Global handlers catch unhandled promise rejections.

```typescript
// server.ts or instrumentation.ts
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection at:', promise, 'reason:', reason)
  // Log to monitoring service
})

process.on('uncaughtException', (error) => {
  console.error('Uncaught Exception:', error)
  // Log to monitoring service
  process.exit(1)
})
```

---

## What NOT to Do

### ❌ Don't Use Express Error Patterns

```typescript
// ❌ BAD - Express asyncErrorWrapper
function asyncErrorWrapper(handler) {
  return async (req, res, next) => {
    try {
      await handler(req, res, next)
    } catch (error) {
      next(error)
    }
  }
}

// ✅ GOOD - Server action try/catch
'use server'
export async function createPost(formData: FormData) {
  try {
    // ... logic
    return { success: true }
  } catch (error) {
    return { success: false, error: 'Failed' }
  }
}
```

### ❌ Don't Throw Errors in Client Components Without Boundaries

```typescript
// ❌ BAD - Throws in client component (crashes app if no boundary)
'use client'
export function PostsList({ posts }) {
  if (!posts) throw new Error('No posts')  // Crashes!
  return <div>{posts.map(...)}</div>
}

// ✅ GOOD - Handle errors gracefully
'use client'
export function PostsList({ posts }) {
  if (!posts) {
    return <div>No posts available</div>
  }
  return <div>{posts.map(...)}</div>
}
```

### ❌ Don't Return Sensitive Error Details

```typescript
// ❌ BAD - Exposes sensitive info
export async function createUser(email: string) {
  try {
    return await prisma.user.create({ data: { email } })
  } catch (error) {
    return {
      success: false,
      error: error.message  // Might contain DB connection strings!
    }
  }
}

// ✅ GOOD - Return user-friendly messages
export async function createUser(email: string) {
  try {
    return await prisma.user.create({ data: { email } })
  } catch (error) {
    console.error('Database error:', error)  // Log full error
    return {
      success: false,
      error: 'Failed to create user'  // Generic message
    }
  }
}
```

### ❌ Don't Use error.tsx for 404s

```typescript
// ❌ BAD - Using error.tsx for 404
export default async function PostPage({ params }) {
  const post = await prisma.post.findUnique({ where: { id: params.id } })

  if (!post) {
    throw new Error('Post not found')  // Triggers error.tsx (wrong!)
  }

  return <div>{post.title}</div>
}

// ✅ GOOD - Use notFound() for 404s
import { notFound } from 'next/navigation'

export default async function PostPage({ params }) {
  const post = await prisma.post.findUnique({ where: { id: params.id } })

  if (!post) {
    notFound()  // Triggers not-found.tsx (correct!)
  }

  return <div>{post.title}</div>
}
```

### ❌ Don't Ignore Prisma Errors

```typescript
// ❌ BAD - Generic error handling
export async function createUser(email: string) {
  try {
    return await prisma.user.create({ data: { email } })
  } catch (error) {
    return { success: false, error: 'Failed' }  // No detail!
  }
}

// ✅ GOOD - Handle specific Prisma errors
export async function createUser(email: string) {
  try {
    return await prisma.user.create({ data: { email } })
  } catch (error) {
    if (error instanceof Prisma.PrismaClientKnownRequestError) {
      if (error.code === 'P2002') {
        return { success: false, error: 'Email already exists' }
      }
    }
    return { success: false, error: 'Failed to create user' }
  }
}
```

---

## Error Handling Decision Tree

### Should I throw an error or return an error object?

```
Is this a form submission or user-fixable error?
├─ YES → Return error object { success: false, error: '...' }
│         Component can show inline error
│
└─ NO → Is this a catastrophic failure or auth error?
    ├─ YES → Throw error (triggers error.tsx)
    │         Shows error page
    │
    └─ NO → Return error object (safer default)
```

### Which error boundary should catch this?

```
Where should the error be displayed?
├─ Specific route → error.tsx in that route folder
├─ Entire section → error.tsx in parent folder
├─ Whole app → error.tsx in app/ folder
└─ Root/layout error → global-error.tsx
```

### Should I use notFound() or throw an error?

```
Is the resource not found (404)?
├─ YES → Use notFound()
│         Triggers not-found.tsx
│
└─ NO → Is it another HTTP error (401, 403, 500)?
    └─ Throw error or return error object
```

### Should I log this error?

```
Is this error expected/operational?
├─ YES (validation, not found) → Log at info/warn level
│                                 User-fixable
│
└─ NO (bug, DB down) → Log at error level
                       Send to monitoring
                       Needs investigation
```

### Should I catch and handle or propagate?

```
Can I handle this error meaningfully at this layer?
├─ YES → Catch and handle (show user message, retry, etc.)
│
└─ NO → Propagate up (throw or return error object)
         Let higher layer decide
```

---

**Related Files:**
- [database-patterns.md](database-patterns.md) - Prisma error handling, transaction errors
- [validation-patterns.md](validation-patterns.md) - Zod validation errors
- [middleware-guide.md](middleware-guide.md) - Auth error handling
- [architecture-overview.md](architecture-overview.md) - Overall error flow in Two-Path Architecture
