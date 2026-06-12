---
name: errors
description: Error boundaries, API error normalization, Sentry integration, user-facing error messages, and recovery patterns for React 19 + Vite projects. Use when adding or improving error handling.
allowed-tools: Read, Write, Edit, Glob, Grep
---

# Error Handling

Error handling patterns for React 19 + Vite projects using TanStack Query + Zustand + Sentry.

## Architecture

```
src/
├── components/
│   └── ErrorBoundary/
│       ├── ErrorBoundary.jsx
│       └── index.js
├── lib/
│   └── createMutation.js     # mutation factory with error handling
├── utils/
│   └── cxaxios.js            # axios instance with interceptors
└── hooks/
    └── useErrorHandler.js    # centralized error handler hook
```

## Error Boundary

```jsx
// src/components/ErrorBoundary/ErrorBoundary.jsx
import { Component } from "react";

export class ErrorBoundary extends Component {
  state = { error: null };

  static getDerivedStateFromError(error) {
    return { error };
  }

  componentDidCatch(error, info) {
    // Log to Sentry or monitoring
    console.error("Caught by boundary:", error, info);
  }

  handleReset = () => this.setState({ error: null });

  render() {
    if (this.state.error) {
      return this.props.fallback ?? (
        <div role="alert" className="p-8 text-center">
          <h2 className="text-lg font-semibold">Something went wrong</h2>
          <p className="text-muted-foreground mt-2">
            {this.state.error.message || "An unexpected error occurred"}
          </p>
          <Button onClick={this.handleReset} className="mt-4">
            Try Again
          </Button>
        </div>
      );
    }

    return this.props.children;
  }
}
```

### Usage

```jsx
// Route-level
<Route
  path="/campaigns"
  element={
    <ErrorBoundary>
      <CampaignListPage />
    </ErrorBoundary>
  }
/>

// Feature-level
<ErrorBoundary fallback={<CampaignErrorState />}>
  <CampaignDetail />
</ErrorBoundary>
```

## Axios Error Interceptors

```js
// src/utils/cxaxios.js
import axios from "axios";

const cxaxios = axios.create({ baseURL: "/api" });

// Normalize errors into a consistent shape
cxaxios.interceptors.response.use(
  (response) => response,
  (error) => {
    const normalized = {
      message: "An unexpected error occurred",
      status: error.response?.status ?? 0,
      code: error.response?.data?.code ?? "UNKNOWN",
      details: error.response?.data?.details ?? null,
      original: error,
    };

    if (error.response) {
      switch (error.response.status) {
        case 400:
          normalized.message =
            error.response.data?.message || "Invalid request";
          break;
        case 401:
          normalized.message = "Session expired. Please log in again.";
          break;
        case 403:
          normalized.message = "You don't have permission to do this.";
          break;
        case 404:
          normalized.message = "Resource not found.";
          break;
        case 409:
          normalized.message = error.response.data?.message || "Conflict";
          break;
        case 422:
          normalized.message =
            error.response.data?.message || "Validation failed";
          normalized.details = error.response.data?.errors;
          break;
        case 429:
          normalized.message = "Too many requests. Please wait.";
          break;
        case 500:
          normalized.message = "Server error. Please try again later.";
          break;
      }
    } else if (error.request) {
      normalized.message = "Network error. Check your connection.";
    }

    return Promise.reject(normalized);
  },
);

export { cxaxios };
```

## Mutation Error Handling

```js
// features/Campaigns/api/mutations/useCampaignMutations.js
import { createMutation } from "~/lib/createMutation";
import { campaignKeys } from "../keys";
import { cxaxios } from "~/utils/cxaxios";

export const useCreateCampaignMutation = (options = {}) =>
  createMutation({
    mutationKey: ["createCampaign"],
    mutationFn: (data) => cxaxios.post("/v2/campaigns", data),
    invalidates: [campaignKeys.lists()],
    onError: (error) => {
      // Toast notifications handled by mutation factory
      // Log to Sentry automatically
      console.error("Create campaign failed:", error);
    },
  })(options);
```

### Mutation Factory with Error Handling

```js
// src/lib/createMutation.js
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { useToast } from "~/components/ui/use-toast";

export function createMutation({ mutationFn, invalidates, ...config }) {
  return (options) => {
    const queryClient = useQueryClient();
    const { toast } = useToast();

    return useMutation({
      mutationFn,
      ...config,
      onSuccess: (...args) => {
        if (invalidates) {
          queryClient.invalidateQueries({ queryKey: invalidates });
        }
        config.onSuccess?.(...args);
      },
      onError: (error, variables, context) => {
        toast({
          variant: "destructive",
          title: "Error",
          description: error.message || "Something went wrong",
        });
        config.onError?.(error, variables, context);
      },
    });
  };
}
```

## Query Error Handling

```js
import { useQuery } from "@tanstack/react-query";
import { campaignKeys } from "../keys";
import { cxaxios } from "~/utils/cxaxios";

export function useCampaignQuery(id) {
  return useQuery({
    queryKey: campaignKeys.detail(id),
    queryFn: ({ signal }) =>
      cxaxios.get(`/v2/campaigns/${id}`, { signal }).then((r) => r.data),
    staleTime: 1000 * 60 * 5,
    enabled: !!id,
  });
}
```

```jsx
// In component
function CampaignDetail({ id }) {
  const { data, isLoading, error, refetch } = useCampaignQuery(id);

  if (isLoading) return <Skeleton />;
  if (error) {
    return (
      <ErrorState
        message={error.message}
        onRetry={refetch}
      />
    );
  }

  return <div>{/* render data */}</div>;
}
```

## Error Recovery Patterns

### Retry with Backoff

```js
// Global QueryClient config
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: 3,
      retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
      staleTime: 1000 * 60 * 5,
    },
  },
});
```

### Optimistic Updates with Rollback

```js
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { todoKeys } from "../keys";

export function useToggleTodoMutation() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, completed }) =>
      cxaxios.patch(`/v2/todos/${id}`, { completed }),

    onMutate: async ({ id, completed }) => {
      await queryClient.cancelQueries({ queryKey: todoKeys.all });
      const previous = queryClient.getQueriesData({ queryKey: todoKeys.all });

      queryClient.setQueriesData({ queryKey: todoKeys.all }, (old) =>
        old?.map((todo) =>
          todo.id === id ? { ...todo, completed } : todo,
        ),
      );

      return { previous };
    },

    onError: (_err, _vars, context) => {
      // Rollback on failure
      context.previous.forEach(([key, data]) =>
        queryClient.setQueryData(key, data),
      );
    },

    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: todoKeys.all });
    },
  });
}
```

## Sentry Integration

```js
// src/lib/sentry.js
import * as Sentry from "@sentry/react";
import { createRoutesFromChildren, matchRoutes, useLocation, useNavigationType } from "react-router-dom";

Sentry.init({
  dsn: import.meta.env.VITE_SENTRY_DSN,
  environment: import.meta.env.MODE,
  integrations: [
    Sentry.reactRouterV7BrowserTracingIntegration({
      useEffect, useLocation, useNavigationType,
      createRoutesFromChildren, matchRoutes,
    }),
    Sentry.replayIntegration(),
  ],
  tracesSampleRate: 0.1,
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
});

// Capture exceptions manually
try {
  await riskyOperation();
} catch (error) {
  Sentry.captureException(error);
}
```

## Patterns to Avoid

```js
// ❌ Swallowing errors silently
try {
  await doSomething();
} catch (e) {
  // empty
}

// ❌ Turning failures into fake successes
catch (error) {
  return error.response.data;  // callers can't distinguish success from error
}

// ❌ Console-only logging without user feedback
catch (error) {
  console.error(error);
}

// ✅ Throw so callers can handle
catch (error) {
  console.error("API Error:", error);
  throw error;
}
```

## Checklist

- [ ] Error boundary wrapping each route or feature
- [ ] Axios interceptor normalizing API errors
- [ ] Mutation `onError` shows toast notification
- [ ] Query `error` displayed with retry option
- [ ] Sentry initialized with tracing and replay
- [ ] No silent catches or swallowed errors
- [ ] Optimistic updates roll back on failure
- [ ] Network errors distinguished from server errors
