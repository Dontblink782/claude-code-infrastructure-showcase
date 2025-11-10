# Database Patterns - Prisma in Next.js Server Actions

Complete guide to using Prisma for complex business logic in Next.js server actions. Covers when to use Prisma vs Supabase RLS, query patterns, transactions, and optimization.

## Table of Contents

- [Prisma vs Supabase RLS](#prisma-vs-supabase-rls)
- [Prisma Client Setup](#prisma-client-setup)
- [Where to Put Queries](#where-to-put-queries)
- [Transaction Patterns](#transaction-patterns)
- [Parallel Queries](#parallel-queries)
- [Query Optimization](#query-optimization)
- [N+1 Query Prevention](#n1-query-prevention)
- [Error Handling](#error-handling)

---

## Prisma vs Supabase RLS

### Decision Tree

**Use Supabase RLS when:**
- ✅ Simple CRUD operations (list posts, update profile)
- ✅ Authorization is row-level (user owns resource)
- ✅ Client-side reads needed
- ✅ Real-time subscriptions needed
- ✅ No complex business logic

**Use Prisma when:**
- ✅ Complex business logic (multitenant staff, subscriptions)
- ✅ Multiple related operations (transactions)
- ✅ Third-party integrations (Stripe, email)
- ✅ Complex authorization (roles, team permissions)
- ✅ Server-side only operations
- ✅ Need full TypeScript type safety

**Both can coexist:**
- RLS for simple reads from client
- Prisma for complex writes in server actions
- See [architecture-overview.md](architecture-overview.md) for examples

---

## Prisma Client Setup

### Standard Pattern

```typescript
// lib/prisma.ts
import { PrismaClient } from '@prisma/client'

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined
}

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === 'development' ? ['query', 'error', 'warn'] : ['error'],
  })

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma
```

**Why this pattern?**
- Prevents multiple Prisma instances in development (hot reload)
- Single client instance across app
- Logs queries in development
- Production-ready

### Basic Usage

```typescript
// actions/posts.ts
'use server'
import { prisma } from '@/lib/prisma'

export async function getPosts() {
  return await prisma.post.findMany({
    orderBy: { createdAt: 'desc' }
  })
}
```

---

## Where to Put Queries

### Decision Guide

**Put directly in server actions (`actions/*.ts`) when:**
- ✅ Single use case
- ✅ Simple query (<10 lines)
- ✅ Action-specific logic

```typescript
// actions/posts.ts
'use server'
import { prisma } from '@/lib/prisma'
import { getCurrentUser } from '@/utils/supabase/server'

export async function createPost(formData: FormData) {
  const user = await getCurrentUser()
  if (!user) throw new Error('Unauthorized')

  return await prisma.post.create({
    data: {
      title: formData.get('title') as string,
      content: formData.get('content') as string,
      userId: user.id
    }
  })
}
```

**Extract to helpers (`lib/db/*.ts`) when:**
- ✅ Used in 2+ actions
- ✅ Complex logic (20+ lines)
- ✅ Needs testing isolation
- ✅ Reusable query builder

```typescript
// lib/db/posts.ts
import { prisma } from '@/lib/prisma'

export async function getPostsByUser(userId: string) {
  return await prisma.post.findMany({
    where: { userId },
    include: {
      comments: {
        take: 5,
        orderBy: { createdAt: 'desc' }
      },
      _count: { select: { likes: true } }
    },
    orderBy: { createdAt: 'desc' }
  })
}

// actions/posts.ts
'use server'
import { getPostsByUser } from '@/lib/db/posts'

export async function getUserPosts(userId: string) {
  return await getPostsByUser(userId)
}
```

**❌ Avoid thin wrappers:**
```typescript
// BAD - pointless abstraction
export async function findPostById(id: string) {
  return prisma.post.findUnique({ where: { id } })
}

// GOOD - use prisma directly in action
export async function getPost(id: string) {
  return await prisma.post.findUnique({
    where: { id },
    include: { author: true, comments: true }
  })
}
```

---

## Transaction Patterns

Transactions ensure multiple operations succeed or fail together. Critical for complex business logic.

### Simple Transaction in Server Action

```typescript
// actions/staff.ts
'use server'
import { prisma } from '@/lib/prisma'
import { getCurrentUser } from '@/utils/supabase/server'
import { revalidatePath } from 'next/cache'

export async function addStaffMember(formData: FormData) {
  const user = await getCurrentUser()
  if (!user) throw new Error('Unauthorized')

  const email = formData.get('email') as string
  const organizationId = formData.get('organizationId') as string

  // Transaction: Create user + add to org + log action
  const result = await prisma.$transaction(async (tx) => {
    // Check if user exists
    let staffUser = await tx.user.findUnique({
      where: { email }
    })

    // Create if doesn't exist
    if (!staffUser) {
      staffUser = await tx.user.create({
        data: { email }
      })
    }

    // Add to organization
    const member = await tx.organizationMember.create({
      data: {
        userId: staffUser.id,
        organizationId,
        role: 'MEMBER',
        invitedBy: user.id
      }
    })

    // Audit log
    await tx.auditLog.create({
      data: {
        action: 'STAFF_ADDED',
        userId: user.id,
        organizationId,
        metadata: { staffUserId: staffUser.id }
      }
    })

    return member
  })

  revalidatePath(`/dashboard/${organizationId}/team`)
  return result
}
```

### Transaction with Business Rules

```typescript
// actions/billing.ts
'use server'
import { prisma } from '@/lib/prisma'
import { getCurrentUser } from '@/utils/supabase/server'

export async function upgradePlan(organizationId: string, newPlan: string) {
  const user = await getCurrentUser()
  if (!user) throw new Error('Unauthorized')

  return await prisma.$transaction(async (tx) => {
    // 1. Check authorization
    const membership = await tx.organizationMember.findFirst({
      where: {
        userId: user.id,
        organizationId,
        role: 'ADMIN'
      }
    })
    if (!membership) throw new Error('Only admins can upgrade')

    // 2. Check current plan
    const org = await tx.organization.findUnique({
      where: { id: organizationId },
      include: { subscription: true }
    })
    if (!org) throw new Error('Organization not found')
    if (org.subscription?.plan === newPlan) {
      throw new Error('Already on this plan')
    }

    // 3. Update subscription
    const subscription = await tx.subscription.upsert({
      where: { organizationId },
      update: {
        plan: newPlan,
        updatedAt: new Date()
      },
      create: {
        organizationId,
        plan: newPlan
      }
    })

    // 4. Log the change
    await tx.auditLog.create({
      data: {
        action: 'PLAN_UPGRADED',
        userId: user.id,
        organizationId,
        metadata: {
          oldPlan: org.subscription?.plan,
          newPlan
        }
      }
    })

    return subscription
  }, {
    maxWait: 5000,  // Max time to wait for transaction slot
    timeout: 10000  // Max time transaction can run
  })
}
```

**When to use transactions:**
- Multiple related operations that must succeed/fail together
- Creating records with relationships
- Business logic requiring multiple checks and updates
- Audit logs that must stay in sync with changes

**When NOT to use transactions:**
- Single operation (just use regular Prisma call)
- Independent operations (use parallel queries instead)
- Long-running operations (transactions lock database)

---

## Parallel Queries

Avoid request waterfalls by running independent queries in parallel.

### Using Promise.all in Server Actions

```typescript
// actions/dashboard.ts
'use server'
import { prisma } from '@/lib/prisma'
import { getCurrentUser } from '@/utils/supabase/server'

export async function getDashboardData(organizationId: string) {
  const user = await getCurrentUser()
  if (!user) throw new Error('Unauthorized')

  // ❌ BAD - Sequential (slow)
  // const members = await prisma.organizationMember.findMany(...)
  // const projects = await prisma.project.findMany(...)
  // const tasks = await prisma.task.findMany(...)

  // ✅ GOOD - Parallel (fast)
  const [members, projects, tasks, stats] = await Promise.all([
    prisma.organizationMember.findMany({
      where: { organizationId },
      include: { user: true }
    }),
    prisma.project.findMany({
      where: { organizationId },
      take: 10,
      orderBy: { updatedAt: 'desc' }
    }),
    prisma.task.findMany({
      where: {
        project: { organizationId },
        status: 'IN_PROGRESS'
      },
      take: 20
    }),
    prisma.organization.findUnique({
      where: { id: organizationId },
      select: {
        _count: {
          select: {
            members: true,
            projects: true
          }
        }
      }
    })
  ])

  return { members, projects, tasks, stats }
}
```

**Total time:**
- Sequential: members (100ms) + projects (150ms) + tasks (120ms) = 370ms
- Parallel: max(100ms, 150ms, 120ms) = 150ms

### When to Use Promise.all

✅ **Use when:**
- Queries are independent
- No query depends on another's result
- Fetching data for dashboard/page
- Multiple unrelated resources

❌ **Don't use when:**
- Query B needs result from Query A
- Transaction is more appropriate
- Operations must be sequential for business logic

---

## Query Optimization

### Use select to Limit Fields

Fetching only needed fields reduces memory usage and speeds up queries.

```typescript
// actions/users.ts
'use server'
import { prisma } from '@/lib/prisma'

// ❌ BAD - Fetches all fields (including password hash, etc)
export async function getUsers() {
  return await prisma.user.findMany()
}

// ✅ GOOD - Only fetch fields you need
export async function getUsers() {
  return await prisma.user.findMany({
    select: {
      id: true,
      email: true,
      name: true,
      createdAt: true
    }
  })
}

// ✅ BETTER - With nested select
export async function getUsersWithProfiles() {
  return await prisma.user.findMany({
    select: {
      id: true,
      email: true,
      profile: {
        select: {
          firstName: true,
          lastName: true,
          avatar: true
        }
      }
    }
  })
}
```

### Use include Carefully

Only include relations you actually need. Nested includes can cause massive queries.

```typescript
// ❌ BAD - Excessive includes (slow, memory intensive)
export async function getUser(id: string) {
  return await prisma.user.findUnique({
    where: { id },
    include: {
      profile: true,
      posts: {
        include: {
          comments: {
            include: {
              author: true,
              replies: true
            }
          },
          likes: true
        }
      },
      organizations: {
        include: {
          members: true,
          projects: true
        }
      }
    }
  })
}

// ✅ GOOD - Only include what's needed for this use case
export async function getUserProfile(id: string) {
  return await prisma.user.findUnique({
    where: { id },
    include: {
      profile: true,
      _count: {
        select: {
          posts: true,
          organizations: true
        }
      }
    }
  })
}

// ✅ Create separate actions for different use cases
export async function getUserPosts(id: string) {
  return await prisma.post.findMany({
    where: { userId: id },
    take: 20,
    orderBy: { createdAt: 'desc' },
    select: {
      id: true,
      title: true,
      createdAt: true,
      _count: { select: { comments: true, likes: true } }
    }
  })
}
```

### Use Pagination

Always paginate large result sets.

```typescript
// actions/posts.ts
'use server'
import { prisma } from '@/lib/prisma'

export async function getPosts(page: number = 1, perPage: number = 20) {
  const skip = (page - 1) * perPage

  const [posts, total] = await Promise.all([
    prisma.post.findMany({
      skip,
      take: perPage,
      orderBy: { createdAt: 'desc' },
      select: {
        id: true,
        title: true,
        excerpt: true,
        createdAt: true,
        author: {
          select: { name: true, avatar: true }
        }
      }
    }),
    prisma.post.count()
  ])

  return {
    posts,
    pagination: {
      page,
      perPage,
      total,
      totalPages: Math.ceil(total / perPage)
    }
  }
}
```

---

## N+1 Query Prevention

N+1 happens when you fetch a list, then loop through it making queries for each item. This creates 1 + N database queries instead of 1 or 2.

### Problem: N+1 Queries

```typescript
// actions/posts.ts
'use server'
import { prisma } from '@/lib/prisma'

// ❌ BAD - N+1 Query Problem
export async function getPostsWithAuthors() {
  const posts = await prisma.post.findMany() // 1 query

  // N queries (one per post) - VERY SLOW!
  const postsWithAuthors = await Promise.all(
    posts.map(async (post) => {
      const author = await prisma.user.findUnique({
        where: { id: post.userId }
      })
      return { ...post, author }
    })
  )

  return postsWithAuthors
}
```

**Performance:**
- 100 posts = 101 database queries
- With network latency: 100 * 10ms = 1000ms+ just for queries

### Solution 1: Use include

Best for simple cases where you need the full related record.

```typescript
// ✅ GOOD - Single query with include
export async function getPostsWithAuthors() {
  return await prisma.post.findMany({
    include: {
      author: {
        select: {
          id: true,
          name: true,
          avatar: true
        }
      }
    }
  })
}
```

**Performance:** 1 database query total

### Solution 2: Batch Query

Best when you need more control or when include isn't possible.

```typescript
// ✅ GOOD - Batch query
export async function getPostsWithAuthors() {
  const posts = await prisma.post.findMany()

  // Get unique user IDs
  const userIds = [...new Set(posts.map(p => p.userId))]

  // Single query for all authors
  const users = await prisma.user.findMany({
    where: { id: { in: userIds } },
    select: { id: true, name: true, avatar: true }
  })

  // Map in memory (fast)
  const userMap = new Map(users.map(u => [u.id, u]))

  return posts.map(post => ({
    ...post,
    author: userMap.get(post.userId)
  }))
}
```

**Performance:** 2 database queries total (regardless of post count)

### Real-World Example: Dashboard

```typescript
// actions/dashboard.ts
'use server'
import { prisma } from '@/lib/prisma'

// ❌ BAD - Multiple N+1 problems
export async function getDashboard(organizationId: string) {
  const projects = await prisma.project.findMany({
    where: { organizationId }
  })

  // N+1 #1
  const projectsWithTasks = await Promise.all(
    projects.map(async (project) => {
      const tasks = await prisma.task.findMany({
        where: { projectId: project.id }
      })
      return { ...project, tasks }
    })
  )

  // N+1 #2
  const projectsWithMembers = await Promise.all(
    projectsWithTasks.map(async (project) => {
      const members = await prisma.projectMember.findMany({
        where: { projectId: project.id }
      })
      return { ...project, members }
    })
  )

  return projectsWithMembers
}

// ✅ GOOD - Single query with includes
export async function getDashboard(organizationId: string) {
  return await prisma.project.findMany({
    where: { organizationId },
    include: {
      tasks: {
        where: { status: 'IN_PROGRESS' },
        take: 5
      },
      members: {
        include: {
          user: {
            select: { name: true, avatar: true }
          }
        }
      },
      _count: {
        select: { tasks: true }
      }
    }
  })
}
```

---

## Error Handling

Handle Prisma errors gracefully in server actions to provide good user experience.

### Prisma Error Types

```typescript
// actions/users.ts
'use server'
import { prisma } from '@/lib/prisma'
import { Prisma } from '@prisma/client'

export async function createUser(email: string, name: string) {
  try {
    return await prisma.user.create({
      data: { email, name }
    })
  } catch (error) {
    if (error instanceof Prisma.PrismaClientKnownRequestError) {
      // Unique constraint violation
      if (error.code === 'P2002') {
        const field = error.meta?.target as string[]
        throw new Error(`${field?.[0] || 'Field'} already exists`)
      }

      // Foreign key constraint
      if (error.code === 'P2003') {
        throw new Error('Invalid reference - related record not found')
      }

      // Record not found (for update/delete)
      if (error.code === 'P2025') {
        throw new Error('Record not found')
      }
    }

    // Log unknown errors
    console.error('Database error:', error)
    throw new Error('An unexpected error occurred')
  }
}
```

### Common Prisma Error Codes

| Code | Meaning | Example |
|------|---------|---------|
| P2002 | Unique constraint failed | Email already exists |
| P2003 | Foreign key constraint failed | Invalid organization ID |
| P2025 | Record not found | User to update doesn't exist |
| P2014 | Required relation violation | Can't delete user with posts |
| P2034 | Transaction conflict | Concurrent update conflict |

### Error Handling in Transactions

```typescript
// actions/staff.ts
'use server'
import { prisma } from '@/lib/prisma'
import { Prisma } from '@prisma/client'
import { getCurrentUser } from '@/utils/supabase/server'

export async function addStaffMember(email: string, orgId: string) {
  const user = await getCurrentUser()
  if (!user) throw new Error('Unauthorized')

  try {
    return await prisma.$transaction(async (tx) => {
      // Business rule check
      const org = await tx.organization.findUnique({
        where: { id: orgId },
        include: { _count: { select: { members: true } } }
      })

      if (!org) {
        throw new Error('Organization not found')
      }

      if (org._count.members >= org.maxMembers) {
        throw new Error('Staff limit reached. Upgrade your plan.')
      }

      // Create staff member
      return await tx.organizationMember.create({
        data: {
          email,
          organizationId: orgId,
          invitedBy: user.id
        }
      })
    })
  } catch (error) {
    // Business logic errors
    if (error instanceof Error && error.message.includes('limit reached')) {
      throw error // Pass through with user-friendly message
    }

    // Prisma errors
    if (error instanceof Prisma.PrismaClientKnownRequestError) {
      if (error.code === 'P2002') {
        throw new Error('This user is already a member')
      }
    }

    console.error('Transaction error:', error)
    throw new Error('Failed to add staff member')
  }
}
```

### Handling Errors in Components

```typescript
// components/CreateUserForm.tsx
'use client'
import { createUser } from '@/actions/users'
import { useState } from 'react'

export function CreateUserForm() {
  const [error, setError] = useState<string | null>(null)

  async function handleSubmit(formData: FormData) {
    try {
      setError(null)
      await createUser(
        formData.get('email') as string,
        formData.get('name') as string
      )
      // Success - redirect or show message
    } catch (err) {
      // Display error to user
      setError(err instanceof Error ? err.message : 'An error occurred')
    }
  }

  return (
    <form action={handleSubmit}>
      {error && <div className="error">{error}</div>}
      {/* form fields */}
    </form>
  )
}
```

---

**Related Files:**
- [architecture-overview.md](architecture-overview.md) - Two-path architecture
- [validation-patterns.md](validation-patterns.md) - Zod validation
- [async-and-errors.md](async-and-errors.md) - Error handling patterns
