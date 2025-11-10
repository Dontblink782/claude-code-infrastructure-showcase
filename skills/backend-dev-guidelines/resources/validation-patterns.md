# Validation Patterns - Next.js + Supabase + Prisma

Complete guide to input validation using Zod schemas in Next.js server actions and API routes.

## Table of Contents

- [Why Zod?](#why-zod)
- [Basic Zod Patterns](#basic-zod-patterns)
- [Validation in Server Actions](#validation-in-server-actions)
- [Validation with RLS Path](#validation-with-rls-path)
- [Schema Examples](#schema-examples)
- [Error Handling](#error-handling)
- [Advanced Patterns](#advanced-patterns)

---

## Why Zod?

### Benefits Over Joi/Other Libraries

**Type Safety:**
- ✅ Full TypeScript inference
- ✅ Runtime + compile-time validation
- ✅ Automatic type generation

**Developer Experience:**
- ✅ Intuitive API
- ✅ Composable schemas
- ✅ Excellent error messages

**Performance:**
- ✅ Fast validation
- ✅ Small bundle size
- ✅ Tree-shakeable

**Next.js Integration:**
- ✅ Works seamlessly with Server Actions
- ✅ Type-safe FormData parsing
- ✅ Client/server validation reuse

### Migration from Joi

Modern validation uses Zod instead of Joi:

```typescript
// ❌ OLD - Joi (being phased out)
const schema = Joi.object({
    email: Joi.string().email().required(),
    name: Joi.string().min(3).required(),
});

// ✅ NEW - Zod (preferred)
const schema = z.object({
    email: z.string().email(),
    name: z.string().min(3),
});
```

---

## Basic Zod Patterns

### Primitive Types

```typescript
import { z } from 'zod';

// Strings
const nameSchema = z.string();
const emailSchema = z.string().email();
const urlSchema = z.string().url();
const uuidSchema = z.string().uuid();
const minLengthSchema = z.string().min(3);
const maxLengthSchema = z.string().max(100);

// Numbers
const ageSchema = z.number().int().positive();
const priceSchema = z.number().positive();
const rangeSchema = z.number().min(0).max(100);

// Booleans
const activeSchema = z.boolean();

// Dates
const dateSchema = z.string().datetime(); // ISO 8601 string
const nativeDateSchema = z.date(); // Native Date object

// Enums
const roleSchema = z.enum(['admin', 'operations', 'user']);
const statusSchema = z.enum(['PENDING', 'APPROVED', 'REJECTED']);
```

### Objects

```typescript
// Simple object
const userSchema = z.object({
    email: z.string().email(),
    name: z.string(),
    age: z.number().int().positive(),
});

// Nested objects
const addressSchema = z.object({
    street: z.string(),
    city: z.string(),
    zipCode: z.string().regex(/^\d{5}$/),
});

const userWithAddressSchema = z.object({
    name: z.string(),
    address: addressSchema,
});

// Optional fields
const userSchema = z.object({
    name: z.string(),
    email: z.string().email().optional(),
    phone: z.string().optional(),
});

// Nullable fields
const userSchema = z.object({
    name: z.string(),
    middleName: z.string().nullable(),
});
```

### Arrays

```typescript
// Array of primitives
const rolesSchema = z.array(z.string());
const numbersSchema = z.array(z.number());

// Array of objects
const usersSchema = z.array(
    z.object({
        id: z.string(),
        name: z.string(),
    })
);

// Array with constraints
const tagsSchema = z.array(z.string()).min(1).max(10);
const nonEmptyArray = z.array(z.string()).nonempty();
```

---

## Validation in Server Actions

### Pattern 1: Server Action with Zod (Recommended)

**When to use:** Complex business logic, transactions, server-side validation required.

```typescript
// actions/posts.ts
'use server'
import { getCurrentUser } from '@/utils/supabase/server'
import { prisma } from '@/lib/prisma'
import { revalidatePath } from 'next/cache'
import { z } from 'zod'

// Define validation schema
const createPostSchema = z.object({
    title: z.string().min(3).max(150),
    content: z.string().min(10).max(10000),
    tags: z.array(z.string()).max(5).optional(),
    published: z.boolean().default(false),
})

// Infer TypeScript type from schema
type CreatePostInput = z.infer<typeof createPostSchema>

export async function createPost(formData: FormData) {
    // 1. Auth check
    const user = await getCurrentUser()
    if (!user) {
        return { success: false, error: 'Unauthorized' }
    }

    // 2. Parse FormData to object
    const rawData = {
        title: formData.get('title'),
        content: formData.get('content'),
        tags: formData.get('tags')
            ? JSON.parse(formData.get('tags') as string)
            : undefined,
        published: formData.get('published') === 'true',
    }

    // 3. Validate with Zod
    const result = createPostSchema.safeParse(rawData)

    if (!result.success) {
        return {
            success: false,
            error: 'Validation failed',
            details: result.error.errors,
        }
    }

    // 4. Business logic with validated data
    try {
        const post = await prisma.post.create({
            data: {
                ...result.data,
                userId: user.id,
            },
        })

        revalidatePath('/posts')
        return { success: true, data: post }
    } catch (error) {
        return {
            success: false,
            error: 'Failed to create post',
        }
    }
}
```

### Client Component Usage

```typescript
// components/CreatePostForm.tsx
'use client'
import { createPost } from '@/actions/posts'
import { useState, useTransition } from 'react'

export function CreatePostForm() {
    const [error, setError] = useState<string | null>(null)
    const [fieldErrors, setFieldErrors] = useState<Record<string, string>>({})
    const [isPending, startTransition] = useTransition()

    async function handleSubmit(formData: FormData) {
        setError(null)
        setFieldErrors({})

        startTransition(async () => {
            const result = await createPost(formData)

            if (!result.success) {
                setError(result.error)

                // Display field-specific errors
                if (result.details) {
                    const errors: Record<string, string> = {}
                    result.details.forEach((err) => {
                        const field = err.path.join('.')
                        errors[field] = err.message
                    })
                    setFieldErrors(errors)
                }
            }
        })
    }

    return (
        <form action={handleSubmit}>
            {error && <div className="error">{error}</div>}

            <div>
                <input name="title" placeholder="Title" required />
                {fieldErrors.title && (
                    <span className="field-error">{fieldErrors.title}</span>
                )}
            </div>

            <div>
                <textarea name="content" placeholder="Content" required />
                {fieldErrors.content && (
                    <span className="field-error">{fieldErrors.content}</span>
                )}
            </div>

            <button type="submit" disabled={isPending}>
                {isPending ? 'Creating...' : 'Create Post'}
            </button>
        </form>
    )
}
```

### Pattern 2: Complex Transaction with Validation

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
    organizationId: z.string().uuid(),
})

export async function addStaffMember(formData: FormData) {
    // 1. Auth
    const user = await getCurrentUser()
    if (!user) {
        return { success: false, error: 'Unauthorized' }
    }

    // 2. Validate
    const result = addStaffSchema.safeParse({
        email: formData.get('email'),
        role: formData.get('role'),
        organizationId: formData.get('organizationId'),
    })

    if (!result.success) {
        return {
            success: false,
            error: 'Validation failed',
            details: result.error.errors,
        }
    }

    const { email, role, organizationId } = result.data

    try {
        // 3. Transaction with business logic
        const member = await prisma.$transaction(async (tx) => {
            // Check admin permission
            const membership = await tx.organizationMember.findFirst({
                where: {
                    userId: user.id,
                    organizationId,
                    role: 'ADMIN',
                },
            })

            if (!membership) {
                throw new Error('Only admins can add staff')
            }

            // Check limits
            const org = await tx.organization.findUnique({
                where: { id: organizationId },
                include: { _count: { select: { members: true } } },
            })

            if (!org) {
                throw new Error('Organization not found')
            }

            if (org._count.members >= org.maxMembers) {
                throw new Error('Staff limit reached')
            }

            // Create/find user
            let staffUser = await tx.user.findUnique({
                where: { email },
            })

            if (!staffUser) {
                staffUser = await tx.user.create({
                    data: { email },
                })
            }

            // Create membership
            const newMember = await tx.organizationMember.create({
                data: {
                    userId: staffUser.id,
                    organizationId,
                    role,
                    invitedBy: user.id,
                },
            })

            // Audit log
            await tx.auditLog.create({
                data: {
                    action: 'STAFF_ADDED',
                    userId: user.id,
                    organizationId,
                    metadata: { staffUserId: staffUser.id, role },
                },
            })

            return newMember
        })

        revalidatePath(`/dashboard/${organizationId}/team`)
        return { success: true, member }
    } catch (error) {
        return {
            success: false,
            error: error instanceof Error ? error.message : 'Failed to add staff',
        }
    }
}
```

---

## Validation with RLS Path

### Pattern 3: RLS with Client-Side Validation

**When to use:** Simple CRUD with straightforward auth, no complex business logic.

```typescript
// lib/validations/posts.ts
import { z } from 'zod'

export const createPostSchema = z.object({
    title: z.string().min(3).max(150),
    content: z.string().min(10).max(10000),
    published: z.boolean().default(false),
})

export type CreatePostInput = z.infer<typeof createPostSchema>
```

```typescript
// hooks/useCreatePost.ts
'use client'
import { createBrowserClient } from '@/utils/supabase/client'
import { useState } from 'react'
import { createPostSchema, CreatePostInput } from '@/lib/validations/posts'

export function useCreatePost() {
    const [loading, setLoading] = useState(false)
    const [errors, setErrors] = useState<Record<string, string>>({})
    const supabase = createBrowserClient()

    async function createPost(input: CreatePostInput) {
        setLoading(true)
        setErrors({})

        // Validate on client
        const result = createPostSchema.safeParse(input)

        if (!result.success) {
            const fieldErrors: Record<string, string> = {}
            result.error.errors.forEach((err) => {
                const field = err.path.join('.')
                fieldErrors[field] = err.message
            })
            setErrors(fieldErrors)
            setLoading(false)
            return { data: null, error: 'Validation failed' }
        }

        try {
            // RLS handles auth check at database level
            const { data, error } = await supabase
                .from('posts')
                .insert(result.data)
                .select()
                .single()

            if (error) throw error
            return { data, error: null }
        } catch (error) {
            return {
                data: null,
                error: error instanceof Error ? error.message : 'Failed to create post',
            }
        } finally {
            setLoading(false)
        }
    }

    return { createPost, loading, errors }
}
```

```typescript
// components/CreatePostForm.tsx
'use client'
import { useCreatePost } from '@/hooks/useCreatePost'
import { useState } from 'react'
import { useRouter } from 'next/navigation'

export function CreatePostForm() {
    const { createPost, loading, errors } = useCreatePost()
    const [title, setTitle] = useState('')
    const [content, setContent] = useState('')
    const router = useRouter()

    async function handleSubmit(e: FormEvent) {
        e.preventDefault()

        const { data, error } = await createPost({
            title,
            content,
            published: false,
        })

        if (error) {
            // Errors displayed via hook
            return
        }

        router.push(`/posts/${data.id}`)
        router.refresh()
    }

    return (
        <form onSubmit={handleSubmit}>
            <div>
                <input
                    value={title}
                    onChange={(e) => setTitle(e.target.value)}
                    placeholder="Title"
                    required
                />
                {errors.title && <span className="error">{errors.title}</span>}
            </div>

            <div>
                <textarea
                    value={content}
                    onChange={(e) => setContent(e.target.value)}
                    placeholder="Content"
                    required
                />
                {errors.content && <span className="error">{errors.content}</span>}
            </div>

            <button type="submit" disabled={loading}>
                {loading ? 'Creating...' : 'Create Post'}
            </button>
        </form>
    )
}
```

---

## Schema Examples

### Multi-Tenant Staff Schema

```typescript
// lib/validations/staff.ts
import { z } from 'zod'

export const staffRoleSchema = z.enum(['ADMIN', 'MANAGER', 'MEMBER'])

export const addStaffSchema = z.object({
    email: z.string().email('Invalid email address'),
    role: staffRoleSchema,
    organizationId: z.string().uuid('Invalid organization ID'),
})

export const updateStaffSchema = z.object({
    role: staffRoleSchema.optional(),
    isActive: z.boolean().optional(),
})

export type AddStaffInput = z.infer<typeof addStaffSchema>
export type UpdateStaffInput = z.infer<typeof updateStaffSchema>
```

### Form Builder Schema

```typescript
// lib/validations/forms.ts
import { z } from 'zod'

export const questionTypeSchema = z.enum([
    'input',
    'textbox',
    'editor',
    'dropdown',
    'autocomplete',
    'checkbox',
    'radio',
    'upload',
])

export const questionOptionSchema = z.object({
    id: z.number().int().positive().optional(),
    label: z.string().max(100),
    order: z.number().int().min(0).default(0),
})

export const questionSchema = z.object({
    id: z.number().int().positive().optional(),
    formId: z.number().int().positive(),
    sectionId: z.number().int().positive().optional(),
    label: z.string().min(1).max(500),
    description: z.string().max(5000).optional(),
    type: questionTypeSchema,
    options: z.array(questionOptionSchema).optional(),
    required: z.boolean().default(false),
    maxLength: z.number().int().positive().optional(),
})

export const formSectionSchema = z.object({
    id: z.number().int().positive(),
    formId: z.number().int().positive(),
    label: z.string().min(1).max(500),
    description: z.string().max(5000).optional(),
    questions: z.array(questionSchema).optional(),
})

export const createFormSchema = z.object({
    label: z.string().min(1).max(150),
    description: z.string().max(6000).optional(),
    isPhase: z.boolean().default(false),
})

export type Question = z.infer<typeof questionSchema>
export type FormSection = z.infer<typeof formSectionSchema>
export type CreateFormInput = z.infer<typeof createFormSchema>
```

### Workflow Schema

```typescript
// lib/validations/workflows.ts
import { z } from 'zod'

export const workflowEntitySchema = z.enum(['Post', 'User', 'Comment', 'Order'])

export const startWorkflowSchema = z.object({
    workflowCode: z.string().min(1, 'Workflow code is required'),
    entityType: workflowEntitySchema,
    entityId: z.number().int().positive(),
    dryRun: z.boolean().default(false),
})

export const completeStepSchema = z.object({
    stepInstanceId: z.number().int().positive(),
    answers: z.record(z.string(), z.any()),
    comment: z.string().max(1000).optional(),
})

export type StartWorkflowInput = z.infer<typeof startWorkflowSchema>
export type CompleteStepInput = z.infer<typeof completeStepSchema>
```

---

## Error Handling

### Zod Error Format

```typescript
try {
    const validated = schema.parse(data)
} catch (error) {
    if (error instanceof z.ZodError) {
        console.log(error.errors)
        // [
        //   {
        //     code: 'invalid_type',
        //     expected: 'string',
        //     received: 'number',
        //     path: ['email'],
        //     message: 'Expected string, received number'
        //   }
        // ]
    }
}
```

### Custom Error Messages

```typescript
const userSchema = z.object({
    email: z.string().email({ message: 'Please provide a valid email address' }),
    name: z.string().min(2, { message: 'Name must be at least 2 characters' }),
    age: z.number().int().positive({ message: 'Age must be a positive number' }),
})
```

### Server Action Error Response Pattern

```typescript
// actions/posts.ts
export async function createPost(formData: FormData) {
    const result = schema.safeParse(data)

    if (!result.success) {
        return {
            success: false,
            error: 'Validation failed',
            details: result.error.errors.map((err) => ({
                field: err.path.join('.'),
                message: err.message,
            })),
        }
    }

    // ... rest of logic
}
```

### Client-Side Error Display

```typescript
// components/CreatePostForm.tsx
'use client'

export function CreatePostForm() {
    const [fieldErrors, setFieldErrors] = useState<Record<string, string>>({})

    async function handleSubmit(formData: FormData) {
        const result = await createPost(formData)

        if (!result.success && result.details) {
            const errors: Record<string, string> = {}
            result.details.forEach((err) => {
                errors[err.field] = err.message
            })
            setFieldErrors(errors)
        }
    }

    return (
        <form action={handleSubmit}>
            <input name="title" />
            {fieldErrors.title && (
                <span className="error">{fieldErrors.title}</span>
            )}
        </form>
    )
}
```

---

## Advanced Patterns

### Conditional Validation

```typescript
// Validate based on other field values
const submissionSchema = z.object({
    type: z.enum(['NEW', 'UPDATE']),
    postId: z.number().optional(),
}).refine(
    (data) => {
        // If type is UPDATE, postId is required
        if (data.type === 'UPDATE') {
            return data.postId !== undefined
        }
        return true
    },
    {
        message: 'postId is required when type is UPDATE',
        path: ['postId'],
    }
)
```

### Transform Data

```typescript
// Transform FormData strings to proper types
const formDataSchema = z.object({
    name: z.string(),
    age: z.string().transform((val) => parseInt(val, 10)),
    tags: z.string().transform((str) => JSON.parse(str)),
    date: z.string().transform((str) => new Date(str)),
})

// In server action
const result = formDataSchema.parse({
    name: formData.get('name'),
    age: formData.get('age'), // "25" → 25
    tags: formData.get('tags'), // '["a","b"]' → ["a", "b"]
    date: formData.get('date'), // "2024-01-01" → Date object
})
```

### Preprocess Data

```typescript
// Trim and normalize before validation
const userSchema = z.object({
    email: z.preprocess(
        (val) => typeof val === 'string' ? val.trim().toLowerCase() : val,
        z.string().email()
    ),
    name: z.preprocess(
        (val) => typeof val === 'string' ? val.trim() : val,
        z.string().min(2)
    ),
})
```

### Union Types

```typescript
// Multiple possible types
const idSchema = z.union([z.string(), z.number()])

// Discriminated unions for different form types
const notificationSchema = z.discriminatedUnion('type', [
    z.object({
        type: z.literal('email'),
        recipient: z.string().email(),
        subject: z.string(),
    }),
    z.object({
        type: z.literal('sms'),
        phoneNumber: z.string(),
        message: z.string(),
    }),
])
```

### Schema Composition

```typescript
// Base schemas
const timestampsSchema = z.object({
    createdAt: z.string().datetime(),
    updatedAt: z.string().datetime(),
})

const auditSchema = z.object({
    createdBy: z.string(),
    updatedBy: z.string(),
})

// Compose schemas
const userSchema = z.object({
    id: z.string(),
    email: z.string().email(),
    name: z.string(),
}).merge(timestampsSchema).merge(auditSchema)

// Extend schemas
const adminUserSchema = userSchema.extend({
    adminLevel: z.number().int().min(1).max(5),
    permissions: z.array(z.string()),
})

// Pick specific fields (for public API responses)
const publicUserSchema = userSchema.pick({
    id: true,
    name: true,
    // email excluded
})

// Omit fields
const userWithoutTimestamps = userSchema.omit({
    createdAt: true,
    updatedAt: true,
})
```

### Shared Validation Logic

```typescript
// lib/validations/shared.ts
import { z } from 'zod'

// Reusable field validators
export const emailField = z.string().email('Invalid email address')
export const passwordField = z.string().min(8, 'Password must be at least 8 characters')
export const uuidField = z.string().uuid('Invalid ID format')
export const slugField = z.string().regex(/^[a-z0-9-]+$/, 'Invalid slug format')

// Reusable object schemas
export const paginationSchema = z.object({
    page: z.number().int().positive().default(1),
    limit: z.number().int().positive().max(100).default(20),
})

export const sortSchema = z.object({
    sortBy: z.string().optional(),
    sortOrder: z.enum(['asc', 'desc']).default('desc'),
})

// Use in multiple actions
export const listPostsSchema = z.object({
    category: z.string().optional(),
}).merge(paginationSchema).merge(sortSchema)
```

### FormData Helper Utility

```typescript
// lib/utils/formData.ts
import { z } from 'zod'

export function parseFormData<T extends z.ZodType>(
    formData: FormData,
    schema: T
): z.infer<T> | { error: string; details: z.ZodError } {
    const data = Object.fromEntries(formData.entries())

    const result = schema.safeParse(data)

    if (!result.success) {
        return { error: 'Validation failed', details: result.error }
    }

    return result.data
}

// Usage in server action
import { parseFormData } from '@/lib/utils/formData'

export async function createPost(formData: FormData) {
    const parsed = parseFormData(formData, createPostSchema)

    if ('error' in parsed) {
        return { success: false, ...parsed }
    }

    // parsed is fully typed!
    const post = await prisma.post.create({ data: parsed })
}
```

---

## Decision Tree: Where to Validate?

**Validate in Server Actions when:**
- ✅ Complex business logic required
- ✅ Transactions needed
- ✅ Server-side only validation (security)
- ✅ Multi-step operations
- ✅ Need to check database state

**Validate in Client (RLS path) when:**
- ✅ Simple CRUD operations
- ✅ Immediate user feedback needed
- ✅ No complex business rules
- ✅ RLS handles authorization
- ✅ Real-time validation desired

**Validate in Both when:**
- ✅ Best UX (client feedback) + Security (server enforcement)
- ✅ Shared validation schemas
- ✅ Progressive enhancement

---

**Related Files:**
- [architecture-overview.md](architecture-overview.md) - Two-path architecture patterns
- [complete-examples.md](complete-examples.md) - Full validation examples
- [async-and-errors.md](async-and-errors.md) - Error handling patterns
