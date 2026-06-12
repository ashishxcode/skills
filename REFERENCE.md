# Agent Skills Documentation

Comprehensive guide for AI-assisted development with modern frontend best practices.

> **Source:** Based on [Vercel's React Best Practices](https://vercel.com/blog/introducing-react-best-practices) — 10+ years of production optimization knowledge encapsulated into 40+ rules across 8 categories, ordered by impact.

---

## Why This Matters

React performance work is usually **reactive**. A release goes out, the app feels slower, and the team starts chasing symptoms. That's expensive, and it's easy to optimize the wrong thing.

**The core insight:** Performance work fails when it starts too low in the stack.

If a request waterfall adds 600ms of waiting time, it doesn't matter how optimized your `useMemo` calls are. If you ship an extra 300KB of JavaScript on every page, shaving microseconds off a loop won't matter.

**This framework prioritizes fixes by impact:**

1. **Eliminate waterfalls** — CRITICAL (biggest user-facing wins)
2. **Reduce bundle size** — CRITICAL (affects every page load)
3. **Server-side performance** — HIGH
4. **Client-side fetching** — MEDIUM-HIGH
5. **Re-render optimization** — MEDIUM
6. **Rendering performance** — MEDIUM
7. **JavaScript performance** — LOW-MEDIUM
8. **Advanced patterns** — LOW

---

## Overview

This directory contains specialized skills for maintaining high-quality React applications. These skills are designed for AI agents (opencode, Claude Code, Codex) to follow when writing, reviewing, or refactoring code.

**Origin:** These practices come from real performance work on production codebases — Vercel's dashboard, Next.js, and other large-scale React applications.

### Real-World Examples

**Combining loop iterations:** A chat page was scanning the same list of messages eight separate times. Combined into a single pass — significant impact with thousands of messages.

**Parallelizing awaits:** An API was waiting for one database call to finish before starting the next, even though they didn't depend on each other. Running them simultaneously cut total wait in half.

**Lazy State Initialization:** A component was parsing a JSON config from `localStorage` on every render. Wrapping it in `useState(() => JSON.parse(...))` eliminated wasted work.

**Available Skills:**

| Skill              | Purpose                     | Trigger                              |
| ------------------ | --------------------------- | ------------------------------------ |
| `react-perf`       | 37 Vercel performance rules | Auto-applied                         |
| `review`           | Code review checklist        | "review this PR" or "review src/..." |
| `generate`         | Generate React components   | "generate a UserCard component"      |
| `refactor`         | Fix code smells             | "refactor this file"                 |
| `perf`             | Performance analysis        | "audit performance"                  |
| `commit`           | Organize conventional commits | "organize my commits"               |

---

## Using with Coding Agents

These best practices are packaged as Agent Skills that install into opencode, Claude Code, Codex, Cursor, and other coding agents.

```bash
npx skills add vercel-labs/agent-skills
```

When your agent spots cascading `useEffect` calls, heavy client-side imports, or other patterns, it references these practices and suggests fixes.

---

## Critical Rules (Always Apply)

### 1. Parallel Async Operations

```javascript
// ❌ Sequential - creates waterfalls
const users = await fetchUsers();
const posts = await fetchPosts();

// ✅ Parallel - eliminates waterfalls
const [users, posts] = await Promise.all([fetchUsers(), fetchPosts()]);
```

**Impact:** 2-10x improvement in data fetching.

### 2. Direct Imports (No Barrel Files)

```javascript
// ❌ Barrel import - loads entire library (200-800ms)
import { Button, Dialog } from "~/components/ui";

// ✅ Direct import - loads only what's needed
import { Button } from "~/components/ui/button";
import { Dialog } from "~/components/ui/dialog";
```

**Impact:** 15-70% faster dev boot, 28% faster builds, 40% faster cold starts.

### 3. Lazy State Initialization

```javascript
// ❌ Runs on every render
const [state, setState] = useState(computeInitialState());

// ✅ Runs only once
const [state, setState] = useState(() => computeInitialState());
```

### 4. Map/Set for O(1) Lookups

```javascript
// ❌ O(n) per lookup
const item = items.find((i) => i.id === id);

// ✅ O(1) per lookup
const itemMap = useMemo(() => new Map(items.map((i) => [i.id, i])), [items]);
const item = itemMap.get(id);
```

**Impact:** 1M operations → 2K operations for 1000 items × 1000 lookups.

### 5. Functional setState Updates

```javascript
// ❌ Requires dependency, causes stale closures
const addItem = useCallback(
  (newItem) => {
    setItems([...items, newItem]);
  },
  [items],
);

// ✅ Stable callback, always latest state
const addItem = useCallback((newItem) => {
  setItems((prev) => [...prev, newItem]);
}, []);
```

### 6. useCallback for Callbacks

```javascript
// ❌ Inline function recreated every render
<Button onClick={() => handleDelete(id)} />;

// ✅ Memoized callback
const handleClick = useCallback(() => handleDelete(id), [id]);
<Button onClick={handleClick} />;
```

### 7. Immutable Sorting

```javascript
// ❌ Mutates original array
const sorted = items.sort((a, b) => a - b);

// ✅ Creates new array
const sorted = items.toSorted((a, b) => a - b);
```

### 8. Explicit Conditionals

```javascript
// ❌ Renders "0" or "NaN" when count is falsy
{
  count && <Badge count={count} />;
}

// ✅ Explicit check
{
  count > 0 ? <Badge count={count} /> : null;
}
```

### 9. Early Returns

```javascript
// ❌ Deep nesting
if (data) {
  if (data.items) {
    return data.items.map(...);
  }
}

// ✅ Early exit
if (!data?.items) return [];
return data.items.map(...);
```

---

## Project Context

- **Framework:** React 19 + Vite 7
- **State:** Zustand (client) + TanStack Query (server)
- **Styling:** Tailwind CSS + Shadcn UI
- **Path alias:** `~` → `./src`
- **Package manager:** pnpm
- **Monorepo:** Turborepo

---

## File Structure

### Shared UI Components (`~/components/`)

```
src/components/
├── ui/                    # Shadcn primitives
│   ├── button.jsx
│   └── dialog.jsx
├── ComponentName/
│   ├── ComponentName.jsx  # Main component
│   └── index.js           # Barrel export
```

### Feature Components (`~/features/{feature}/`)

```
src/features/FeatureName/
├── api/                   # React Query hooks
│   └── useFeatureData.js
├── components/
│   ├── ComponentName/
│   │   ├── ComponentName.jsx
│   │   └── index.js
│   └── index.js           # Feature barrel export
├── hooks/                 # Feature-specific hooks
├── store/                 # Zustand stores
│   └── useFeatureStore.js
├── constants/
│   └── index.js
└── utils/
    └── index.js
```

---

## Component Templates

### Basic Component

```jsx
import { cn } from "~/utils/cn";

export function ComponentName({ className, children, ...props }) {
  return (
    <div className={cn("base-styles", className)} {...props}>
      {children}
    </div>
  );
}
```

### Stateful Component

```jsx
import { useState, useCallback } from "react";
import { cn } from "~/utils/cn";

export function ComponentName({ initialValue, onChange, className }) {
  const [value, setValue] = useState(initialValue);

  const handleChange = useCallback(
    (newValue) => {
      setValue(newValue);
      onChange?.(newValue);
    },
    [onChange],
  );

  return <div className={cn("base-styles", className)}>{/* content */}</div>;
}
```

### Zustand Store

```jsx
import { create } from "zustand";
import { immer } from "zustand/middleware/immer";

export const useFeatureStore = create(
  immer((set) => ({
    items: [],
    isLoading: false,

    setItems: (items) =>
      set((state) => {
        state.items = items;
      }),

    addItem: (item) =>
      set((state) => {
        state.items.push(item);
      }),

    reset: () =>
      set((state) => {
        state.items = [];
        state.isLoading = false;
      }),
  })),
);
```

### React Query Hook

```jsx
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import axios from "axios";

const QUERY_KEY = ["feature", "items"];

export function useFeatureData(id) {
  return useQuery({
    queryKey: [...QUERY_KEY, id],
    queryFn: async () => {
      const { data } = await axios.get(`/api/feature/${id}`);
      return data;
    },
    staleTime: 5 * 60 * 1000,
    enabled: !!id,
  });
}

export function useCreateFeature() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (payload) => {
      const { data } = await axios.post("/api/feature", payload);
      return data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: QUERY_KEY });
    },
  });
}
```

---

## Code Review Checklist

Every change must pass these checks:

### Critical (Block)

- [ ] No sequential async (use Promise.all)
- [ ] No barrel imports (direct paths only)
- [ ] useState uses lazy init for objects
- [ ] Map/Set for repeated lookups
- [ ] useCallback for all callbacks in JSX
- [ ] useMemo for expensive computations
- [ ] No .sort() mutation (use .toSorted())
- [ ] Early returns used
- [ ] No `&&` with potentially falsy values
- [ ] Functional setState for updates

### High Priority

- [ ] Proper Zustand selectors (not subscribing to entire store)
- [ ] React Query staleTime configured
- [ ] Error handling implemented
- [ ] Loading states managed
- [ ] Cleanup in useEffect

### Medium Priority

- [ ] No inline object/array creation in JSX props
- [ ] Key prop uses stable IDs (not index)
- [ ] Components sized appropriately (<300 lines)
- [ ] No prop drilling (>3 levels)

### Low Priority

- [ ] Semantic HTML elements
- [ ] ARIA labels where needed
- [ ] Responsive design considered
- [ ] Dark mode support (if applicable)

---

## Performance Optimization

### Priority Matrix

| Priority | Category               | Impact      | Rules         |
| -------- | ---------------------- | ----------- | ------------- |
| 1        | Eliminating Waterfalls | CRITICAL    | `async-*`     |
| 2        | Bundle Size            | CRITICAL    | `bundle-*`    |
| 3        | Server-Side            | HIGH        | `server-*`    |
| 4        | Client-Side            | MEDIUM-HIGH | `client-*`    |
| 5        | Re-render              | MEDIUM      | `rerender-*`  |
| 6        | Rendering              | MEDIUM      | `rendering-*` |
| 7        | JavaScript             | LOW-MEDIUM  | `js-*`        |
| 8        | Advanced               | LOW         | `advanced-*`  |

### Bundle Analysis

```bash
# Build and analyze
npm run build
npm run analyze

# Check for large dependencies
du -sh node_modules/* | sort -hr | head -20

# Find unused exports
npx unimported
```

### CSS Content-Visibility

```css
/* Defer off-screen rendering */
.list-item {
  content-visibility: auto;
  contain-intrinsic-size: 0 80px;
}
```

**Impact:** 10x faster initial render for long lists.

### Lazy Loading Routes

```jsx
import { lazy, Suspense } from "react";

const Dashboard = lazy(() => import("./pages/Dashboard"));
const Reports = lazy(() => import("./pages/Reports"));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/reports" element={<Reports />} />
      </Routes>
    </Suspense>
  );
}
```

---

## Security Checklist

### XSS Prevention

```jsx
// ❌ Dangerous
<div dangerouslySetInnerHTML={{ __html: userInput }} />

// ✅ Safe
<div>{userInput}</div>

// ✅ If HTML needed, sanitize
import DOMPurify from "dompurify";
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userInput) }} />
```

### Input Validation

```jsx
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

### Secret Detection

```bash
# Scan for exposed secrets
grep -r "apiKey\|api_key\|password\|token" --include="*.js" --include="*.jsx" src/
```

---

## Conventional Commits

### Format

```
<type>: <description>
```

### Types

| Type       | Description        |
| ---------- | ------------------ |
| `feat`     | New features       |
| `fix`      | Bug fixes          |
| `refactor` | Code restructuring |
| `docs`     | Documentation      |
| `test`     | Tests              |
| `perf`     | Performance        |
| `chore`    | Maintenance        |

### Rules

- Imperative mood: "add" not "added"
- Under 50 characters
- No period at end
- Lowercase after colon
- **NEVER** include AI attribution

### Example Workflow

```bash
# Natural commit order for new features:
feat: add Reports table constants and utilities
feat: add Reports table primitive cells
feat: add Reports table composite cells
feat: add Reports table column definitions
feat: add Reports table Zustand store
feat: add ReportsList component
refactor: migrate legacy report list to use Reports table
```

---

## Testing Guidelines

### Unit Tests

```jsx
import { render, screen, fireEvent } from "@testing-library/react";
import { describe, it, expect, jest } from "vitest";

describe("ItemCard", () => {
  it("renders campaign name", () => {
    render(<ItemCard name="Test Campaign" />);
    expect(screen.getByText("Test Campaign")).toBeInTheDocument();
  });

  it("calls onEdit when clicked", () => {
    const onEdit = jest.fn();
    render(<ItemCard onEdit={onEdit} />);
    fireEvent.click(screen.getByRole("button", { name: /edit/i }));
    expect(onEdit).toHaveBeenCalled();
  });
});
```

### Hook Tests

```jsx
import { renderHook, act } from "@testing-library/react";
import { describe, it, expect } from "vitest";

describe("useItemData", () => {
  it("fetches campaign data", async () => {
    const { result, waitFor } = renderHook(() => useItemData("123"));
    await waitFor(() => result.current.isSuccess);
    expect(result.current.data).toEqual(mockCampaign);
  });
});
```

### Coverage Target

- Aim for ≥80% coverage
- Focus on user interactions
- Mock external dependencies
- Keep tests deterministic

---

## Accessibility Checklist

- [ ] Semantic HTML elements used
- [ ] ARIA labels where needed
- [ ] Keyboard navigation works
- [ ] Focus management implemented
- [ ] Color contrast adequate (WCAG 2.2 AA)
- [ ] Touch targets ≥44x44px
- [ ] Screen reader compatible
- [ ] Focus states visible

---

## Refactoring Patterns

### Extract Custom Hook

```jsx
// Before: Mixed concerns
function Component() {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch("/api/data")
      .then((res) => res.json())
      .then(setData)
      .finally(() => setLoading(false));
  }, []);

  // ... render logic
}

// After: Separated concerns
function useData() {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch("/api/data")
      .then((res) => res.json())
      .then(setData)
      .finally(() => setLoading(false));
  }, []);

  return { data, loading };
}

function Component() {
  const { data, loading } = useData();
  // ... simpler render logic
}
```

### Derived State vs Stored State

```jsx
// ❌ Bad: Storing derived state
const [items, setItems] = useState([]);
const [filteredItems, setFilteredItems] = useState([]);

useEffect(() => {
  setFilteredItems(items.filter((i) => i.active));
}, [items]);

// ✅ Good: Compute during render
const [items, setItems] = useState([]);
const filteredItems = useMemo(() => items.filter((i) => i.active), [items]);
```

---

## Resources

### Official Documentation

- [React 19 Docs](https://react.dev/learn)
- [Vite Guide](https://vitejs.dev/guide/)
- [TanStack Query](https://tanstack.com/query/latest)
- [Zustand](https://docs.pmnd.rs/zustand/getting-started/introduction)
- [Tailwind CSS](https://tailwindcss.com/docs)

### Performance

- [Vercel: Package Import Optimization](https://vercel.com/blog/how-we-optimized-package-imports-in-next-js)
- [React Compiler](https://react.dev/learn/react-compiler)
- [Web Vitals](https://web.dev/vitals/)

### Tools

- [React DevTools](https://react.dev/learn/react-developer-tools)
- [Bundle Analyzer](https://www.npmjs.com/package/rollup-plugin-visualizer)
- [ESLint Plugin React Hooks](https://www.npmjs.com/package/eslint-plugin-react-hooks)

---

## Changelog

| Version | Date       | Changes                             |
| ------- | ---------- | ----------------------------------- |
| 1.0.0   | 2026-03-28 | Initial comprehensive documentation |

---

**Last Updated:** 2026-03-28
**Maintained By:** Agent Skills System
