---
name: perf
description: Analyze React application performance including bundle size, component rendering, and Core Web Vitals. Use when checking performance or optimizing.
allowed-tools: Bash, Read, Glob, Grep
---

# Performance Audit

Comprehensive performance analysis for React Vite applications.

## Audit Checklist

### 1. Bundle Analysis
```bash
npm run build
npm run analyze  # If available
```

Check for:
- Large dependencies (>100KB)
- Duplicate packages
- Unused exports
- Missing tree-shaking

### 2. React Performance Issues

**Component Re-renders**
- Missing `React.memo()` on expensive components
- Inline object/array props causing re-renders
- Missing dependency arrays in `useEffect`, `useMemo`, `useCallback`

**State Management**
- Zustand selectors not using shallow equality
- Large context providers causing cascading updates
- Missing React Query stale time configuration

### 3. Code Splitting Opportunities

Look for:
```jsx
// Should use React.lazy for route-level splitting
import HeavyComponent from './HeavyComponent';

// Better:
const HeavyComponent = React.lazy(() => import('./HeavyComponent'));
```

Large features that should be lazy-loaded:
- Dashboard modules
- Admin panels
- Chart libraries
- PDF generators

### 4. Image Optimization

Check for:
- Missing width/height attributes
- Non-WebP formats for large images
- Missing lazy loading on below-fold images
- Unoptimized SVGs

### 5. Core Web Vitals

**LCP (Largest Contentful Paint)**
- Preload critical assets
- Optimize hero images
- Reduce server response time

**FID (First Input Delay)**
- Break up long tasks
- Defer non-critical JavaScript
- Use `requestIdleCallback` for low-priority work

**CLS (Cumulative Layout Shift)**
- Set explicit dimensions on images/embeds
- Avoid inserting content above existing content
- Use CSS containment

## Commands

```bash
# Build and check bundle size
npm run build

# Analyze bundle (after adding visualizer)
npm run build -- --mode analyze

# Check for large dependencies
du -sh node_modules/* | sort -hr | head -20
```

## Common Fixes

### Memo expensive components
```jsx
const ExpensiveList = React.memo(({ items }) => {
  return items.map(item => <Item key={item.id} {...item} />);
});
```

### Use proper Zustand selectors
```jsx
// Bad - subscribes to entire store
const store = useStore();

// Good - subscribes to specific slice
const items = useStore(state => state.items);
const { items, count } = useStore(
  state => ({ items: state.items, count: state.count }),
  shallow
);
```

### Lazy load routes
```jsx
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Reports = lazy(() => import('./pages/Reports'));

<Suspense fallback={<Loading />}>
  <Routes>
    <Route path="/dashboard" element={<Dashboard />} />
    <Route path="/reports" element={<Reports />} />
  </Routes>
</Suspense>
```

### Configure React Query
```jsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000, // 5 minutes
      cacheTime: 10 * 60 * 1000, // 10 minutes
    },
  },
});
```

## Output Format

Provide findings as:

| Category | Issue | Impact | Fix |
|----------|-------|--------|-----|
| Bundle | lodash full import | +70KB | Use lodash-es or individual imports |
| React | Missing memo on CreatorList | High re-renders | Add React.memo() |
| Images | hero.png is 2MB | Slow LCP | Convert to WebP, resize |
