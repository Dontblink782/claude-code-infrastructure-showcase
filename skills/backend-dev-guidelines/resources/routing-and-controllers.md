# Server Actions - Next.js Pattern Guide

Complete guide to clean server action patterns for Next.js applications.

## Table of Contents

- [The Next.js Approach](#the-nextjs-approach)
- [Server Actions: The Golden Rule](#server-actions-the-golden-rule)
- [Good Examples](#good-examples)
- [Anti-Patterns](#anti-patterns)
- [Error Handling](#error-handling)
- [Refactoring Guide](#refactoring-guide)

---

## The Next.js Approach

### No Traditional API Routes

**In Next.js + Supabase + Prisma:**
- ❌ No Express routes
- ❌ No controller classes
- ❌ No REST endpoints
- ✅ Server actions for mutations
- ✅ Server components for queries
- ✅ RLS or server actions depending on complexity

**Why this is simpler:**
- Direct function calls from client → server
- No HTTP layer to manage
- Type-safe end-to-end
- Automatic serialization
- Built-in revalidation

---

## Server Actions: The Golden Rule

### What Server Actions Should Be

**Server Actions should:**
- ✅ Handle ONE operation clearly (create, update, delete)
- ✅ Validate input with Zod schemas
- ✅ Get auth context (getCurrentUser)
- ✅ Call services/Prisma for complex logic
- ✅ Return error objects (not throw for UX errors)
- ✅ Call revalidatePath/revalidateTag when needed

**Server Actions should NEVER:**
- ❌ Contain 100+ lines of business logic
- ❌ Mix multiple unrelated operations
- ❌ Skip auth checks
- ❌ Return raw Prisma errors to client
- ❌ Forget to revalidate after mutations

### Clean Server Action Pattern

```typescript
// actions/posts.ts
'use server'
import { getCurrentUser } from '@/utils/supabase/server'
import { prisma } from '@/lib/prisma'
import { createPostSchema } from '@/lib/validations/post'
import { revalidatePath } from 'next/cache'

// ✅ CLEAN: Single responsibility, clear flow
export async function createPost(formData: FormData) {
  // 1. Auth
  const user = await getCurrentUser()
  if (!user) {
    return { success: false, error: 'Unauthorized' }
  }

  // 2. Validate
  const result = createPostSchema.safeParse({
    title: formData.get('title'),
    content: formData.get('content')
  })

  if (!result.success) {
    return { success: false, error: 'Invalid input' }
  }

  // 3. Execute (simple case - direct Prisma)
  try {
    const post = await prisma.post.create({
      data: {
        ...result.data,
        userId: user.id
      }
    })

    // 4. Revalidate
    revalidatePath('/posts')

    return { success: true, data: post }
  } catch (error) {
    return {
      success: false,
      error: 'Failed to create post'
    }
  }
}
```

**Key Points:**
- One operation per action
- Always check auth
- Validate with Zod
- Return error objects for UX
- Revalidate after mutations
- Keep simple logic inline

---

## Good Examples

### Example 1: Simple CRUD (Excellent ✅)

```typescript
// actions/posts.ts
'use server'
import { getCurrentUser } from '@/utils/supabase/server'
import { prisma } from '@/lib/prisma'
import { revalidatePath } from 'next/cache'
import { updatePostSchema } from '@/lib/validations/post'

export async function updatePost(postId: string, formData: FormData) {
  const user = await getCurrentUser()
  if (!user) {
    return { success: false, error: 'Unauthorized' }
  }

  const result = updatePostSchema.safeParse({
    title: formData.get('title'),
    content: formData.get('content')
  })

  if (!result.success) {
    return { success: false, error: 'Invalid input' }
  }

  try {
    // Check ownership
    const post = await prisma.post.findUnique({
      where: { id: postId }
    })

    if (!post || post.userId !== user.id) {
      return { success: false, error: 'Post not found or no permission' }
    }

    // Update
    const updated = await prisma.post.update({
      where: { id: postId },
      data: result.data
    })

    revalidatePath(`/posts/${postId}`)
    revalidatePath('/posts')

    return { success: true, data: updated }
  } catch (error) {
    return { success: false, error: 'Failed to update post' }
  }
}

export async function deletePost(postId: string) {
  const user = await getCurrentUser()
  if (!user) {
    return { success: false, error: 'Unauthorized' }
  }

  try {
    const post = await prisma.post.findUnique({
      where: { id: postId }
    })

    if (!post || post.userId !== user.id) {
      return { success: false, error: 'Not found or no permission' }
    }

    await prisma.post.delete({
      where: { id: postId }
    })

    revalidatePath('/posts')

    return { success: true }
  } catch (error) {
    return { success: false, error: 'Failed to delete post' }
  }
}
```

**What Makes This Excellent:**
- Clear, single-responsibility actions
- Consistent auth pattern
- Ownership checks
- Error objects for UX
- Proper revalidation

### Example 2: Complex Logic - Extract to Service (Good ✅)

```typescript
// actions/staff.ts
'use server'
import { getCurrentUser } from '@/utils/supabase/server'
import { addStaffSchema } from '@/lib/validations/staff'
import { StaffService } from '@/lib/services/staffService'
import { revalidatePath } from 'next/cache'

// ✅ GOOD: Action delegates to service for complex logic
export async function addStaffMember(formData: FormData) {
  const user = await getCurrentUser()
  if (!user) {
    return { success: false, error: 'Unauthorized' }
  }

  const result = addStaffSchema.safeParse({
    email: formData.get('email'),
    role: formData.get('role'),
    organizationId: formData.get('organizationId')
  })

  if (!result.success) {
    return { success: false, error: 'Invalid input' }
  }

  try {
    // Delegate complex logic to service
    const staffService = new StaffService()
    const member = await staffService.addMember(
      result.data,
      user.id
    )

    revalidatePath(`/dashboard/${result.data.organizationId}/team`)

    return { success: true, data: member }
  } catch (error) {
    return {
      success: false,
      error: error instanceof Error ? error.message : 'Failed to add staff'
    }
  }
}
```

**Service handles complexity:**

```typescript
// lib/services/staffService.ts
import { prisma } from '@/lib/prisma'

export class StaffService {
  async addMember(
    data: { email: string; role: string; organizationId: string },
    requestingUserId: string
  ) {
    // Complex authorization
    const membership = await prisma.organizationMember.findFirst({
      where: {
        userId: requestingUserId,
        organizationId: data.organizationId,
        role: 'ADMIN'
      }
    })

    if (!membership) {
      throw new Error('Only admins can add staff')
    }

    // Business rules
    const org = await prisma.organization.findUnique({
      where: { id: data.organizationId },
      include: { _count: { select: { members: true } } }
    })

    if (!org) {
      throw new Error('Organization not found')
    }

    if (org._count.members >= org.maxMembers) {
      throw new Error('Staff limit reached')
    }

    // Transaction
    return await prisma.$transaction(async (tx) => {
      let staffUser = await tx.user.findUnique({
        where: { email: data.email }
      })

      if (!staffUser) {
        staffUser = await tx.user.create({
          data: { email: data.email }
        })
      }

      const member = await tx.organizationMember.create({
        data: {
          userId: staffUser.id,
          organizationId: data.organizationId,
          role: data.role,
          invitedBy: requestingUserId
        }
      })

      await tx.auditLog.create({
        data: {
          action: 'STAFF_ADDED',
          userId: requestingUserId,
          organizationId: data.organizationId,
          metadata: { staffUserId: staffUser.id }
        }
      })

      return member
    })
  }
}
```

**Why This Works:**
- Action stays thin (20 lines)
- Service handles complexity (60 lines)
- Clear separation of concerns
- Easy to test service logic
- Reusable across actions

### Example 3: Using Helper Functions (Good ✅)

```typescript
// actions/projects.ts
'use server'
import { requireOrganizationRole } from '@/lib/auth-helpers'
import { prisma } from '@/lib/prisma'
import { revalidatePath } from 'next/cache'
import { createProjectSchema } from '@/lib/validations/project'

// ✅ GOOD: Reusable auth helper
export async function createProject(
  organizationId: string,
  formData: FormData
) {
  // Helper throws if auth fails
  const { user } = await requireOrganizationRole(
    organizationId,
    ['ADMIN', 'MANAGER']
  )

  const result = createProjectSchema.safeParse({
    name: formData.get('name'),
    description: formData.get('description')
  })

  if (!result.success) {
    return { success: false, error: 'Invalid input' }
  }

  try {
    const project = await prisma.project.create({
      data: {
        ...result.data,
        organizationId,
        createdBy: user.id
      }
    })

    revalidatePath(`/dashboard/${organizationId}/projects`)

    return { success: true, data: project }
  } catch (error) {
    return { success: false, error: 'Failed to create project' }
  }
}
```

**Helper function:**

```typescript
// lib/auth-helpers.ts
import { getCurrentUser } from '@/utils/supabase/server'
import { prisma } from '@/lib/prisma'

export async function requireOrganizationRole(
  organizationId: string,
  allowedRoles: string[]
) {
  const user = await getCurrentUser()
  if (!user) throw new Error('Unauthorized')

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

  if (!allowedRoles.includes(membership.role)) {
    throw new Error(`Must be ${allowedRoles.join(' or ')}`)
  }

  return { user, membership }
}
```

---

## Anti-Patterns

### Anti-Pattern 1: Business Logic Overload (Bad ❌)

```typescript
// ❌ BAD: 200+ lines of logic in server action
'use server'
export async function submitForm(formId: string, formData: FormData) {
  // ❌ Everything inline
  const user = await getCurrentUser()
  const responses = JSON.parse(formData.get('responses') as string)

  // ❌ Complex permission checking
  const canComplete = await prisma.permission.findFirst({
    where: { userId: user.id, formId },
    include: { roles: true }
  })

  if (!canComplete) {
    return { success: false, error: 'No permission' }
  }

  // ❌ Workflow logic inline
  const engine = await createWorkflowEngine()
  const command = new CompleteStepCommand(/* ... */)
  const events = await engine.executeCommand(command)

  // ❌ Impersonation handling
  if (formData.get('isImpersonating')) {
    await impersonationStore.store(/* ... */)
  }

  // ❌ Response processing
  const processed = await processResponses(responses)

  // ... 150+ more lines

  return { success: true, data: result }
}
```

**Why This Is Terrible:**
- 200+ lines in one action
- Hard to test
- Hard to reuse
- Mixed responsibilities
- Difficult to debug

### How to Refactor

**Step 1: Extract to Service**

```typescript
// actions/forms.ts (CLEAN)
'use server'
import { getCurrentUser } from '@/utils/supabase/server'
import { FormSubmissionService } from '@/lib/services/formSubmissionService'
import { submitFormSchema } from '@/lib/validations/form'

export async function submitForm(formId: string, formData: FormData) {
  const user = await getCurrentUser()
  if (!user) {
    return { success: false, error: 'Unauthorized' }
  }

  const result = submitFormSchema.safeParse({
    responses: JSON.parse(formData.get('responses') as string),
    isImpersonating: formData.get('isImpersonating') === 'true'
  })

  if (!result.success) {
    return { success: false, error: 'Invalid input' }
  }

  try {
    const service = new FormSubmissionService()
    const submission = await service.submit(
      formId,
      user.id,
      result.data
    )

    revalidatePath(`/forms/${formId}`)

    return { success: true, data: submission }
  } catch (error) {
    return {
      success: false,
      error: error instanceof Error ? error.message : 'Failed to submit'
    }
  }
}
```

**Step 2: Service Handles Complexity**

```typescript
// lib/services/formSubmissionService.ts
export class FormSubmissionService {
  async submit(
    formId: string,
    userId: string,
    data: { responses: any; isImpersonating: boolean }
  ) {
    // Permission check
    const canComplete = await this.checkPermission(userId, formId)
    if (!canComplete) {
      throw new Error('No permission to submit form')
    }

    // Workflow logic
    const engine = await createWorkflowEngine()
    const command = new CompleteStepCommand(formId, userId, data.responses)
    const events = await engine.executeCommand(command)

    // Impersonation
    if (data.isImpersonating) {
      await this.handleImpersonation(formId, userId)
    }

    // Process responses
    const processed = await this.processResponses(data.responses)

    return { events, processed }
  }

  private async checkPermission(userId: string, formId: string) {
    const permission = await prisma.permission.findFirst({
      where: { userId, formId }
    })
    return !!permission
  }

  private async handleImpersonation(formId: string, userId: string) {
    await impersonationStore.store({ formId, userId })
  }

  private async processResponses(responses: any) {
    // Processing logic
    return responses
  }
}
```

**Result:**
- Action: 30 lines (was 200+)
- Service: 60 lines (testable, reusable)
- Clear separation
- Easy to maintain

---

## Error Handling

### Return Error Objects (Preferred)

```typescript
// ✅ GOOD: Return error objects for UX
export async function createPost(formData: FormData) {
  const user = await getCurrentUser()
  if (!user) {
    // Return error object
    return { success: false, error: 'Unauthorized' }
  }

  try {
    const post = await prisma.post.create({ /* ... */ })
    return { success: true, data: post }
  } catch (error) {
    // Catch and return error
    return { success: false, error: 'Failed to create post' }
  }
}
```

**Client usage:**

```typescript
'use client'
import { createPost } from '@/actions/posts'
import { useState } from 'react'

export function CreatePostForm() {
  const [error, setError] = useState<string | null>(null)

  async function handleSubmit(formData: FormData) {
    const result = await createPost(formData)

    if (!result.success) {
      setError(result.error)
      return
    }

    // Success!
    setError(null)
  }

  return (
    <form action={handleSubmit}>
      {error && <div className="error">{error}</div>}
      {/* form fields */}
    </form>
  )
}
```

### Throw Errors (For Unexpected Issues)

```typescript
// ✅ GOOD: Throw for unexpected errors
export async function createPost(formData: FormData) {
  const user = await getCurrentUser()
  if (!user) {
    // Expected error - return object
    return { success: false, error: 'Unauthorized' }
  }

  const result = createPostSchema.safeParse(/* ... */)
  if (!result.success) {
    // Expected error - return object
    return { success: false, error: 'Invalid input' }
  }

  // Unexpected errors will throw and be caught by error boundary
  const post = await prisma.post.create({
    data: { ...result.data, userId: user.id }
  })

  return { success: true, data: post }
}
```

### Error Boundaries

```typescript
// app/posts/error.tsx
'use client'
import { useEffect } from 'react'

export default function PostsError({
  error,
  reset
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  useEffect(() => {
    console.error('Posts error:', error)
  }, [error])

  return (
    <div>
      <h2>Something went wrong!</h2>
      <button onClick={reset}>Try again</button>
    </div>
  )
}
```

---

## Refactoring Guide

### Identify Actions Needing Refactoring

**Red Flags:**
- Action file > 100 lines
- Multiple operations in one action
- Direct complex Prisma queries (should be in service/helper)
- No auth checks
- Complex business logic inline
- Permission checks duplicated across actions

### When to Extract

**Extract to lib/ when:**
- ✅ Used in 2+ actions
- ✅ Complex logic (20+ lines)
- ✅ Needs testing isolation
- ✅ Permission checking (reusable)
- ✅ Third-party integration

**Keep in action when:**
- ✅ Single use
- ✅ Simple (< 20 lines)
- ✅ Action-specific

### Refactoring Process

**1. Simple → Keep Inline:**

```typescript
// ✅ GOOD: Simple logic stays in action
export async function updateProfile(formData: FormData) {
  const user = await getCurrentUser()
  if (!user) return { success: false, error: 'Unauthorized' }

  const result = profileSchema.safeParse({
    name: formData.get('name'),
    bio: formData.get('bio')
  })

  if (!result.success) {
    return { success: false, error: 'Invalid input' }
  }

  try {
    const updated = await prisma.user.update({
      where: { id: user.id },
      data: result.data
    })

    revalidatePath('/profile')

    return { success: true, data: updated }
  } catch (error) {
    return { success: false, error: 'Failed to update' }
  }
}
```

**2. Complex → Extract to Helper:**

```typescript
// actions/projects.ts
import { requireOrganizationRole } from '@/lib/auth-helpers'

export async function createProject(orgId: string, formData: FormData) {
  // Reusable helper
  await requireOrganizationRole(orgId, ['ADMIN', 'MANAGER'])

  // Rest of action...
}
```

**3. Very Complex → Extract to Service:**

```typescript
// actions/billing.ts
import { BillingService } from '@/lib/services/billingService'

export async function createSubscription(formData: FormData) {
  const user = await getCurrentUser()
  if (!user) return { success: false, error: 'Unauthorized' }

  try {
    const billingService = new BillingService()
    const subscription = await billingService.createSubscription(
      user.id,
      formData.get('planId') as string
    )

    return { success: true, data: subscription }
  } catch (error) {
    return { success: false, error: error.message }
  }
}
```

---

## Decision Tree

```
Is this a simple CRUD operation?
├─ Yes → Use RLS (Supabase client from server action)
└─ No → Does it need complex logic?
    ├─ No → Server action with inline Prisma
    └─ Yes → Server action + Service/Helper
        ├─ Used 2+ times? → Extract to lib/
        └─ Single use? → Keep in action but organized
```

---

**Related Files:**
- [architecture-overview.md](architecture-overview.md) - Two-path architecture
- [services-and-repositories.md](services-and-repositories.md) - When to extract
- [complete-examples.md](complete-examples.md) - Full examples
- [async-and-errors.md](async-and-errors.md) - Error handling patterns
