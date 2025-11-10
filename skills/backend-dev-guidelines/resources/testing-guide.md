# Testing Guide - Next.js + Supabase + Prisma Testing

Complete guide to testing Next.js server actions, Supabase RLS policies, and Prisma transactions following the two-path architecture.

## Table of Contents

- [Testing Philosophy](#testing-philosophy)
- [Testing Server Actions](#testing-server-actions)
- [Testing RLS Policies](#testing-rls-policies)
- [Testing Prisma Queries](#testing-prisma-queries)
- [Testing Transactions](#testing-transactions)
- [Testing Auth Flow](#testing-auth-flow)
- [Mocking Strategies](#mocking-strategies)
- [Test Data Management](#test-data-management)
- [Coverage Targets](#coverage-targets)

---

## Testing Philosophy

### Two-Path Testing Strategy

**Path 1: RLS-Based Operations**
- Test RLS policies at database level
- Test server actions that call Supabase client
- Focus on authorization rules

**Path 2: Server Action + Prisma**
- Test server actions with complex logic
- Test Prisma transactions
- Test business rules and validations

### Testing Priorities

1. **Server actions** (main business logic)
2. **Database operations** (Prisma/Supabase)
3. **Auth boundaries** (getCurrentUser checks)
4. **Error handling** (validation, database errors)

---

## Testing Server Actions

Server actions are the primary place for business logic in Next.js. Test them thoroughly.

### Basic Server Action Test

```typescript
// __tests__/actions/posts.test.ts
import { createPost, getPosts } from '@/actions/posts'
import { createServerClient } from '@/utils/supabase/server'

// Mock Supabase
jest.mock('@/utils/supabase/server')

const mockSupabase = {
  from: jest.fn(() => mockSupabase),
  select: jest.fn(() => mockSupabase),
  insert: jest.fn(() => mockSupabase),
  single: jest.fn(),
  order: jest.fn(() => mockSupabase),
  auth: {
    getUser: jest.fn()
  }
}

describe('Post Actions', () => {
  beforeEach(() => {
    jest.clearAllMocks()
    ;(createServerClient as jest.Mock).mockResolvedValue(mockSupabase)
  })

  describe('createPost', () => {
    it('should create post when authenticated', async () => {
      const formData = new FormData()
      formData.set('title', 'Test Post')
      formData.set('content', 'Test content')

      mockSupabase.insert.mockReturnValue(mockSupabase)
      mockSupabase.select.mockReturnValue(mockSupabase)
      mockSupabase.single.mockResolvedValue({
        data: { id: '123', title: 'Test Post' },
        error: null
      })

      const result = await createPost(formData)

      expect(mockSupabase.from).toHaveBeenCalledWith('posts')
      expect(mockSupabase.insert).toHaveBeenCalledWith({
        title: 'Test Post',
        content: 'Test content'
      })
      expect(result).toEqual({ id: '123', title: 'Test Post' })
    })

    it('should throw error on database failure', async () => {
      const formData = new FormData()
      formData.set('title', 'Test')
      formData.set('content', 'Test')

      mockSupabase.insert.mockReturnValue(mockSupabase)
      mockSupabase.select.mockReturnValue(mockSupabase)
      mockSupabase.single.mockResolvedValue({
        data: null,
        error: { message: 'Database error' }
      })

      await expect(createPost(formData)).rejects.toThrow('Database error')
    })
  })

  describe('getPosts', () => {
    it('should return posts ordered by date', async () => {
      const mockPosts = [
        { id: '1', title: 'Post 1', createdAt: '2024-01-02' },
        { id: '2', title: 'Post 2', createdAt: '2024-01-01' }
      ]

      mockSupabase.select.mockReturnValue(mockSupabase)
      mockSupabase.order.mockResolvedValue({
        data: mockPosts,
        error: null
      })

      const posts = await getPosts()

      expect(mockSupabase.from).toHaveBeenCalledWith('posts')
      expect(mockSupabase.order).toHaveBeenCalledWith('created_at', {
        ascending: false
      })
      expect(posts).toEqual(mockPosts)
    })
  })
})
```

### Testing Complex Server Actions with Prisma

```typescript
// __tests__/actions/staff.test.ts
import { addStaffMember } from '@/actions/staff'
import { getCurrentUser } from '@/utils/supabase/server'
import { prisma } from '@/lib/prisma'

// Mock auth
jest.mock('@/utils/supabase/server')

// Mock Prisma
jest.mock('@/lib/prisma', () => ({
  prisma: {
    $transaction: jest.fn(),
    organizationMember: {
      findFirst: jest.fn()
    },
    organization: {
      findUnique: jest.fn()
    },
    user: {
      findUnique: jest.fn(),
      create: jest.fn()
    },
    auditLog: {
      create: jest.fn()
    }
  }
}))

describe('addStaffMember', () => {
  const mockUser = { id: 'user-123', email: 'admin@test.com' }
  const orgId = 'org-456'

  beforeEach(() => {
    jest.clearAllMocks()
    ;(getCurrentUser as jest.Mock).mockResolvedValue(mockUser)
  })

  it('should add staff member when all checks pass', async () => {
    const formData = new FormData()
    formData.set('email', 'new@test.com')
    formData.set('role', 'MEMBER')
    formData.set('organizationId', orgId)

    // Mock transaction callback execution
    const mockTransaction = jest.fn(async (callback) => {
      // Mock transaction context
      const tx = {
        organizationMember: {
          findFirst: jest.fn().mockResolvedValue({
            role: 'ADMIN' // Admin check passes
          }),
          create: jest.fn().mockResolvedValue({
            id: 'member-789',
            userId: 'new-user-123',
            organizationId: orgId,
            role: 'MEMBER'
          })
        },
        organization: {
          findUnique: jest.fn().mockResolvedValue({
            id: orgId,
            maxMembers: 10,
            _count: { members: 5 } // Under limit
          })
        },
        user: {
          findUnique: jest.fn().mockResolvedValue(null), // User doesn't exist
          create: jest.fn().mockResolvedValue({
            id: 'new-user-123',
            email: 'new@test.com'
          })
        },
        auditLog: {
          create: jest.fn().mockResolvedValue({ id: 'audit-123' })
        }
      }

      return callback(tx)
    })

    ;(prisma.$transaction as jest.Mock).mockImplementation(mockTransaction)

    const result = await addStaffMember(formData)

    expect(result.success).toBe(true)
    expect(result.member).toBeDefined()
    expect(mockTransaction).toHaveBeenCalled()
  })

  it('should reject when user is not admin', async () => {
    const formData = new FormData()
    formData.set('email', 'new@test.com')
    formData.set('organizationId', orgId)

    const mockTransaction = jest.fn(async (callback) => {
      const tx = {
        organizationMember: {
          findFirst: jest.fn().mockResolvedValue(null) // Not admin
        }
      }
      return callback(tx)
    })

    ;(prisma.$transaction as jest.Mock).mockImplementation(mockTransaction)

    const result = await addStaffMember(formData)

    expect(result.success).toBe(false)
    expect(result.error).toContain('admin')
  })

  it('should reject when org at member limit', async () => {
    const formData = new FormData()
    formData.set('email', 'new@test.com')
    formData.set('organizationId', orgId)

    const mockTransaction = jest.fn(async (callback) => {
      const tx = {
        organizationMember: {
          findFirst: jest.fn().mockResolvedValue({ role: 'ADMIN' })
        },
        organization: {
          findUnique: jest.fn().mockResolvedValue({
            maxMembers: 5,
            _count: { members: 5 } // At limit!
          })
        }
      }
      return callback(tx)
    })

    ;(prisma.$transaction as jest.Mock).mockImplementation(mockTransaction)

    const result = await addStaffMember(formData)

    expect(result.success).toBe(false)
    expect(result.error).toContain('limit')
  })

  it('should return error when not authenticated', async () => {
    ;(getCurrentUser as jest.Mock).mockResolvedValue(null)

    const formData = new FormData()
    const result = await addStaffMember(formData)

    expect(result.success).toBe(false)
    expect(result.error).toBe('Unauthorized')
  })
})
```

---

## Testing RLS Policies

RLS policies are tested at the database level. You can't unit test them - use integration tests with a test database.

### RLS Integration Test Setup

```typescript
// __tests__/rls/posts.test.ts
import { createClient } from '@supabase/supabase-js'

const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL!
const supabaseAnonKey = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!

describe('Posts RLS Policies', () => {
  let supabase: ReturnType<typeof createClient>
  let testUser: any
  let testPost: any

  beforeAll(async () => {
    // Create authenticated client
    supabase = createClient(supabaseUrl, supabaseAnonKey)

    // Sign up test user
    const { data, error } = await supabase.auth.signUp({
      email: 'test@example.com',
      password: 'testpassword123'
    })

    if (error) throw error
    testUser = data.user
  })

  afterAll(async () => {
    // Cleanup: Delete test post
    if (testPost) {
      await supabase.from('posts').delete().eq('id', testPost.id)
    }

    // Sign out
    await supabase.auth.signOut()
  })

  describe('SELECT policy', () => {
    it('should allow anyone to view posts', async () => {
      // Create post first (as authenticated user)
      const { data: post } = await supabase
        .from('posts')
        .insert({ title: 'Test Post', content: 'Test' })
        .select()
        .single()

      testPost = post

      // Sign out
      await supabase.auth.signOut()

      // Try to read as anonymous user
      const { data, error } = await supabase
        .from('posts')
        .select('*')
        .eq('id', testPost.id)
        .single()

      expect(error).toBeNull()
      expect(data).toBeDefined()
      expect(data.title).toBe('Test Post')
    })
  })

  describe('INSERT policy', () => {
    it('should allow authenticated users to create posts', async () => {
      const { data, error } = await supabase
        .from('posts')
        .insert({
          title: 'New Post',
          content: 'New content'
        })
        .select()
        .single()

      expect(error).toBeNull()
      expect(data).toBeDefined()
      expect(data.user_id).toBe(testUser.id)

      testPost = data
    })

    it('should reject anonymous users creating posts', async () => {
      await supabase.auth.signOut()

      const { data, error } = await supabase
        .from('posts')
        .insert({
          title: 'Anonymous Post',
          content: 'Should fail'
        })
        .select()
        .single()

      expect(error).toBeDefined()
      expect(error?.code).toBe('42501') // Insufficient privilege
      expect(data).toBeNull()
    })
  })

  describe('UPDATE policy', () => {
    it('should allow users to update their own posts', async () => {
      const { data, error } = await supabase
        .from('posts')
        .update({ title: 'Updated Title' })
        .eq('id', testPost.id)
        .select()
        .single()

      expect(error).toBeNull()
      expect(data.title).toBe('Updated Title')
    })

    it('should prevent users from updating others posts', async () => {
      // Create another user
      const supabase2 = createClient(supabaseUrl, supabaseAnonKey)
      await supabase2.auth.signUp({
        email: 'other@example.com',
        password: 'testpassword123'
      })

      // Try to update first user's post
      const { data, error } = await supabase2
        .from('posts')
        .update({ title: 'Hacked' })
        .eq('id', testPost.id)
        .select()

      expect(error).toBeDefined()
      expect(data).toEqual([]) // No rows updated

      await supabase2.auth.signOut()
    })
  })
})
```

---

## Testing Prisma Queries

### Unit Testing Prisma Queries

```typescript
// __tests__/lib/db/posts.test.ts
import { getPostsByUser } from '@/lib/db/posts'
import { prisma } from '@/lib/prisma'

jest.mock('@/lib/prisma', () => ({
  prisma: {
    post: {
      findMany: jest.fn()
    }
  }
}))

describe('getPostsByUser', () => {
  beforeEach(() => {
    jest.clearAllMocks()
  })

  it('should fetch posts with comments and likes', async () => {
    const mockPosts = [
      {
        id: '1',
        title: 'Post 1',
        comments: [{ id: 'c1', content: 'Comment 1' }],
        _count: { likes: 5 }
      }
    ]

    ;(prisma.post.findMany as jest.Mock).mockResolvedValue(mockPosts)

    const posts = await getPostsByUser('user-123')

    expect(prisma.post.findMany).toHaveBeenCalledWith({
      where: { userId: 'user-123' },
      include: {
        comments: {
          take: 5,
          orderBy: { createdAt: 'desc' }
        },
        _count: { select: { likes: true } }
      },
      orderBy: { createdAt: 'desc' }
    })

    expect(posts).toEqual(mockPosts)
  })

  it('should handle empty results', async () => {
    ;(prisma.post.findMany as jest.Mock).mockResolvedValue([])

    const posts = await getPostsByUser('user-123')

    expect(posts).toEqual([])
  })
})
```

### Integration Testing with Real Database

```typescript
// __tests__/integration/prisma.test.ts
import { prisma } from '@/lib/prisma'

describe('Prisma Integration', () => {
  let testUserId: string

  beforeAll(async () => {
    // Create test user
    const user = await prisma.user.create({
      data: {
        email: `test-${Date.now()}@test.com`
      }
    })
    testUserId = user.id
  })

  afterAll(async () => {
    // Cleanup
    await prisma.post.deleteMany({
      where: { userId: testUserId }
    })
    await prisma.user.delete({
      where: { id: testUserId }
    })
    await prisma.$disconnect()
  })

  it('should create and fetch post', async () => {
    const post = await prisma.post.create({
      data: {
        title: 'Test Post',
        content: 'Test content',
        userId: testUserId
      }
    })

    expect(post.id).toBeDefined()

    const fetched = await prisma.post.findUnique({
      where: { id: post.id }
    })

    expect(fetched).toBeDefined()
    expect(fetched?.title).toBe('Test Post')
  })
})
```

---

## Testing Transactions

Transactions are critical - test them thoroughly.

### Mocking Transaction Callbacks

```typescript
// __tests__/actions/inventory.test.ts
import { updateInventory } from '@/actions/inventory'
import { prisma } from '@/lib/prisma'
import { getCurrentUser } from '@/utils/supabase/server'

jest.mock('@/lib/prisma')
jest.mock('@/utils/supabase/server')

describe('updateInventory transaction', () => {
  const mockUser = { id: 'user-123' }
  const productId = 'product-456'

  beforeEach(() => {
    jest.clearAllMocks()
    ;(getCurrentUser as jest.Mock).mockResolvedValue(mockUser)
  })

  it('should update inventory and log in transaction', async () => {
    const mockTransaction = jest.fn(async (callback) => {
      const tx = {
        staff: {
          findFirst: jest.fn().mockResolvedValue({ role: 'WAREHOUSE' })
        },
        product: {
          findUnique: jest.fn().mockResolvedValue({
            id: productId,
            inventory: 100
          }),
          update: jest.fn().mockResolvedValue({
            id: productId,
            inventory: 110
          })
        },
        inventoryLog: {
          create: jest.fn().mockResolvedValue({ id: 'log-123' })
        }
      }
      return callback(tx)
    })

    ;(prisma.$transaction as jest.Mock).mockImplementation(mockTransaction)

    const result = await updateInventory(productId, 10, 'Restock')

    expect(result.success).toBe(true)
    expect(mockTransaction).toHaveBeenCalled()
  })

  it('should rollback on negative inventory', async () => {
    const mockTransaction = jest.fn(async (callback) => {
      const tx = {
        staff: {
          findFirst: jest.fn().mockResolvedValue({ role: 'WAREHOUSE' })
        },
        product: {
          findUnique: jest.fn().mockResolvedValue({
            id: productId,
            inventory: 5
          })
        }
      }
      return callback(tx)
    })

    ;(prisma.$transaction as jest.Mock).mockImplementation(mockTransaction)

    const result = await updateInventory(productId, -10, 'Sale')

    expect(result.success).toBe(false)
    expect(result.error).toContain('Insufficient')
  })

  it('should handle transaction errors', async () => {
    ;(prisma.$transaction as jest.Mock).mockRejectedValue(
      new Error('Transaction failed')
    )

    const result = await updateInventory(productId, 10, 'Test')

    expect(result.success).toBe(false)
    expect(result.error).toBeDefined()
  })
})
```

---

## Testing Auth Flow

### Testing getCurrentUser Helper

```typescript
// __tests__/utils/supabase/server.test.ts
import { getCurrentUser, createServerSupabaseClient } from '@/utils/supabase/server'
import { createServerClient } from '@supabase/ssr'
import { cookies } from 'next/headers'

jest.mock('@supabase/ssr')
jest.mock('next/headers')

describe('getCurrentUser', () => {
  const mockCookies = {
    getAll: jest.fn(() => []),
    set: jest.fn()
  }

  const mockSupabase = {
    auth: {
      getUser: jest.fn()
    }
  }

  beforeEach(() => {
    jest.clearAllMocks()
    ;(cookies as jest.Mock).mockResolvedValue(mockCookies)
    ;(createServerClient as jest.Mock).mockReturnValue(mockSupabase)
  })

  it('should return user when authenticated', async () => {
    const mockUser = { id: 'user-123', email: 'test@test.com' }
    mockSupabase.auth.getUser.mockResolvedValue({
      data: { user: mockUser },
      error: null
    })

    const user = await getCurrentUser()

    expect(user).toEqual(mockUser)
  })

  it('should return null when not authenticated', async () => {
    mockSupabase.auth.getUser.mockResolvedValue({
      data: { user: null },
      error: null
    })

    const user = await getCurrentUser()

    expect(user).toBeNull()
  })

  it('should return null on error', async () => {
    mockSupabase.auth.getUser.mockResolvedValue({
      data: { user: null },
      error: { message: 'Invalid token' }
    })

    const user = await getCurrentUser()

    expect(user).toBeNull()
  })
})
```

### Testing Auth Boundaries in Server Actions

```typescript
// __tests__/actions/protected.test.ts
import { updateProfile } from '@/actions/dashboard'
import { getCurrentUser } from '@/utils/supabase/server'
import { prisma } from '@/lib/prisma'

jest.mock('@/utils/supabase/server')
jest.mock('@/lib/prisma')

describe('Protected server actions', () => {
  it('should reject when user not authenticated', async () => {
    ;(getCurrentUser as jest.Mock).mockResolvedValue(null)

    const formData = new FormData()
    formData.set('name', 'Test')

    const result = await updateProfile(formData)

    expect(result.success).toBe(false)
    expect(result.error).toBe('Unauthorized')
    expect(prisma.user.update).not.toHaveBeenCalled()
  })

  it('should process when user authenticated', async () => {
    const mockUser = { id: 'user-123', email: 'test@test.com' }
    ;(getCurrentUser as jest.Mock).mockResolvedValue(mockUser)
    ;(prisma.user.update as jest.Mock).mockResolvedValue({
      id: 'user-123',
      name: 'Updated'
    })

    const formData = new FormData()
    formData.set('name', 'Updated')

    const result = await updateProfile(formData)

    expect(result.success).toBe(true)
    expect(prisma.user.update).toHaveBeenCalledWith({
      where: { id: 'user-123' },
      data: expect.objectContaining({ name: 'Updated' })
    })
  })
})
```

---

## Mocking Strategies

### Mock Supabase Client

```typescript
// __tests__/__mocks__/supabase.ts
export const mockSupabaseClient = {
  from: jest.fn(() => mockSupabaseClient),
  select: jest.fn(() => mockSupabaseClient),
  insert: jest.fn(() => mockSupabaseClient),
  update: jest.fn(() => mockSupabaseClient),
  delete: jest.fn(() => mockSupabaseClient),
  eq: jest.fn(() => mockSupabaseClient),
  single: jest.fn(),
  order: jest.fn(() => mockSupabaseClient),
  limit: jest.fn(() => mockSupabaseClient),
  auth: {
    getUser: jest.fn(),
    signUp: jest.fn(),
    signIn: jest.fn(),
    signOut: jest.fn()
  }
}

// Usage in tests:
import { createServerClient } from '@/utils/supabase/server'
import { mockSupabaseClient } from '../__mocks__/supabase'

jest.mock('@/utils/supabase/server')
;(createServerClient as jest.Mock).mockResolvedValue(mockSupabaseClient)
```

### Mock Prisma Client

```typescript
// __tests__/__mocks__/prisma.ts
export const mockPrismaClient = {
  user: {
    findMany: jest.fn(),
    findUnique: jest.fn(),
    create: jest.fn(),
    update: jest.fn(),
    delete: jest.fn()
  },
  post: {
    findMany: jest.fn(),
    findUnique: jest.fn(),
    create: jest.fn(),
    update: jest.fn(),
    delete: jest.fn()
  },
  $transaction: jest.fn()
}

// Usage:
jest.mock('@/lib/prisma', () => ({
  prisma: mockPrismaClient
}))
```

### Mock Next.js Functions

```typescript
// Mock revalidatePath
jest.mock('next/cache', () => ({
  revalidatePath: jest.fn(),
  revalidateTag: jest.fn()
}))

// Mock redirect
jest.mock('next/navigation', () => ({
  redirect: jest.fn(),
  notFound: jest.fn()
}))
```

---

## Test Data Management

### Test Database Setup

For integration tests, use a separate test database:

```typescript
// jest.setup.ts
import { prisma } from '@/lib/prisma'

beforeAll(async () => {
  // Ensure test database is clean
  await prisma.$executeRaw`TRUNCATE TABLE posts CASCADE`
  await prisma.$executeRaw`TRUNCATE TABLE users CASCADE`
})

afterAll(async () => {
  await prisma.$disconnect()
})
```

### Factory Functions

Create reusable test data factories:

```typescript
// __tests__/factories/user.ts
import { prisma } from '@/lib/prisma'

export async function createTestUser(overrides = {}) {
  return await prisma.user.create({
    data: {
      email: `test-${Date.now()}@test.com`,
      ...overrides
    }
  })
}

export async function createTestPost(userId: string, overrides = {}) {
  return await prisma.post.create({
    data: {
      title: 'Test Post',
      content: 'Test content',
      userId,
      ...overrides
    }
  })
}

// Usage:
const user = await createTestUser({ email: 'specific@test.com' })
const post = await createTestPost(user.id, { title: 'Custom Title' })
```

### Cleanup Strategies

```typescript
// __tests__/helpers/cleanup.ts
import { prisma } from '@/lib/prisma'

export async function cleanupTestData(ids: {
  posts?: string[]
  users?: string[]
}) {
  if (ids.posts) {
    await prisma.post.deleteMany({
      where: { id: { in: ids.posts } }
    })
  }

  if (ids.users) {
    await prisma.user.deleteMany({
      where: { id: { in: ids.users } }
    })
  }
}

// Usage:
describe('Test suite', () => {
  const testIds = { posts: [], users: [] }

  afterAll(async () => {
    await cleanupTestData(testIds)
  })

  it('creates data', async () => {
    const user = await createTestUser()
    testIds.users.push(user.id)
    // test...
  })
})
```

---

## Coverage Targets

### Recommended Coverage

Following the two-path architecture:

**Server Actions (actions/*.ts):**
- Target: 80%+ coverage
- Priority: High (main business logic)
- Test: All auth checks, validations, error paths

**Prisma Queries (lib/db/*.ts):**
- Target: 70%+ coverage
- Priority: Medium (reusable queries)
- Test: Query logic, includes, filters

**RLS Policies:**
- Target: 100% integration tests
- Priority: High (security boundary)
- Test: All CRUD operations, ownership checks

**Helper Functions (lib/*):**
- Target: 80%+ coverage
- Priority: Medium
- Test: Edge cases, error handling

### Running Tests

```bash
# Run all tests
npm test

# Run with coverage
npm test -- --coverage

# Run specific test file
npm test posts.test.ts

# Run integration tests only
npm test -- --testPathPattern=integration

# Watch mode
npm test -- --watch
```

### Coverage Configuration

```javascript
// jest.config.js
module.exports = {
  collectCoverageFrom: [
    'actions/**/*.ts',
    'lib/**/*.ts',
    '!lib/prisma.ts', // Exclude prisma client setup
    '!**/*.d.ts',
    '!**/__tests__/**'
  ],
  coverageThresholds: {
    global: {
      branches: 70,
      functions: 80,
      lines: 80,
      statements: 80
    },
    './actions/**/*.ts': {
      branches: 80,
      functions: 90,
      lines: 90,
      statements: 90
    }
  }
}
```

---

**Related Files:**
- [architecture-overview.md](architecture-overview.md) - Two-path testing strategy
- [database-patterns.md](database-patterns.md) - Prisma query patterns
- [complete-examples.md](complete-examples.md) - Full examples to test
- [async-and-errors.md](async-and-errors.md) - Error handling in tests
