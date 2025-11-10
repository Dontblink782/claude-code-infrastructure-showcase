---
name: backend-dev-guidelines
description: Complete backend development guide for Next.js + Supabase + Prisma applications. Use when creating server actions, server components, implementing RLS policies, building complex business logic with Prisma, auth with Supabase, Zod validation, or async error patterns. Covers two-path architecture (RLS vs Server Actions), when to use each path, helper functions, transactions, and migration from Express patterns.
---

# Backend Development Guidelines

## Purpose

Establish consistency and best practices for Next.js + Supabase + Prisma applications using the two-path architecture pattern.

## When to Use This Skill

Automatically activates when working on:
- Creating or modifying server actions, server components
- Implementing RLS policies for Supabase
- Complex business logic with Prisma transactions
- Supabase authentication and authorization
- Input validation with Zod
- Database queries and optimization
- Error handling patterns
- Backend testing and refactoring

---

## Quick Start

### New Backend Feature Checklist

- [ ] **Decision**: RLS or Server Action?
- [ ] **Auth**: getCurrentUser() check
- [ ] **Validation**: Zod schema
- [ ] **Logic**: Direct or extract to lib/?
- [ ] **Revalidation**: revalidatePath/revalidateTag
- [ ] **Error Handling**: Return objects or throw?
- [ ] **Tests**: Unit + integration tests

### RLS-Only Feature Checklist

- [ ] RLS policy (SELECT/INSERT/UPDATE/DELETE)
- [ ] Helper functions (if complex logic)
- [ ] Client hook (for mutations)
- [ ] Server action (for SSR data fetching)
- [ ] Validation (Zod schema)

### Server Action Feature Checklist

- [ ] Server action in actions/
- [ ] Auth check (getCurrentUser)
- [ ] Validation (Zod schema)
- [ ] Business logic (inline or lib/)
- [ ] Prisma queries or transactions
- [ ] Revalidation
- [ ] Error handling

---

## Architecture Overview

### Two-Path Architecture

```
PATH 1: Simple CRUD (RLS)
Client → Supabase RLS → Database

PATH 2: Complex Logic (Server Actions + Prisma)
Client → Server Action → Prisma → Database
```

**Key Decision:** Simple CRUD? → RLS. Complex logic? → Server Action + Prisma.

See [architecture-overview.md](architecture-overview.md) for complete details.

---

## Directory Structure

```
app/                           # Next.js App Router
├── (auth)/                    # Route groups
├── dashboard/
│   └── page.tsx
└── api/                       # API routes (webhooks only)

actions/                       # Server actions (flat)
├── posts.ts                   # Post actions
├── staff.ts                   # Staff actions
└── billing.ts                 # Billing actions

lib/
├── db/                        # Prisma helpers (2x+ reuse)
│   └── posts.ts
├── validations/               # Zod schemas
│   └── post.ts
├── stripe/                    # Third-party integrations
└── email/                     # Email helpers

utils/supabase/                # Supabase clients
├── client.ts                  # Browser client
├── server.ts                  # Server client
└── middleware.ts              # Token refresh
```

**Key Pattern:** Keep logic in `actions/` until used 2+ times, then extract to `lib/`.

---

## Core Principles (7 Key Rules)

### 1. Choose the Right Path

```typescript
// ✅ Simple CRUD → RLS
const { data } = await supabase.from('posts').insert({ title, content })

// ✅ Complex logic → Server Action + Prisma
export async function addStaffMember(formData: FormData) {
    const result = await prisma.$transaction(...)
}
```

### 2. All Server Actions Must Check Auth

```typescript
// ❌ NEVER: No auth check
export async function createPost(formData: FormData) {
    await prisma.post.create(...)
}

// ✅ ALWAYS: Check auth first
export async function createPost(formData: FormData) {
    const user = await getCurrentUser()
    if (!user) return { success: false, error: 'Unauthorized' }
    // ...
}
```

### 3. Validate All Input with Zod

```typescript
const schema = z.object({ email: z.string().email() })
const result = schema.safeParse(data)
if (!result.success) return { success: false, error: 'Invalid input' }
```

### 4. Extract to lib/ Only When Reused 2+ Times

```typescript
// ❌ NEVER: Thin wrappers
export async function getPost(id: string) {
    return prisma.post.findUnique({ where: { id } })
}

// ✅ ALWAYS: Direct in action until reused
export async function getPost(id: string) {
    const post = await prisma.post.findUnique({
        where: { id },
        include: { author: true }
    })
    return post
}
```

### 5. Revalidate After Mutations

```typescript
await prisma.post.create({ data })
revalidatePath('/posts')  // ← Required!
```

### 6. Return Error Objects for UX

```typescript
try {
    const post = await prisma.post.create(...)
    return { success: true, data: post }
} catch (error) {
    return { success: false, error: 'Failed to create post' }
}
```

### 7. Use Transactions for Multi-Step Operations

```typescript
await prisma.$transaction(async (tx) => {
    const user = await tx.user.create({ data: { email } })
    await tx.organizationMember.create({ data: { userId: user.id } })
    await tx.auditLog.create({ data: { action: 'USER_ADDED' } })
})
```

---

## Common Imports

```typescript
// Next.js
import { revalidatePath, revalidateTag } from 'next/cache'
import { redirect, notFound } from 'next/navigation'

// Validation
import { z } from 'zod'

// Database
import { prisma } from '@/lib/prisma'
import type { Prisma } from '@prisma/client'

// Supabase
import { createServerClient } from '@/utils/supabase/server'
import { createBrowserClient } from '@/utils/supabase/client'
import { getCurrentUser } from '@/utils/supabase/server'

// Auth helpers
import { requireOrganizationRole } from '@/lib/auth-helpers'
```

---

## Quick Reference

### When to Use RLS vs Server Actions

| Use Case | Path | Why |
|----------|------|-----|
| List posts | RLS | Simple read |
| Update profile | RLS | User owns resource |
| Add staff member | Server Action | Complex auth + transaction |
| Subscribe to plan | Server Action | Third-party (Stripe) |
| Real-time chat | RLS | Needs subscriptions |

### HTTP Status Codes

| Code | Use Case |
|------|----------|
| 200 | Success |
| 201 | Created |
| 400 | Bad Request |
| 401 | Unauthorized |
| 403 | Forbidden |
| 404 | Not Found |
| 500 | Server Error |

---

## Anti-Patterns to Avoid

❌ No auth check in server actions
❌ Missing input validation
❌ Thin wrapper functions in lib/
❌ Using Prisma in middleware
❌ Forgetting revalidatePath
❌ Exposing sensitive errors to client

---

## Resource Files

| Need to... | Read this |
|------------|-----------|
| Understand two-path architecture | [architecture-overview.md](architecture-overview.md) - RLS vs Server Actions, when to use each |
| Create server actions | [routing-and-controllers.md](routing-and-controllers.md) - Server actions, helper functions, patterns |
| Extract reusable logic | [services-and-repositories.md](services-and-repositories.md) - When to extract to lib/, services pattern |
| Validate input | [validation-patterns.md](validation-patterns.md) - Zod schemas, FormData parsing, error handling |
| Handle auth/middleware | [middleware-guide.md](middleware-guide.md) - Supabase auth, token refresh, route protection |
| Query with Prisma | [database-patterns.md](database-patterns.md) - Transactions, parallel queries, N+1 prevention |
| Handle async/errors | [async-and-errors.md](async-and-errors.md) - Error boundaries, custom errors, Prisma errors |
| Write tests | [testing-guide.md](testing-guide.md) - Unit/integration tests, mocking |
| See full examples | [complete-examples.md](complete-examples.md) - Complete patterns, refactoring guide |

---

**Skill Status**: COMPLETE ✅
**Line Count**: < 300 ✅
**Progressive Disclosure**: 9 resource files ✅
