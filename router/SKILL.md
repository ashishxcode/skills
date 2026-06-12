---
name: router
description: React Router v7 patterns — lazy loading, layout routes, page structure, navigation guards, breadcrumbs, URL state. Use when creating or modifying routes.
allowed-tools: Read, Write, Edit, Glob, Grep
---

# Routing

React Router v7 patterns for React 19 + Vite projects with feature-module architecture.

## Route Structure

Routes wire to pages by **direct lazy import of the page file**, not a barrel.

```js
// src/router.jsx
import { createBrowserRouter, Navigate } from "react-router-dom";
import { lazyLoad } from "~/lib/lazyLoad";
import { AppLayout } from "~/components/AppLayout";
import { AuthLayout } from "~/components/AuthLayout";
import { ErrorBoundary } from "~/components/ErrorBoundary";

export const router = createBrowserRouter([
  {
    path: "/",
    element: <Navigate to="/campaigns" replace />,
  },
  {
    element: <AuthLayout />,
    children: [
      {
        path: "/login",
        element: lazyLoad(() => import("~/features/Auth/pages/LoginPage")),
      },
    ],
  },
  {
    element: <AppLayout />,
    errorElement: <ErrorBoundary />,
    children: [
      {
        path: "campaigns",
        children: [
          {
            index: true,
            element: lazyLoad(
              () => import("~/features/Campaigns/pages/CampaignListPage"),
            ),
          },
          {
            path: "create",
            element: lazyLoad(
              () => import("~/features/Campaigns/pages/CampaignCreatePage"),
            ),
          },
          {
            path: ":uuid",
            element: lazyLoad(
              () => import("~/features/Campaigns/pages/CampaignDetailPage"),
            ),
          },
          {
            path: ":uuid/edit",
            element: lazyLoad(
              () => import("~/features/Campaigns/pages/CampaignEditPage"),
            ),
          },
        ],
      },
    ],
  },
]);
```

## Lazy Load Utility

```js
// src/lib/lazyLoad.jsx
import { lazy, Suspense } from "react";
import { Skeleton } from "~/components/ui/skeleton";

export function lazyLoad(importFn) {
  const Component = lazy(importFn);

  return (
    <Suspense fallback={<Skeleton className="h-96 w-full" />}>
      <Component />
    </Suspense>
  );
}
```

## Layout Routes

### App Layout

```jsx
// src/components/AppLayout.jsx
import { Outlet } from "react-router-dom";
import { Sidebar } from "~/components/Sidebar";
import { Header } from "~/components/Header";

export function AppLayout() {
  return (
    <div className="flex h-screen">
      <Sidebar />
      <div className="flex-1 flex flex-col">
        <Header />
        <main className="flex-1 overflow-auto p-6">
          <Outlet />
        </main>
      </div>
    </div>
  );
}
```

### Auth Layout

```jsx
export function AuthLayout() {
  return (
    <div className="min-h-screen flex items-center justify-center">
      <Outlet />
    </div>
  );
}
```

## Navigation

```jsx
import { useNavigate } from "react-router-dom";

function CampaignListPage() {
  const navigate = useNavigate();

  return (
    <Button onClick={() => navigate("/campaigns/create")}>
      New Campaign
    </Button>
  );
}
```

### useNavigate vs Link

```jsx
// ✅ For navigation triggered by user click
<Link to={`/campaigns/${uuid}`}>{name}</Link>

// ✅ For navigation triggered by side effect
const navigate = useNavigate();
const handleSubmit = async () => {
  await mutation.mutateAsync(data);
  navigate("/campaigns");
};
```

## URL State

Keep list filters, pagination, and active tabs in URL search params — not Zustand.

```jsx
import { useSearchParams } from "react-router-dom";

function CampaignListPage() {
  const [searchParams, setSearchParams] = useSearchParams();
  const status = searchParams.get("status") ?? "all";
  const page = Number(searchParams.get("page")) ?? 1;

  const setStatus = (newStatus) => {
    setSearchParams((prev) => {
      prev.set("status", newStatus);
      prev.set("page", "1"); // reset page on filter change
      return prev;
    });
  };

  const setPage = (newPage) => {
    setSearchParams((prev) => {
      prev.set("page", String(newPage));
      return prev;
    });
  };

  return (
    <CampaignList
      status={status}
      page={page}
      onStatusChange={setStatus}
      onPageChange={setPage}
    />
  );
}
```

## Navigation Guards

```jsx
// src/components/AuthGuard.jsx
import { Navigate, Outlet, useLocation } from "react-router-dom";
import { useAuth } from "~/hooks/useAuth";

export function AuthGuard() {
  const { user, isLoading } = useAuth();
  const location = useLocation();

  if (isLoading) return <FullPageSpinner />;
  if (!user) {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  return <Outlet />;
}
```

```jsx
// Route with guard
{
  element: <AuthGuard />,
  children: [
    { path: "campaigns", element: <CampaignListPage /> },
  ],
}
```

## Breadcrumbs

```jsx
import { Link, useMatches } from "react-router-dom";

export function Breadcrumbs() {
  const matches = useMatches();

  const crumbs = matches
    .filter((match) => match.handle?.crumb)
    .map((match) => ({
      label: match.handle.crumb(match.data),
      path: match.pathname,
    }));

  return (
    <nav aria-label="Breadcrumb">
      <ol className="flex items-center gap-2">
        {crumbs.map((crumb, i) => (
          <li key={crumb.path}>
            {i < crumbs.length - 1 ? (
              <Link to={crumb.path}>{crumb.label}</Link>
            ) : (
              <span aria-current="page">{crumb.label}</span>
            )}
          </li>
        ))}
      </ol>
    </nav>
  );
}
```

```js
// Route definition with breadcrumb handle
{
  path: ":uuid",
  element: lazyLoad(() => import("./pages/CampaignDetailPage")),
  handle: {
    crumb: (data) => data?.name ?? "Campaign",
  },
}
```

## Query Param Hooks

```js
// src/hooks/useListFilters.js
import { useSearchParams } from "react-router-dom";
import { useCallback, useMemo } from "react";

export function useListFilters(defaults = {}) {
  const [searchParams, setSearchParams] = useSearchParams();

  const filters = useMemo(
    () => ({
      page: Number(searchParams.get("page")) || defaults.page || 1,
      limit: Number(searchParams.get("limit")) || defaults.limit || 20,
      search: searchParams.get("search") || defaults.search || "",
      sort: searchParams.get("sort") || defaults.sort || "-created_at",
    }),
    [searchParams],
  );

  const setFilters = useCallback(
    (updates) => {
      setSearchParams((prev) => {
        Object.entries(updates).forEach(([key, value]) => {
          if (value === undefined || value === null || value === "") {
            prev.delete(key);
          } else {
            prev.set(key, String(value));
          }
        });
        return prev;
      }, { replace: true });
    },
    [setSearchParams],
  );

  return { filters, setFilters };
}
```

## Best Practices

- One page file per route entry — never reuse page files across routes
- Feature pages live in `features/<Feature>/pages/`
- Layouts live in `src/components/` (shared layout) or `features/<Feature>/components/` (feature-specific)
- Use `searchParams` for filter/pagination state — it survives refresh and enables shareable URLs
- Use `state` for ephemeral navigation data (e.g., "show toast after redirect")
- Wrap each route group in its own `ErrorBoundary` to contain failures
