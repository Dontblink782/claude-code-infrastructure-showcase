# Styling Guide

Modern styling patterns for Next.js 15 with Tailwind CSS and ShadCN UI. Learn utility-first styling, conditional classes, theme customization, and responsive design.

---

## Tailwind CSS Philosophy

**Utility-first CSS** - Compose styles from small, single-purpose utility classes.

**Traditional CSS (Old Way):**
```css
/* styles.css */
.card {
    padding: 1rem;
    border-radius: 0.5rem;
    background-color: white;
    box-shadow: 0 1px 3px rgba(0,0,0,0.1);
}
```

**Tailwind (Modern Way):**
```typescript
// ✅ Compose with utility classes
<div className="p-4 rounded-lg bg-white shadow-md">
    Card content
</div>
```

**Benefits:**
- No context switching (HTML → CSS)
- No naming things
- No unused CSS
- Responsive by default
- Easy to maintain

---

## Basic Utility Patterns

### Spacing

```typescript
// Padding
p-4       // padding: 1rem (all sides)
px-4      // padding-left/right: 1rem
py-2      // padding-top/bottom: 0.5rem
pt-2      // padding-top: 0.5rem
pr-4      // padding-right: 1rem

// Margin
m-4, mx-4, my-2, mt-2, mr-4, mb-2, ml-4

// Units (default scale)
p-0   // 0
p-1   // 0.25rem (4px)
p-2   // 0.5rem (8px)
p-4   // 1rem (16px)
p-8   // 2rem (32px)
```

### Colors

```typescript
// Text
text-black
text-white
text-gray-500
text-blue-600

// Background
bg-white
bg-gray-100
bg-blue-500

// Border
border-gray-300
border-red-500

// ShadCN Theme Colors (CSS variables)
text-foreground        // Primary text
text-muted-foreground  // Secondary text
bg-background          // Page background
bg-card               // Card background
border-border         // Border color
```

### Layout

```typescript
// Display
block, inline-block, flex, inline-flex, grid, hidden

// Flexbox
<div className="flex items-center justify-between gap-4">
    Flex container
</div>

// Grid
<div className="grid grid-cols-3 gap-4">
    Grid container
</div>
```

### Sizing

```typescript
// Width
w-full       // 100%
w-1/2       // 50%
w-64        // 16rem (256px)
max-w-sm    // max-width: 24rem

// Height
h-full, h-screen, h-32, min-h-screen

// Examples
<div className="w-full max-w-4xl mx-auto">
    Centered container with max width
</div>
```

---

## Responsive Design

### Breakpoint Prefixes

```typescript
// Mobile-first (default = mobile)
<div className="p-4 md:p-8 lg:p-12">
    // p-4 on mobile
    // p-8 on tablet (768px+)
    // p-12 on desktop (1024px+)
</div>

// Breakpoints:
sm:  // 640px
md:  // 768px
lg:  // 1024px
xl:  // 1280px
2xl: // 1536px
```

### Common Responsive Patterns

```typescript
// Stack on mobile, row on desktop
<div className="flex flex-col md:flex-row gap-4">
    <div>Left</div>
    <div>Right</div>
</div>

// Full width on mobile, half on desktop
<div className="w-full md:w-1/2">
    Responsive width
</div>

// Grid: 1 column mobile, 2 tablet, 3 desktop
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
    <div>Item 1</div>
    <div>Item 2</div>
    <div>Item 3</div>
</div>
```

---

## Conditional Classes with cn()

### The cn() Helper

ShadCN provides `cn()` utility for combining class names with conditionals:

```typescript
// lib/utils.ts (provided by ShadCN)
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
    return twMerge(clsx(inputs));
}
```

**What it does:**
- Merges class names
- Handles conditionals
- Resolves Tailwind conflicts (last one wins)

### Basic Conditional Styling

```typescript
import { cn } from '@/lib/utils';

interface ButtonProps {
    variant: 'primary' | 'secondary';
    size: 'sm' | 'md' | 'lg';
}

export function Button({ variant, size }: ButtonProps) {
    return (
        <button
            className={cn(
                'rounded font-medium',
                variant === 'primary' && 'bg-blue-600 text-white',
                variant === 'secondary' && 'bg-gray-200 text-gray-900',
                size === 'sm' && 'px-3 py-1 text-sm',
                size === 'md' && 'px-4 py-2 text-base',
                size === 'lg' && 'px-6 py-3 text-lg'
            )}
        >
            Button
        </button>
    );
}
```

### Advanced Conditional Patterns

```typescript
export function OrderCard({ status, isSelected, className }: OrderCardProps) {
    return (
        <div
            className={cn(
                // Base styles
                'p-4 rounded-lg border transition-colors',
                // Status-based
                status === 'completed' && 'border-green-500 bg-green-50',
                status === 'pending' && 'border-yellow-500 bg-yellow-50',
                status === 'cancelled' && 'border-red-500 bg-red-50',
                // Selection state
                isSelected && 'ring-2 ring-blue-500',
                // Allow override from parent
                className
            )}
        >
            Content
        </div>
    );
}
```

### Config Object Pattern

For complex variants, extract to config object:

```typescript
const statusStyles = {
    completed: 'border-green-500 bg-green-50 text-green-900',
    pending: 'border-yellow-500 bg-yellow-50 text-yellow-900',
    cancelled: 'border-red-500 bg-red-50 text-red-900',
};

export function OrderCard({ status }: OrderCardProps) {
    return (
        <div className={cn('p-4 rounded-lg border', statusStyles[status])}>
            Content
        </div>
    );
}
```

---

## ShadCN Theme Customization

### CSS Variables (Theme Colors)

ShadCN uses CSS variables for theming:

```css
/* app/globals.css */
@layer base {
    :root {
        --background: 0 0% 100%;
        --foreground: 222.2 84% 4.9%;
        --card: 0 0% 100%;
        --card-foreground: 222.2 84% 4.9%;
        --primary: 222.2 47.4% 11.2%;
        --primary-foreground: 210 40% 98%;
        /* ... more variables */
    }

    .dark {
        --background: 222.2 84% 4.9%;
        --foreground: 210 40% 98%;
        /* ... dark mode overrides */
    }
}
```

**Usage:**
```typescript
// Automatically uses theme colors
<div className="bg-background text-foreground">
    Themed content
</div>

<div className="bg-primary text-primary-foreground">
    Primary button
</div>
```

### Customizing Theme

**Option 1: Update CSS Variables**
```css
/* app/globals.css */
:root {
    --primary: 200 100% 50%; /* Custom blue */
    --radius: 0.75rem; /* Larger border radius */
}
```

**Option 2: Tailwind Config**
```javascript
// tailwind.config.ts
export default {
    theme: {
        extend: {
            colors: {
                border: 'hsl(var(--border))',
                background: 'hsl(var(--background))',
                // Add custom colors
                brand: {
                    50: '#f0f9ff',
                    500: '#0ea5e9',
                    900: '#0c4a6e',
                },
            },
        },
    },
};
```

---

## Dark Mode

### Using next-themes

```typescript
// app/layout.tsx
import { ThemeProvider } from 'next-themes';

export default function RootLayout({ children }: { children: React.ReactNode }) {
    return (
        <html lang="en" suppressHydrationWarning>
            <body>
                <ThemeProvider attribute="class" defaultTheme="system">
                    {children}
                </ThemeProvider>
            </body>
        </html>
    );
}
```

### Dark Mode Classes

```typescript
// Auto-switches based on theme
<div className="bg-white dark:bg-gray-900">
    <p className="text-gray-900 dark:text-gray-100">
        Text that adapts to theme
    </p>
</div>
```

### Theme Toggle Component

```typescript
'use client';

import { useTheme } from 'next-themes';
import { Moon, Sun } from 'lucide-react';
import { Button } from '@/components/ui/button';

export function ThemeToggle() {
    const { theme, setTheme } = useTheme();

    return (
        <Button
            variant="ghost"
            size="sm"
            onClick={() => setTheme(theme === 'dark' ? 'light' : 'dark')}
        >
            <Sun className="h-4 w-4 rotate-0 scale-100 transition-all dark:-rotate-90 dark:scale-0" />
            <Moon className="absolute h-4 w-4 rotate-90 scale-0 transition-all dark:rotate-0 dark:scale-100" />
        </Button>
    );
}
```

---

## Common UI Patterns

### Card Component

```typescript
<div className="rounded-lg border bg-card text-card-foreground shadow-sm">
    <div className="p-6">
        <h3 className="text-2xl font-semibold">Title</h3>
        <p className="text-sm text-muted-foreground">Description</p>
    </div>
</div>
```

### Button Variants

```typescript
// Primary
<button className="px-4 py-2 bg-primary text-primary-foreground rounded-md hover:bg-primary/90">
    Primary
</button>

// Secondary
<button className="px-4 py-2 bg-secondary text-secondary-foreground rounded-md hover:bg-secondary/80">
    Secondary
</button>

// Outline
<button className="px-4 py-2 border border-input rounded-md hover:bg-accent hover:text-accent-foreground">
    Outline
</button>

// Ghost
<button className="px-4 py-2 hover:bg-accent hover:text-accent-foreground rounded-md">
    Ghost
</button>
```

### Form Input

```typescript
<input
    type="text"
    className="flex h-10 w-full rounded-md border border-input bg-background px-3 py-2 text-sm ring-offset-background file:border-0 placeholder:text-muted-foreground focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:cursor-not-allowed disabled:opacity-50"
    placeholder="Enter text"
/>
```

### Badge

```typescript
<span className="inline-flex items-center rounded-full px-2.5 py-0.5 text-xs font-semibold bg-green-100 text-green-800">
    Success
</span>
```

---

## Hover and Focus States

### Interactive States

```typescript
// Hover
<div className="hover:bg-gray-100 hover:shadow-lg transition-all">
    Hover me
</div>

// Focus
<input className="focus:ring-2 focus:ring-blue-500 focus:border-transparent" />

// Active
<button className="active:scale-95 transition-transform">
    Click me
</button>

// Group hover (child responds to parent hover)
<div className="group hover:bg-gray-100">
    <span className="group-hover:text-blue-600">
        Hover parent to change me
    </span>
</div>
```

### Transition Classes

```typescript
// Basic transition
<div className="transition-colors duration-200">
    Smooth color transition
</div>

// Multiple properties
<div className="transition-all duration-300 ease-in-out">
    Smooth everything
</div>

// Transform
<button className="transform hover:scale-110 transition-transform">
    Zoom on hover
</button>
```

---

## When to Extract Classes

### Keep Inline When:

**✅ Simple, one-off styles**
```typescript
<div className="p-4 rounded-lg bg-white">
    Simple card
</div>
```

**✅ Clear and readable**
```typescript
<div className="flex items-center justify-between gap-4">
    Layout is obvious
</div>
```

### Extract to Component When:

**❌ Repeated 3+ times**
```typescript
// Instead of repeating:
<div className="p-4 rounded-lg border bg-card shadow-sm">...</div>
<div className="p-4 rounded-lg border bg-card shadow-sm">...</div>

// Extract to wrapper component:
<Card>...</Card>
```

**❌ Complex conditional logic**
```typescript
// Instead of giant className expressions:
className={cn(
    'base',
    condition1 && 'class1',
    condition2 && 'class2',
    // 10 more conditions...
)}

// Extract to wrapper with props
<OrderCard status="completed" priority="high" />
```

---

## Arbitrary Values

### Custom Values (Escape Hatch)

```typescript
// Use [brackets] for one-off custom values
<div className="w-[137px] top-[117px] bg-[#1da1f2]">
    Custom sizes
</div>

// Prefer Tailwind scale when possible
<div className="w-32 top-24 bg-blue-500">
    Standard scale
</div>
```

---

## Typography

### Font Styles

```typescript
// Size
text-xs, text-sm, text-base, text-lg, text-xl, text-2xl, text-3xl

// Weight
font-normal, font-medium, font-semibold, font-bold

// Color
text-foreground         // Primary text
text-muted-foreground   // Secondary text

// Line height
leading-none, leading-tight, leading-normal, leading-relaxed

// Example heading
<h1 className="text-3xl font-bold tracking-tight">
    Page Title
</h1>

// Example body
<p className="text-base text-muted-foreground leading-relaxed">
    Body text
</p>
```

---

## Layout Patterns

### Centered Container

```typescript
<div className="container mx-auto px-4">
    Centered with padding
</div>

// With max width
<div className="w-full max-w-4xl mx-auto px-4">
    Centered, capped at 1024px
</div>
```

### Sticky Header

```typescript
<header className="sticky top-0 z-50 bg-background border-b">
    Sticky navigation
</header>
```

### Sidebar Layout

```typescript
<div className="flex min-h-screen">
    <aside className="w-64 border-r bg-card">
        Sidebar
    </aside>
    <main className="flex-1 p-8">
        Main content
    </main>
</div>
```

---

## Performance Tips

### Avoid Dynamic Class Names

```typescript
// ❌ DON'T - Breaks PurgeCSS
const color = 'blue';
<div className={`text-${color}-500`}>Bad</div>

// ✅ DO - Full class names
const color = 'blue';
<div className={color === 'blue' ? 'text-blue-500' : 'text-red-500'}>
    Good
</div>
```

### Use safelist for Dynamic Classes

```javascript
// tailwind.config.ts
export default {
    safelist: [
        'text-blue-500',
        'text-red-500',
        'bg-green-500',
    ],
};
```

---

## Summary

**Tailwind Best Practices:**

1. **Utility-first** - Compose from small classes
2. **Mobile-first responsive** - Default styles, then `md:`, `lg:`
3. **cn() for conditionals** - Merge classes safely
4. **Theme colors** - Use CSS variables (bg-primary, text-foreground)
5. **Dark mode** - Use `dark:` prefix
6. **Extract wisely** - Inline until 3+ uses
7. **Avoid dynamic classes** - Use full class names
8. **Hover/focus states** - Built-in utility classes

**Common Patterns:**
```typescript
// Card
<div className="rounded-lg border bg-card p-6 shadow-sm">

// Button
<button className="px-4 py-2 rounded-md bg-primary text-primary-foreground hover:bg-primary/90">

// Input
<input className="w-full rounded-md border border-input px-3 py-2 focus:ring-2 focus:ring-ring" />

// Layout
<div className="flex items-center justify-between gap-4">
```

**See Also:**
- [shadcn-patterns.md](shadcn-patterns.md) - ShadCN wrapper components
- [component-patterns.md](component-patterns.md) - Component structure
- [complete-examples.md](complete-examples.md) - Full examples
