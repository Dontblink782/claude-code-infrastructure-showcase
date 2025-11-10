# Services Pattern - Next.js Server Actions

Simple guide to when and how to extract reusable logic from Next.js server actions.

## Table of Contents

- [The Simple Rule](#the-simple-rule)
- [When to Extract to lib/](#when-to-extract-to-lib)
- [Service Pattern Examples](#service-pattern-examples)
- [Helper Functions vs Services](#helper-functions-vs-services)
- [Testing Extracted Logic](#testing-extracted-logic)

---

## The Simple Rule

**Default: Keep logic in server actions**

Put business logic directly in server actions (`actions/*.ts`) until you have a reason to extract it.

**Extract to helpers (`lib/`) only when:**
- ✅ Used in 2+ actions
- ✅ Complex logic (20+ lines)
- ✅ Needs testing isolation
- ✅ Third-party integration (Stripe, email)

**What this is NOT:**
- ❌ No dependency injection containers
- ❌ No repository layer (use Prisma directly)
- ❌ No service layer unless needed
- ❌ No over-engineering

**Decision tree:**
```
Is this used in multiple actions?
├─ No → Keep in action
└─ Yes → Is it complex (20+ lines)?
    ├─ No → Keep in action (DRY isn't always better)
    └─ Yes → Extract to lib/
```

---

## When to Extract to lib/

### Example 1: Simple - Keep in Action

**Scenario:** Creating a blog post

```typescript
// actions/posts.ts
'use server'
import { getCurrentUser } from '@/utils/supabase/server'
import { prisma } from '@/lib/prisma'
import { revalidatePath } from 'next/cache'

// ✅ GOOD - Logic directly in action
export async function createPost(formData: FormData) {
  const user = await getCurrentUser()
  if (!user) throw new Error('Unauthorized')

  const post = await prisma.post.create({
    data: {
      title: formData.get('title') as string,
      content: formData.get('content') as string,
      userId: user.id
    }
  })

  revalidatePath('/posts')
  return post
}
```

**Why not extract?**
- Single use case
- Simple logic (<10 lines)
- Clear and readable as-is

### Example 2: Complex - Extract to lib/

**Scenario:** Email service used by multiple actions

```typescript
// lib/email.ts
import { Resend } from 'resend'

const resend = new Resend(process.env.RESEND_API_KEY)

export async function sendWelcomeEmail(email: string, name: string) {
  return await resend.emails.send({
    from: 'noreply@example.com',
    to: email,
    subject: 'Welcome to our app!',
    html: `<p>Hi ${name}, welcome aboard!</p>`
  })
}

export async function sendPasswordResetEmail(email: string, token: string) {
  return await resend.emails.send({
    from: 'noreply@example.com',
    to: email,
    subject: 'Reset your password',
    html: `<p>Click here to reset: ${process.env.APP_URL}/reset?token=${token}</p>`
  })
}

export async function sendInvitationEmail(
  email: string,
  organizationName: string,
  inviteToken: string
) {
  return await resend.emails.send({
    from: 'noreply@example.com',
    to: email,
    subject: `Invitation to ${organizationName}`,
    html: `<p>You've been invited to join ${organizationName}. Click here to accept: ${process.env.APP_URL}/invite?token=${inviteToken}</p>`
  })
}
```

**Usage in multiple actions:**

```typescript
// actions/auth.ts
'use server'
import { sendWelcomeEmail } from '@/lib/email'

export async function registerUser(formData: FormData) {
  // ... create user
  await sendWelcomeEmail(user.email, user.name)
}

// actions/staff.ts
'use server'
import { sendInvitationEmail } from '@/lib/email'

export async function inviteStaffMember(email: string, orgId: string) {
  // ... create invitation
  await sendInvitationEmail(email, org.name, invitation.token)
}
```

**Why extract?**
- Used in 3+ actions
- Third-party integration
- Reusable email patterns
- Easy to test in isolation

### Example 3: Complex Database Queries - Extract When Reused

**Scenario:** Complex query used in multiple places

```typescript
// ❌ BAD - Thin wrapper (not worth it)
// lib/db/posts.ts
export async function getPost(id: string) {
  return prisma.post.findUnique({ where: { id } })
}

// ✅ GOOD - Complex reusable query
// lib/db/posts.ts
import { prisma } from '@/lib/prisma'

export async function getPostWithEngagement(postId: string) {
  return await prisma.post.findUnique({
    where: { id: postId },
    include: {
      author: {
        select: {
          id: true,
          name: true,
          avatar: true
        }
      },
      comments: {
        take: 10,
        orderBy: { createdAt: 'desc' },
        include: {
          author: {
            select: { name: true, avatar: true }
          }
        }
      },
      _count: {
        select: {
          likes: true,
          comments: true
        }
      }
    }
  })
}

// Used in multiple actions/pages
// actions/posts.ts
export async function getPostForEdit(postId: string) {
  return await getPostWithEngagement(postId)
}

// app/posts/[id]/page.tsx
export default async function PostPage({ params }: { params: { id: string } }) {
  const post = await getPostWithEngagement(params.id)
  return <PostDetail post={post} />
}
```

**Why extract?**
- Used in 2+ places
- Complex include logic (20+ lines if inline)
- Consistent data structure needed

---

## Service Pattern Examples

### Pattern 1: Stripe Payment Service

**When:** Third-party integration used across multiple actions

```typescript
// lib/stripe/payments.ts
import Stripe from 'stripe'

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2024-01-01'
})

export async function createPaymentIntent(
  amount: number,
  currency: string,
  metadata: Record<string, string>
) {
  return await stripe.paymentIntents.create({
    amount,
    currency,
    metadata,
    automatic_payment_methods: { enabled: true }
  })
}

export async function createCustomer(email: string, name: string) {
  return await stripe.customers.create({
    email,
    name
  })
}

export async function createSubscription(
  customerId: string,
  priceId: string
) {
  return await stripe.subscriptions.create({
    customer: customerId,
    items: [{ price: priceId }]
  })
}
```

**Usage:**

```typescript
// actions/billing.ts
'use server'
import { getCurrentUser } from '@/utils/supabase/server'
import { createPaymentIntent } from '@/lib/stripe/payments'
import { prisma } from '@/lib/prisma'

export async function createCheckoutSession(organizationId: string) {
  const user = await getCurrentUser()
  if (!user) throw new Error('Unauthorized')

  const org = await prisma.organization.findUnique({
    where: { id: organizationId }
  })

  if (!org) throw new Error('Organization not found')

  // Use extracted Stripe logic
  const paymentIntent = await createPaymentIntent(
    org.planPrice * 100, // Convert to cents
    'usd',
    {
      organizationId: org.id,
      userId: user.id
    }
  )

  return { clientSecret: paymentIntent.client_secret }
}
```

### Pattern 2: Permission Checking Helper

**When:** Complex authorization logic used in multiple actions

```typescript
// lib/auth-helpers.ts
import { getCurrentUser } from '@/utils/supabase/server'
import { prisma } from '@/lib/prisma'

export async function requireOrganizationAccess(organizationId: string) {
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

  return { user, membership }
}

export async function requireOrganizationRole(
  organizationId: string,
  allowedRoles: string[]
) {
  const { user, membership } = await requireOrganizationAccess(organizationId)

  if (!allowedRoles.includes(membership.role)) {
    throw new Error(`Must be ${allowedRoles.join(' or ')}`)
  }

  return { user, membership }
}
```

**Usage in multiple actions:**

```typescript
// actions/staff.ts
'use server'
import { requireOrganizationRole } from '@/lib/auth-helpers'
import { prisma } from '@/lib/prisma'

export async function addStaffMember(organizationId: string, email: string) {
  // Only admins can add staff
  await requireOrganizationRole(organizationId, ['ADMIN'])

  return await prisma.organizationMember.create({
    data: { organizationId, email, role: 'MEMBER' }
  })
}

export async function deleteStaffMember(organizationId: string, memberId: string) {
  // Only admins can delete staff
  await requireOrganizationRole(organizationId, ['ADMIN'])

  return await prisma.organizationMember.delete({
    where: { id: memberId }
  })
}

export async function updateProject(organizationId: string, projectId: string, data: any) {
  // Admins and managers can update projects
  await requireOrganizationRole(organizationId, ['ADMIN', 'MANAGER'])

  return await prisma.project.update({
    where: { id: projectId },
    data
  })
}
```

### Pattern 3: Notification Service

**When:** Complex logic with multiple operations, used across app

```typescript
// lib/notifications.ts
import { prisma } from '@/lib/prisma'
import { sendEmail } from '@/lib/email'

export async function notifyUserAboutPost(
  userId: string,
  postId: string,
  type: 'mention' | 'reply' | 'like'
) {
  // Get user preferences
  const preferences = await prisma.userPreference.findUnique({
    where: { userId }
  })

  // Don't notify if user has disabled notifications
  if (!preferences?.notificationsEnabled) {
    return
  }

  // Create in-app notification
  const notification = await prisma.notification.create({
    data: {
      userId,
      type,
      postId,
      read: false
    }
  })

  // Send email if user has email notifications enabled
  if (preferences.emailNotifications) {
    const user = await prisma.user.findUnique({
      where: { id: userId }
    })

    if (user?.email) {
      await sendEmail(
        user.email,
        `New ${type} on your post`,
        `You have a new ${type} on your post.`
      )
    }
  }

  return notification
}
```

**Usage:**

```typescript
// actions/comments.ts
'use server'
import { notifyUserAboutPost } from '@/lib/notifications'

export async function createComment(postId: string, content: string) {
  const user = await getCurrentUser()
  if (!user) throw new Error('Unauthorized')

  const post = await prisma.post.findUnique({
    where: { id: postId }
  })

  const comment = await prisma.comment.create({
    data: {
      postId,
      content,
      userId: user.id
    }
  })

  // Notify post author about the reply
  if (post && post.userId !== user.id) {
    await notifyUserAboutPost(post.userId, postId, 'reply')
  }

  return comment
}
```

---

## Helper Functions vs Services

### Helper Functions (Preferred for Simple Cases)

**Characteristics:**
- Pure functions or simple async functions
- No state
- Single responsibility
- Easy to test

```typescript
// lib/utils/formatting.ts
export function formatCurrency(amount: number): string {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD'
  }).format(amount)
}

export function slugify(text: string): string {
  return text
    .toLowerCase()
    .replace(/[^\w\s-]/g, '')
    .replace(/\s+/g, '-')
}

// lib/utils/dates.ts
export function isExpired(expiresAt: Date): boolean {
  return new Date() > expiresAt
}

export function addDays(date: Date, days: number): Date {
  const result = new Date(date)
  result.setDate(result.getDate() + days)
  return result
}
```

### "Services" (Only for Complex Cases)

**Use when:**
- Multiple related operations
- Third-party integrations
- Needs configuration
- Used in 3+ places

```typescript
// lib/storage.ts
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3'

const s3 = new S3Client({
  region: process.env.AWS_REGION!,
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID!,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY!
  }
})

export async function uploadFile(
  key: string,
  body: Buffer,
  contentType: string
): Promise<string> {
  await s3.send(
    new PutObjectCommand({
      Bucket: process.env.AWS_BUCKET!,
      Key: key,
      Body: body,
      ContentType: contentType
    })
  )

  return `https://${process.env.AWS_BUCKET}.s3.amazonaws.com/${key}`
}

export async function uploadUserAvatar(
  userId: string,
  file: Buffer,
  contentType: string
): Promise<string> {
  const key = `avatars/${userId}-${Date.now()}`
  return uploadFile(key, file, contentType)
}
```

---

## Testing Extracted Logic

### Testing Helper Functions

```typescript
// lib/utils/formatting.test.ts
import { formatCurrency, slugify } from './formatting'

describe('formatCurrency', () => {
  it('formats USD correctly', () => {
    expect(formatCurrency(1234.56)).toBe('$1,234.56')
  })
})

describe('slugify', () => {
  it('converts text to slug', () => {
    expect(slugify('Hello World!')).toBe('hello-world')
  })
})
```

### Testing Services

```typescript
// lib/email.test.ts
import { sendWelcomeEmail } from './email'
import { Resend } from 'resend'

jest.mock('resend')

describe('sendWelcomeEmail', () => {
  it('sends email with correct parameters', async () => {
    const mockSend = jest.fn().mockResolvedValue({ id: '123' })
    ;(Resend as jest.Mock).mockImplementation(() => ({
      emails: { send: mockSend }
    }))

    await sendWelcomeEmail('test@example.com', 'John')

    expect(mockSend).toHaveBeenCalledWith({
      from: 'noreply@example.com',
      to: 'test@example.com',
      subject: 'Welcome to our app!',
      html: expect.stringContaining('John')
    })
  })
})
```

### Testing Auth Helpers

```typescript
// lib/auth-helpers.test.ts
import { requireOrganizationRole } from './auth-helpers'
import { prisma } from '@/lib/prisma'

jest.mock('@/lib/prisma')
jest.mock('@/utils/supabase/server')

describe('requireOrganizationRole', () => {
  it('allows admin access', async () => {
    const mockUser = { id: 'user-123' }
    const mockMembership = { userId: 'user-123', role: 'ADMIN' }

    ;(getCurrentUser as jest.Mock).mockResolvedValue(mockUser)
    ;(prisma.organizationMember.findFirst as jest.Mock).mockResolvedValue(
      mockMembership
    )

    const result = await requireOrganizationRole('org-123', ['ADMIN'])

    expect(result.user).toEqual(mockUser)
    expect(result.membership).toEqual(mockMembership)
  })

  it('throws error for insufficient role', async () => {
    const mockUser = { id: 'user-123' }
    const mockMembership = { userId: 'user-123', role: 'MEMBER' }

    ;(getCurrentUser as jest.Mock).mockResolvedValue(mockUser)
    ;(prisma.organizationMember.findFirst as jest.Mock).mockResolvedValue(
      mockMembership
    )

    await expect(
      requireOrganizationRole('org-123', ['ADMIN'])
    ).rejects.toThrow('Must be ADMIN')
  })
})
```

---

## What NOT to Do

### ❌ Don't Create Thin Wrappers

```typescript
// ❌ BAD - Pointless abstraction
// lib/db/posts.ts
export async function getPost(id: string) {
  return prisma.post.findUnique({ where: { id } })
}

export async function createPost(data: any) {
  return prisma.post.create({ data })
}

// ✅ GOOD - Use Prisma directly in action
// actions/posts.ts
export async function getPost(id: string) {
  return await prisma.post.findUnique({
    where: { id },
    include: { author: true, comments: true }
  })
}
```

### ❌ Don't Over-Engineer with Classes

```typescript
// ❌ BAD - Unnecessary class
class PostService {
  constructor(private prisma: PrismaClient) {}

  async getPost(id: string) {
    return this.prisma.post.findUnique({ where: { id } })
  }
}

// ✅ GOOD - Simple function
export async function getPost(id: string) {
  return await prisma.post.findUnique({ where: { id } })
}
```

### ❌ Don't Create Repository Layer

```typescript
// ❌ BAD - Unnecessary repository
class PostRepository {
  async findById(id: string) {
    return prisma.post.findUnique({ where: { id } })
  }
}

// ✅ GOOD - Use Prisma directly
export async function getPost(id: string) {
  return await prisma.post.findUnique({ where: { id } })
}
```

### ❌ Don't Extract Too Early

```typescript
// ❌ BAD - Extracted too early (only used once)
// lib/posts.ts
export async function validatePostOwnership(postId: string, userId: string) {
  const post = await prisma.post.findUnique({ where: { id: postId } })
  return post?.userId === userId
}

// ✅ GOOD - Keep inline until used 2+ times
// actions/posts.ts
export async function deletePost(postId: string) {
  const user = await getCurrentUser()
  if (!user) throw new Error('Unauthorized')

  const post = await prisma.post.findUnique({ where: { id: postId } })
  if (!post || post.userId !== user.id) {
    throw new Error('Cannot delete this post')
  }

  return await prisma.post.delete({ where: { id: postId } })
}
```

---

## Directory Structure

```
actions/                      # Server actions (primary location)
├── posts.ts                  # Post CRUD actions
├── comments.ts               # Comment actions
├── staff.ts                  # Staff management
└── billing.ts                # Billing actions

lib/
├── email.ts                  # ✅ Email service (used by 5+ actions)
├── stripe/                   # ✅ Stripe integration (complex)
│   ├── payments.ts
│   └── subscriptions.ts
├── auth-helpers.ts           # ✅ Reusable auth checks
├── notifications.ts          # ✅ Notification logic (complex)
├── storage.ts                # ✅ S3 uploads (third-party)
└── utils/                    # ✅ Pure utility functions
    ├── formatting.ts
    ├── dates.ts
    └── validation.ts
```

**Key principle:** Actions are the primary location. Extract to lib/ only when necessary.

---

## Decision Checklist

Before extracting to lib/:

- [ ] Is this used in 2+ actions? (If no, keep in action)
- [ ] Is it complex (20+ lines)? (If no, duplication is fine)
- [ ] Is it a third-party integration? (Extract)
- [ ] Does it need isolated testing? (Extract)
- [ ] Would it make the action more readable? (Maybe extract)

**When in doubt, keep it in the action.** You can always extract later when you actually need it.

---

**Related Files:**
- [architecture-overview.md](architecture-overview.md) - Two-path architecture
- [database-patterns.md](database-patterns.md) - Prisma usage patterns
- [complete-examples.md](complete-examples.md) - Full examples with helpers
- [middleware-guide.md](middleware-guide.md) - Auth helpers pattern
