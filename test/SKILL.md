---
name: test
description: Write Vitest + React Testing Library tests for React components, custom hooks, Zustand stores, and TanStack Query integrations. Use when adding tests, debugging a recurring bug, or ensuring regression coverage.
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# Testing

Vitest + React Testing Library test patterns for React 19 + Vite projects. Tests are added sparingly — only when a bug ships twice or the logic is too complex to verify manually.

## When to write tests

- A bug shipped to production twice
- Complex business logic (date calculations, status transitions, sorting)
- Utility functions with many edge cases
- Custom hooks with non-trivial state logic
- Critical user flows (auth, checkout, data export)

Do **not** write tests for:
- Presentational components (they change too often)
- TanStack Query hooks (test the logic, not the framework)
- Simple Zustand stores

## Stack

- **Runner:** Vitest (inline with Vite config)
- **DOM:** jsdom (via `@testing-library/jest-dom`)
- **Components:** `@testing-library/react` + `@testing-library/user-event`
- **Hooks:** `@testing-library/react-hooks`
- **API mocking:** `msw` (Mock Service Worker)
- **Queries:** Custom `render` wrapper with providers (Router, QueryClient)

## Setup

### Vite config

```js
// vite.config.js
/// <reference types="vitest" />
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: "jsdom",
    setupFiles: "./src/test/setup.js",
    css: true,
  },
});
```

### Test setup

```js
// src/test/setup.js
import "@testing-library/jest-dom";
```

### Custom render with providers

```js
// src/test/test-utils.jsx
import { render as rtlRender } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { MemoryRouter } from "react-router-dom";

const createTestQueryClient = () =>
  new QueryClient({
    defaultOptions: {
      queries: { retry: false, gcTime: 0 },
      mutations: { retry: false },
    },
  });

export function render(ui, { route = "/", ...options } = {}) {
  const queryClient = createTestQueryClient();

  function Wrapper({ children }) {
    return (
      <QueryClientProvider client={queryClient}>
        <MemoryRouter initialEntries={[route]}>
          {children}
        </MemoryRouter>
      </QueryClientProvider>
    );
  }

  return { ...rtlRender(ui, { wrapper: Wrapper, ...options }), queryClient };
}
```

## Component Tests

### Rendering and interactions

```jsx
import { render, screen } from "src/test/test-utils";
import userEvent from "@testing-library/user-event";
import { describe, it, expect, vi } from "vitest";
import { CampaignCard } from "./CampaignCard";

describe("CampaignCard", () => {
  it("renders campaign name", () => {
    render(<CampaignCard name="Summer Sale" status="active" />);
    expect(screen.getByText("Summer Sale")).toBeInTheDocument();
  });

  it("calls onEdit when edit button clicked", async () => {
    const onEdit = vi.fn();
    render(<CampaignCard name="Test" onEdit={onEdit} />);
    await userEvent.click(screen.getByRole("button", { name: /edit/i }));
    expect(onEdit).toHaveBeenCalledWith("campaign-1");
  });
});
```

### Testing list states

```jsx
import { render, screen } from "src/test/test-utils";
import { describe, it, expect } from "vitest";
import { CampaignList } from "./CampaignList";

describe("CampaignList", () => {
  it("shows empty state when no campaigns", () => {
    render(<CampaignList campaigns={[]} />);
    expect(screen.getByText(/no campaigns/i)).toBeInTheDocument();
  });

  it("shows loading skeleton", () => {
    render(<CampaignList isLoading />);
    expect(screen.getByTestId("skeleton")).toBeInTheDocument();
  });

  it("shows error state", () => {
    render(<CampaignList error={new Error("Failed")} />);
    expect(screen.getByText(/failed/i)).toBeInTheDocument();
  });
});
```

## Hook Tests

```jsx
import { renderHook, act, waitFor } from "@testing-library/react";
import { describe, it, expect, vi } from "vitest";
import { useCampaignFilters } from "./useCampaignFilters";

describe("useCampaignFilters", () => {
  it("initializes with defaults", () => {
    const { result } = renderHook(() => useCampaignFilters());
    expect(result.current.status).toBe("all");
    expect(result.current.page).toBe(1);
  });

  it("updates status filter", () => {
    const { result } = renderHook(() => useCampaignFilters());
    act(() => result.current.setStatus("active"));
    expect(result.current.status).toBe("active");
  });

  it("resets page when filter changes", () => {
    const { result } = renderHook(() => useCampaignFilters());
    act(() => result.current.setPage(3));
    act(() => result.current.setStatus("active"));
    expect(result.current.page).toBe(1);
  });
});
```

## Store Tests

```jsx
import { describe, it, expect, beforeEach } from "vitest";
import { useCampaignsStore } from "./useCampaignsStore";

describe("useCampaignsStore", () => {
  beforeEach(() => {
    useCampaignsStore.setState(useCampaignsStore.getInitialState());
  });

  it("opens delete dialog", () => {
    const campaign = { id: "1", name: "Test" };
    useCampaignsStore.getState().openDelete(campaign);
    const state = useCampaignsStore.getState();
    expect(state.deleteDialog.open).toBe(true);
    expect(state.deleteDialog.campaign).toEqual(campaign);
  });

  it("closes delete dialog", () => {
    useCampaignsStore.getState().closeDelete();
    const state = useCampaignsStore.getState();
    expect(state.deleteDialog.open).toBe(false);
    expect(state.deleteDialog.campaign).toBeNull();
  });
});
```

## MSW API Mocking

```js
// src/test/mocks/handlers.js
import { http, HttpResponse } from "msw";

export const handlers = [
  http.get("/v2/campaigns", ({ request }) => {
    const url = new URL(request.url);
    const page = Number(url.searchParams.get("page")) || 1;
    return HttpResponse.json({
      data: [{ id: "1", name: "Test Campaign", status: "active" }],
      meta: { page, total: 1 },
    });
  }),

  http.post("/v2/campaigns", async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({ id: "new-1", ...body }, { status: 201 });
  }),
];
```

```js
// src/test/mocks/server.js
import { setupServer } from "msw/node";
import { handlers } from "./handlers";
export const server = setupServer(...handlers);
```

```js
// src/test/setup.js (updated)
import "@testing-library/jest-dom";
import { server } from "./mocks/server";

beforeAll(() => server.listen({ onUnhandledRequest: "warn" }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

## Running Tests

```bash
pnpm vitest                  # watch mode
pnpm vitest run              # single run
pnpm vitest run --coverage   # with coverage
pnpm vitest run src/features/Campaigns --reporter=verbose  # single feature
```

## Verification

```bash
pnpm vitest run
pnpm build    # ensure build still passes
pnpm lint     # ensure lint is clean
```
