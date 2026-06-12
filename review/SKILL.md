---
name: review
description: Review React/Vite frontend code for quality, performance, security, and best practices. Use when reviewing PRs, code changes, or providing feedback.
allowed-tools: Read, Grep, Glob, Bash(git:*)
---

# Frontend Code Review

Comprehensive code review checklist for React 19 + Vite + Turborepo projects.

## Review Workflow

1. **Understand Context**
   - Read the PR description or change summary
   - Check related files and dependencies
   - Understand the feature/fix purpose

2. **Static Analysis**
   - Check for obvious issues (syntax, imports, unused code)
   - Verify naming conventions
   - Check file organization

3. **Logic Review**
   - Verify business logic correctness
   - Check edge cases
   - Verify error handling

4. **Performance Check**
   - Look for re-render issues
   - Check bundle impact
   - Verify data fetching patterns

5. **Security Scan**
   - Check for XSS vulnerabilities
   - Verify input validation
   - Check for exposed secrets

6. **Final Verification**
   - Run lint/typecheck if available
   - Check for missing tests
   - Verify accessibility

---

## Critical Checks (Must Pass)

### 1. React Performance

**Re-render Issues**

```jsx
// ❌ BAD: Creates new object every render
<Component config={{ timeout: 5000 }} />;

// ✅ GOOD: Memoized or hoisted
const CONFIG = { timeout: 5000 };
<Component config={CONFIG} />;
```

**Missing Dependencies**

```jsx
// ❌ BAD: Missing dependency
useEffect(() => {
  fetchUser(userId);
}, []); // userId missing!

// ✅ GOOD: All dependencies included
useEffect(() => {
  fetchUser(userId);
}, [userId]);
```

**State Management**

```jsx
// ❌ BAD: Subscribing to entire store
const store = useUserStore();

// ✅ GOOD: Select specific slices
const user = useUserStore((state) => state.user);
const updateUser = useUserStore((state) => state.updateUser);
```

### 2. Import Patterns

**Direct Imports Required**

```jsx
// ❌ BAD: Barrel import
import { Button, Dialog } from "~/components/ui";

// ✅ GOOD: Direct imports
import { Button } from "~/components/ui/button";
import { Dialog } from "~/components/ui/dialog";
```

**No Unused Imports**

```bash
# Check for unused imports
npx unimported
# or
npx depcheck
```

### 3. Async Patterns

**Parallel Fetching**

```jsx
// ❌ BAD: Sequential (waterfall)
const user = await fetchUser();
const posts = await fetchPosts(user.id);

// ✅ GOOD: Parallel where possible
const [user, posts] = await Promise.all([fetchUser(), fetchPosts()]);
```

**Error Handling**

```jsx
// ❌ BAD: No error handling
data = await fetchData();

// ✅ GOOD: Try-catch with user feedback
try {
  const data = await fetchData();
} catch (error) {
  showToast({ type: "error", message: error.message });
  console.error("Fetch failed:", error);
}
```

### 4. State Initialization

**Lazy Init for Objects**

```jsx
// ❌ BAD: Runs on every render
const [state, setState] = useState(computeInitialState());

// ✅ GOOD: Lazy initialization
const [state, setState] = useState(() => computeInitialState());
```

### 5. Conditional Rendering

**Explicit Conditionals**

```jsx
// ❌ BAD: Can render "0" or NaN
{
  count && <Badge />;
}

// ✅ GOOD: Explicit check
{
  count > 0 ? <Badge /> : null;
}
```

---

## Component Review Checklist

### Structure & Organization

- [ ] Component is in correct directory (ui/ vs features/)
- [ ] File follows naming convention (PascalCase.jsx)
- [ ] Barrel export exists in index.js
- [ ] No unused props or variables
- [ ] Props are destructured properly

### Logic & State

- [ ] State is minimal and derived where possible
- [ ] useEffect dependencies are complete
- [ ] No prop drilling (use context/store if >2 levels)
- [ ] Side effects are properly cleaned up
- [ ] Loading/error states handled

### Performance

- [ ] Expensive computations use useMemo
- [ ] Callbacks use useCallback (especially in JSX)
- [ ] Components that don't need updates use React.memo
- [ ] No inline object/array creation in JSX
- [ ] Map/Set used for repeated lookups

### Styling

- [ ] Uses Tailwind classes (not inline styles)
- [ ] Uses cn() utility for conditional classes
- [ ] Responsive design considered
- [ ] Dark mode support (if applicable)
- [ ] No arbitrary values without justification

### Accessibility

- [ ] Semantic HTML elements used
- [ ] ARIA labels where needed
- [ ] Keyboard navigation works
- [ ] Focus management implemented
- [ ] Color contrast adequate

---

## API/Data Layer Review

### TanStack Query Patterns

```jsx
// ✅ GOOD: Proper query configuration
export function useItemData(id) {
  return useQuery({
    queryKey: ["campaign", id],
    queryFn: async () => {
      const { data } = await axios.get(`/api/campaigns/${id}`);
      return data;
    },
    staleTime: 5 * 60 * 1000, // 5 minutes
    enabled: !!id, // Only fetch when id exists
  });
}
```

**Checklist:**

- [ ] Proper query keys used
- [ ] Stale time configured
- [ ] Error handling implemented
- [ ] Loading states managed
- [ ] Cache invalidation on mutations

### Zustand Store Patterns

```jsx
// ✅ GOOD: Immer for mutations, selectors for subscriptions
export const useCampaignStore = create(
  immer((set, get) => ({
    campaigns: [],
    selectedId: null,

    // Action with immer
    updateCampaign: (id, updates) =>
      set((state) => {
        const campaign = state.campaigns.find((c) => c.id === id);
        if (campaign) {
          Object.assign(campaign, updates);
        }
      }),

    // Computed with selector
    get selectedCampaign() {
      return get().campaigns.find((c) => c.id === get().selectedId);
    },
  })),
);
```

**Checklist:**

- [ ] Immer middleware used for mutations
- [ ] Actions don't return new state directly
- [ ] Selectors are specific (not subscribing to entire store)
- [ ] Computed values use getters
- [ ] Store is feature-scoped, not global

---

## Security Review

### XSS Prevention

```jsx
// ❌ BAD: Dangerous HTML injection
<div dangerouslySetInnerHTML={{ __html: userInput }} />

// ✅ GOOD: Text content only
<div>{userInput}</div>

// ✅ GOOD: If HTML needed, sanitize first
import DOMPurify from 'dompurify';
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userInput) }} />
```

### Input Validation

```jsx
// ❌ BAD: No validation
const handleSubmit = (data) => {
  api.save(data);
};

// ✅ GOOD: Schema validation with Zod
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

Check for:

- Hardcoded API keys
- Firebase configs with exposed credentials
- JWT tokens in code
- `.env` files committed
- Console logs with sensitive data

```bash
# Scan for secrets
grep -r "apiKey\|api_key\|password\|token" --include="*.js" --include="*.jsx" src/
```

---

## Testing Review

### Unit Tests

```jsx
// ✅ GOOD: Testing component behavior
import { render, screen, fireEvent } from "@testing-library/react";

describe("ItemCard", () => {
  it("renders campaign name", () => {
    render(<ItemCard name="Test Campaign" />);
    expect(screen.getByText("Test Campaign")).toBeInTheDocument();
  });

  it("calls onEdit when edit button clicked", () => {
    const onEdit = jest.fn();
    render(<ItemCard onEdit={onEdit} />);
    fireEvent.click(screen.getByRole("button", { name: /edit/i }));
    expect(onEdit).toHaveBeenCalled();
  });
});
```

**Checklist:**

- [ ] Component has test file
- [ ] Tests cover user interactions
- [ ] Mocked external dependencies
- [ ] Assertions are specific
- [ ] Tests are deterministic

### Hook Tests

```jsx
// ✅ GOOD: Testing custom hooks
import { renderHook, act } from "@testing-library/react";

describe("useItemData", () => {
  it("fetches campaign data", async () => {
    const { result, waitFor } = renderHook(() => useItemData("123"));

    await waitFor(() => result.current.isSuccess);

    expect(result.current.data).toEqual(mockCampaign);
  });
});
```

---

## Common Issues to Flag

### High Priority

1. **Memory leaks** - Missing cleanup in useEffect
2. **Infinite loops** - Incorrect useEffect dependencies
3. **XSS vulnerabilities** - Unsanitized HTML
4. **Exposed secrets** - API keys in code
5. **Broken error handling** - Silent failures

### Medium Priority

1. **Performance issues** - Unnecessary re-renders
2. **Accessibility problems** - Missing ARIA labels
3. **Missing tests** - No coverage for new features
4. **Inconsistent naming** - Doesn't follow conventions
5. **Prop drilling** - Passing props through many levels

### Low Priority

1. **Console warnings** - React strict mode warnings
2. **Unused imports** - Dead code
3. **Inconsistent formatting** - Prettier can fix
4. **Missing JSDoc** - For complex functions
5. **Suboptimal patterns** - Could be cleaner

---

## Review Output Format

```markdown
## Code Review: PR #XXX

### Summary

Brief description of changes and overall quality.

### Critical Issues (Must Fix)

1. **Memory leak in useEffect** - Missing cleanup function
   - File: `src/features/Campaign/hooks/usePolling.js:23`
   - Fix: Add return () => clearInterval(intervalId)

2. **XSS vulnerability** - Unsanitized user input
   - File: `src/components/RichText/Editor.jsx:45`
   - Fix: Use DOMPurify before setting innerHTML

### Suggestions (Nice to Have)

1. **Performance**: Consider memoizing expensive computation
2. **Accessibility**: Add aria-label to icon button
3. **Testing**: Add test for error state

### Approved?

- [ ] Approved
- [ ] Approved with minor changes
- [ ] Changes requested
```

---

## Commands for Review

```bash
# Check for TypeScript errors
pnpm typecheck

# Run linter
pnpm lint

# Check test coverage
pnpm test -- --coverage

# Build to catch bundle issues
pnpm build

# Check for unused exports
npx unimported
```

## Usage

When asked to review code:

1. Read all changed files
2. Run through the Critical Checks
3. Check Component/API/Security checklists
4. Provide structured feedback with severity levels
5. Suggest specific fixes with code examples
