---
name: ui
description: Shadcn/ui + Radix primitives component patterns — composition, customization, variants, layout. Use when creating or modifying UI components, adding variants, or establishing design patterns.
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# Design System

Component patterns for Shadcn/ui + Radix primitives with Tailwind CSS v4 in React 19 + Vite projects.

## Stack

- **Primitives:** Radix UI (via Shadcn/ui)
- **Styling:** Tailwind CSS v4 + `cn()` utility
- **Icons:** Lucide React
- **Theme:** CSS variables via `@workspace/ui`

## Component Patterns

### Primitive Composition

Compose Radix primitives into opinionated components. Never expose raw Radix APIs to feature code.

```jsx
// ✅ Shadcn-style composed component
import * as Dialog from "@radix-ui/react-dialog";
import { cn } from "@workspace/ui/lib/utils";
import { X } from "lucide-react";

export function Dialog({ open, onOpenChange, title, description, children }) {
  return (
    <Dialog.Root open={open} onOpenChange={onOpenChange}>
      <Dialog.Portal>
        <Dialog.Overlay className="fixed inset-0 bg-black/50" />
        <Dialog.Content className="fixed left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2 rounded-lg bg-background p-6 shadow-lg">
          <Dialog.Title className="text-lg font-semibold">
            {title}
          </Dialog.Title>
          {description && (
            <Dialog.Description className="text-sm text-muted-foreground">
              {description}
            </Dialog.Description>
          )}
          {children}
          <Dialog.Close className="absolute right-4 top-4">
            <X className="h-4 w-4" />
          </Dialog.Close>
        </Dialog.Content>
      </Dialog.Portal>
    </Dialog.Root>
  );
}
```

### Compound Components

Build high-level components from primitives for consistent UX patterns.

```jsx
// ✅ DeleteAlertDialog - a reusable destructive confirm
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
} from "~/components/ui/delete-alert-dialog";

export function DeleteCampaignDialog({ campaign, open, onOpenChange }) {
  return (
    <AlertDialog open={open} onOpenChange={onOpenChange}>
      <AlertDialogContent>
        <AlertDialogHeader>
          <AlertDialogTitle>Delete Campaign</AlertDialogTitle>
          <AlertDialogDescription>
            Are you sure you want to delete "{campaign?.name}"? This action
            cannot be undone.
          </AlertDialogDescription>
        </AlertDialogHeader>
        <AlertDialogFooter>
          <AlertDialogCancel>Cancel</AlertDialogCancel>
          <AlertDialogAction className="bg-destructive">
            Delete
          </AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  );
}
```

## Styling Patterns

### cn() Utility

```js
// @workspace/ui/lib/utils.js
import { clsx } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs) {
  return twMerge(clsx(inputs));
}
```

### Variant Props

```jsx
import { cn } from "@workspace/ui/lib/utils";

const badgeVariants = {
  default: "bg-primary text-primary-foreground",
  success: "bg-green-100 text-green-800",
  warning: "bg-yellow-100 text-yellow-800",
  danger: "bg-red-100 text-red-800",
  outline: "border border-border text-foreground",
};

export function Badge({ variant = "default", className, children }) {
  return (
    <span
      className={cn(
        "inline-flex items-center rounded-full px-2.5 py-0.5 text-xs font-medium",
        badgeVariants[variant],
        className,
      )}
    >
      {children}
    </span>
  );
}
```

### Conditional Classes

```jsx
<div
  className={cn(
    "rounded-lg border p-4 transition-colors",
    isSelected && "border-primary bg-primary/5",
    isDisabled && "opacity-50 cursor-not-allowed",
    className,
  )}
/>
```

### Responsive Design

```jsx
// Mobile-first responsive classes
<div className="grid grid-cols-1 gap-4 md:grid-cols-2 lg:grid-cols-3" />

// Visibility
<div className="hidden md:block" />   {/* Desktop only */}
<div className="block md:hidden" />   {/* Mobile only */}
```

## Layout Components

### Page Header

```jsx
export function PageHeader({ title, description, actions }) {
  return (
    <div className="flex items-center justify-between mb-6">
      <div>
        <h1 className="text-2xl font-semibold tracking-tight">{title}</h1>
        {description && (
          <p className="text-sm text-muted-foreground mt-1">{description}</p>
        )}
      </div>
      {actions && <div className="flex items-center gap-2">{actions}</div>}
    </div>
  );
}
```

### Card with Header

```jsx
export function Card({ title, description, children, className }) {
  return (
    <div className={cn("rounded-lg border bg-card p-6", className)}>
      {title && (
        <div className="mb-4">
          <h3 className="font-semibold">{title}</h3>
          {description && (
            <p className="text-sm text-muted-foreground">{description}</p>
          )}
        </div>
      )}
      {children}
    </div>
  );
}
```

## Icon Conventions

```jsx
// ✅ Lucide named import (tree-shakeable)
import { Search, Plus, MoreHorizontal } from "lucide-react";

// ✅ Icon button
<Button variant="ghost" size="icon" aria-label="Search">
  <Search className="h-4 w-4" />
</Button>

// ✅ Icon with label
<Button>
  <Plus className="h-4 w-4 mr-2" />
  New Campaign
</Button>
```

## Theme

```css
/* CSS variables for theming — defined on :root and .dark */
:root {
  --background: 0 0% 100%;
  --foreground: 222.2 84% 4.9%;
  --primary: 222.2 47.4% 11.2%;
  --primary-foreground: 210 40% 98%;
  --destructive: 0 84.2% 60.2%;
  --muted: 210 40% 96.1%;
  --muted-foreground: 215.4 16.3% 46.9%;
  --border: 214.3 31.8% 91.4%;
  --ring: 222.2 84% 4.9%;
}

.dark {
  --background: 222.2 84% 4.9%;
  --foreground: 210 40% 98%;
  /* ... */
}
```

## Verification

```bash
pnpm build   # ensure all imports resolve
pnpm lint    # ensure consistent patterns
```

## Best Practices

- **Reuse before hand-rolling** — check if a shared component already exists
- **Direct imports** — never barrel-import from `@workspace/ui/components`
- **Compose primitives** — don't expose raw Radix APIs outside the component file
- **One variant system** — use the `variant` prop pattern consistently
- **Avoid inline styles** — use Tailwind classes + `cn()` exclusively
- **Dark mode** — support via CSS variables, no manual class toggling
