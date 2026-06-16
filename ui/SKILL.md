---
name: ui
description: How to approach UI work (design mindset) plus Shadcn/ui + Radix component patterns — composition, variants, layout. Use for any UI/screen work — building, redesigning, or reviewing — even when no Figma/mockup exists.
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# Design System

Component patterns for Shadcn/ui + Radix primitives with Tailwind CSS v4 in React 19 + Vite projects. Read the **Design Mindset** first (how to approach a screen), then the patterns below (what to build with).

## Design Mindset (work in this order; never skip step 1)

A Figma/mockup often won't exist — when it doesn't, you are the designer: derive the screen from intent + data + the design system. Never ship the raw/default rendering as if "no mockup" means "no opinion."

1. **Anchor to the real data contract.** Use the actual API response, not an assumed one. Enumerate every field's states: null / empty / loading / pending / in-progress / skipped / multiple-versions / error / complete. If you can't list a field's states, you're guessing — go find out. Most UI bugs are a screen built against an imagined shape.

2. **State the design intent (this replaces the missing Figma).** One sentence: what the screen is for, who reads it, what's primary vs secondary vs meta. That sentence is your spec — design to it. If intent is genuinely ambiguous, ask one sharp question instead of guessing the layout; short directional asks ("richer", "previous UI") need a scope question before rebuilding.

3. **Design from the states, not decoration.** Every state from step 1 gets a defined, intentional rendering — including empty, skipped, and loading. A blank or default-looking state is unfinished work, not an edge case. Lead with the primary info; push meta (timestamps, versions, ids) to the edges or into the item's own header.

4. **Reuse the system before inventing.** Reach for an existing design-system component before dropping to a raw element or a bare primitive. Match the spacing scale (even steps — `p-2`/`p-4`/`gap-2`, not `p-3`/`p-1.5`/`gap-3`), the standard type scale (`text-xs`/`sm`/`base`/`lg`, never arbitrary `text-[13px]`), and the theme color tokens. No one-off colors, no arbitrary pixel values.

5. **Color carries meaning, not decoration.** A status maps to a shared tone (reuse one status palette / `Badge` variants so the same status reads the same everywhere); give AI/automated surfaces a single shared accent. Neutral by default; add color only where it signals something.

6. **Name and place things truthfully.** Meta lives with its object (a timestamp belongs in the item header, not floating above it). Label ambiguous content ("Caption", not naked text). If a viewer would ask "what is this?", the design failed. Same for code identifiers — the name should tell the truth.

7. **Build a system, not a screen.** If two places show the same thing (a preview, a status badge, an empty state), it is one component reused — never two copies that can drift. Co-locate the tightly-coupled pieces; derive values from a single source so presence/labels/colors can't fall out of sync.

8. **Root-cause, then sweep.** When something's wrong, ask _why_ it happened and whether the same flaw exists elsewhere. Fix the cause (one source of truth), grep for siblings of the bug, and don't patch the symptom (no sentinel resets, no lint-disable to dodge it).

9. **Verify across states, then leave a clean trail.** Confirm it renders for each state from step 1 (lint; build when imports/structure changed). Organize the work into logical, self-contained commits — one concern each.

**Mindset in one line:** _Real data first. No mockup is not an excuse — state the intent and design to it. Every state covered. Reuse over reinvention. Name and place things truthfully. Fix causes, not symptoms._

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
- **Even spacing scale** — `p-2`/`p-4`/`gap-2`/`gap-4`; avoid odd/half steps (`p-3`, `p-1.5`)
- **Standard type scale** — `text-xs`/`sm`/`base`/`lg`; never arbitrary `text-[13px]`
- **Color = meaning** — status → shared tone/`Badge` variant; one accent for AI; neutral otherwise
- **Cover every state** — empty / loading / skipped / error, not just the happy path
- **Single source of truth** — derive presence/labels/colors from one helper so they can't drift
- **Dark mode** — support via CSS variables, no manual class toggling
