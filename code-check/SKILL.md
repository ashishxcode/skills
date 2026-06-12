---
name: code-check
description: Run pre-commit quality checks — ESLint, Prettier, accessibility (WCAG 2.2, ARIA, keyboard), security (secrets, XSS), and React anti-patterns. Use before committing, pushing, or opening a PR.
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# Code Check

Pre-commit quality gate for React 19 + Vite projects. Run this skill before any commit or PR to catch issues early.

When an agent is asked to prepare or review code before a commit, use this skill to check all quality dimensions in one pass.

## Quick Run

```bash
pnpm format:check    # Prettier
pnpm lint            # ESLint
pnpm build           # Build check
pnpm vitest run      # Tests (if they exist)
```

## 1. ESLint & Formatting

### Run checks

```bash
pnpm lint                  # ESLint with max-warnings 0
pnpm format:check          # Prettier check
```

### Auto-fix

```bash
pnpm format                # Prettier write
pnpm lint --fix            # ESLint auto-fix
```

### Common lint issues

```jsx
// ❌ Missing React Hook deps
useEffect(() => { fetch(id); }, []);        // missing: id

// ❌ Unused imports
import { useState } from "react";           // never used

// ✅ Fix deps
useEffect(() => { fetch(id); }, [id]);
```

## 2. Accessibility (WCAG 2.2 AA)

### ARIA & Semantic HTML

```jsx
// ❌ Non-semantic clickable
<div onClick={handleClick}>Submit</div>

// ✅ Semantic button
<button onClick={handleClick}>Submit</button>

// ❌ Missing label on interactive element
<input type="email" placeholder="Email" />

// ✅ Associated label
<label htmlFor="email">Email</label>
<input id="email" type="email" />

// ❌ Icon button without label
<button><XIcon /></button>

// ✅ Label for screen readers
<button aria-label="Close"><XIcon /></button>
```

### Keyboard Navigation

```jsx
// ❌ Missing keyboard handler
<div onClick={handleClick} role="button">Save</div>

// ✅ Keyboard support
<div
  onClick={handleClick}
  onKeyDown={(e) => { if (e.key === "Enter") handleClick(); }}
  role="button"
  tabIndex={0}
>
  Save
</div>

// ❌ Focus outline removed
*:focus { outline: none; }

// ✅ Visible focus ring
*:focus-visible { outline: 2px solid var(--color-ring); outline-offset: 2px; }
```

### Focus Management

```jsx
// ❌ No focus trap in modal
function Dialog({ open }) {
  if (!open) return null;
  return <div role="dialog">{children}</div>;
}

// ✅ Focus management
function Dialog({ open, onClose }) {
  const closeRef = useRef(null);

  useEffect(() => {
    if (open) closeRef.current?.focus();
  }, [open]);

  useEffect(() => {
    const handleKey = (e) => {
      if (e.key === "Escape") onClose();
    };
    if (open) window.addEventListener("keydown", handleKey);
    return () => window.removeEventListener("keydown", handleKey);
  }, [open, onClose]);

  if (!open) return null;

  return (
    <div role="dialog" aria-modal="true">
      {children}
      <button ref={closeRef} onClick={onClose}>Close</button>
    </div>
  );
}
```

### Color & Contrast

- Text: minimum 4.5:1 contrast ratio
- Large text (18px+ / 14px bold+): minimum 3:1
- Focus indicators: 3:1 against adjacent colors

## 3. React Anti-Patterns

### Barrel Imports

```jsx
// ❌ Barrel import - loads entire library
import { Button, Dialog } from "~/components/ui";

// ✅ Direct import - loads only what's needed
import { Button } from "~/components/ui/button";
import { Dialog } from "~/components/ui/dialog";
```

### Conditional Rendering

```jsx
// ❌ Renders "0" when count is 0
{count && <Badge count={count} />}

// ✅ Explicit check
{count > 0 ? <Badge count={count} /> : null}
```

### Async Patterns

```jsx
// ❌ Sequential waterfalls
const user = await fetchUser();
const posts = await fetchPosts(user.id);

// ✅ Parallel independent operations
const [user, posts] = await Promise.all([fetchUser(), fetchPosts()]);
```

### State Initialization

```jsx
// ❌ Runs on every render
const [state, setState] = useState(computeInitial());

// ✅ Lazy initialization - runs once
const [state, setState] = useState(() => computeInitial());
```

### Immutable Updates

```jsx
// ❌ Mutates original array (breaks React state)
const sorted = items.sort((a, b) => a - b);

// ✅ Creates new array
const sorted = items.toSorted((a, b) => a - b);
```

### Store Subscriptions

```jsx
// ❌ Subscribes to entire store
const store = useStore();

// ✅ Selective subscription
const items = useStore((s) => s.items);
const { items, count } = useStore(
  (s) => ({ items: s.items, count: s.count }),
  shallow,
);
```

## 4. Security

### XSS Prevention

```jsx
// ❌ Dangerous HTML injection
<div dangerouslySetInnerHTML={{ __html: userInput }} />

// ✅ Safe text content
<div>{userInput}</div>

// ✅ If HTML is required, sanitize first
import DOMPurify from "dompurify";
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userInput) }} />
```

### Secrets Detection

```bash
# Scan for exposed secrets
grep -r "apiKey\|api_key\|password\|secret\|token" \
  --include="*.{js,jsx,ts,tsx}" \
  --exclude-dir=node_modules \
  --exclude-dir=.next \
  src/
```

### Input Validation

```jsx
// ❌ No validation
const handleSubmit = (data) => api.save(data);

// ✅ Zod schema validation
import { z } from "zod";
const schema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
});
const handleSubmit = (data) => {
  const valid = schema.parse(data);
  api.save(valid);
};
```

## 5. Build & Type Verification

```bash
pnpm build         # Must exit 0
pnpm vitest run    # Must pass (if tests exist)
```

## Pre-Commit Checklist

- [ ] `pnpm format:check` — Prettier passes
- [ ] `pnpm lint` — ESLint passes (max-warnings 0)
- [ ] `pnpm build` — Build succeeds
- [ ] Semantic HTML used (no `<div>` where `<button>`, `<nav>`, etc. works)
- [ ] All interactive elements keyboard-accessible
- [ ] Labels provided for form inputs and icon buttons
- [ ] No barrel imports for UI components
- [ ] No `&&` with falsy numeric values
- [ ] No secrets, API keys, or tokens in code
- [ ] Input validated with zod schemas
- [ ] No sequential awaits without data dependency
