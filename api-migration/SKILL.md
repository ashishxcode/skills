---
name: api-migration
description: Migrate legacy imperative API hooks (useXxxAPI.js with hand-rolled async + loading state) into a feature's api/ folder using TanStack Query — queries, mutations, and centralized query keys. Use when migrating legacy data-fetching, paying down src/api debt, or moving an API into a feature.
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git:*)
---

# Migrate API Skill

Moves a legacy imperative API hook (`src/api/useXxxAPI.js` — async functions + manual `isLoading`/`error` state) into its owning feature's `api/` folder in the project's TanStack Query shape. Behavior-preserving: same endpoints, params, payloads.

## When to use

"Migrate `useXxxAPI`", paying down `src/api/` legacy debt, or pulling data-fetching into a feature.

## Migration Order

Cheap and safe first:

1. **Dead files first** — 0 consumers → just delete.
   ```bash
   grep -rl "useWidgetAPI" src/ | grep -v "src/api/useWidgetAPI.js"   # empty = dead → git rm
   ```
2. **Leaf files next** — don't import other `src/api/*` hooks.
3. **Hub files last** — e.g. a `useActivityAPI` that other legacy hooks import; migrating it early breaks them.

Map every consumer + the functions each uses before touching code:
```bash
grep -rn "useWidgetAPI" src/features src/pages src/components
```

## The Transformation

### 1. Centralize query keys

```js
// ✅ features/<owner>/api/keys.js — single source of truth
export const widgetKeys = {
  all: ["widgets"],
  list: (params) => ["widgets", "list", params],
  detail: (id) => ["widgets", "detail", id],
};
```

### 2. Reads → query hooks

```js
// ❌ Before: imperative, manual state, fires on call
function useWidgetAPI() {
  const getWidget = async (id) => {
    const { data } = await axios.get(`/widgets/${id}`);
    return data;
  };
  return { getWidget };
}

// ✅ After: features/<owner>/api/queries/useWidgetQuery.js
export const getWidgetQueryOptions = (id) => ({
  queryKey: widgetKeys.detail(id),
  queryFn: ({ signal }) => getWidget(id, signal),
  staleTime: 5 * 60 * 1000,
  enabled: !!id,
});
export const useWidgetQuery = (id, options) =>
  useQuery({ ...getWidgetQueryOptions(id), ...options });
```

### 3. Writes → mutation hooks (factory + invalidation)

```js
// ❌ Before: caller must remember to refetch → stale UI
const { createWidget } = useWidgetAPI();
await createWidget(payload);

// ✅ After: features/<owner>/api/mutations/useCreateWidgetMutation.js
export const useCreateWidgetMutation = () => {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: createWidget,
    onSuccess: () => queryClient.invalidateQueries({ queryKey: widgetKeys.all }),
  });
};
```
Centralize invalidation in `onSuccess` so callers never forget to refresh. Preserve any side effects (logging, toasts) there too. If your project has a mutation factory that *enforces* an `invalidates` declaration, prefer it.

### 4. Fix error handling on the way

```js
// ❌ Legacy anti-pattern — turns failures into fake successes
catch (error) {
  return error.response.data;
}

// ✅ Throw so Query surfaces isError (flag this as a deliberate behavior change)
catch (error) {
  console.error("API Error:", error);
  throw error;
}
```

### 5. Rewire consumers — delete the hand-rolled state

```jsx
// ❌ Before
const { getWidget } = useWidgetAPI();
const [widget, setWidget] = useState(null);
const [loading, setLoading] = useState(true);
useEffect(() => {
  getWidget(id).then(setWidget).finally(() => setLoading(false));
}, [id]);

// ✅ After — loading/error come from Query
const { data: widget, isPending, isError } = useWidgetQuery(id);
```

### 6. Delete the legacy file

```bash
git rm src/api/useWidgetAPI.js
grep -rn "useWidgetAPI" src/   # must be empty
```

## Rules

- `src/api/` is **frozen** — only migrate out of it, never add to it.
- **One legacy file per change** — no big-bang migrations. Update `MIGRATION.md` if the repo tracks it.
- **Behavior-preserving** — same endpoints/params/payloads. The error-handling fix is the one allowed behavior change; call it out.

## Verify

```bash
pnpm build
npx eslint <touched-paths> --max-warnings 0
grep -rn "useWidgetAPI" src/   # zero references remain
```

## Reference

Target shape: a feature whose API already lives in `features/<feature>/api/{queries,mutations,keys.js}`. Legacy source lives in `src/api/useXxxAPI.js`.
