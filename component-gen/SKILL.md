---
name: component-gen
description: Generate new React components following project patterns and conventions. Use when creating components or features.
allowed-tools: Write, Read, Glob, Grep
---

# Component Generator

Creates React components following project architecture and conventions.

## Project Patterns

### File Structure

**Shared UI Components** (`~/components/`)
```
src/components/
├── ui/                    # Shadcn primitives
│   ├── button.jsx
│   └── dialog.jsx
├── ComponentName/
│   ├── ComponentName.jsx  # Main component
│   └── index.js           # Barrel export
```

**Feature Components** (`~/features/{feature}/`)
```
src/features/FeatureName/
├── api/                   # React Query hooks
│   └── useFeatureData.js
├── components/
│   ├── ComponentName/
│   │   ├── ComponentName.jsx
│   │   └── index.js
│   └── index.js           # Barrel export
├── hooks/                 # Feature-specific hooks
├── store/                 # Zustand stores
│   └── useFeatureStore.js
├── constants/
│   └── index.js
└── utils/
    └── index.js
```

## Component Templates

### Basic Component
```jsx
import { cn } from '~/utils/cn';

export function ComponentName({ className, children, ...props }) {
  return (
    <div className={cn('base-styles', className)} {...props}>
      {children}
    </div>
  );
}
```

### Component with State
```jsx
import { useState, useCallback } from 'react';
import { cn } from '~/utils/cn';

export function ComponentName({ initialValue, onChange, className }) {
  const [value, setValue] = useState(initialValue);

  const handleChange = useCallback((newValue) => {
    setValue(newValue);
    onChange?.(newValue);
  }, [onChange]);

  return (
    <div className={cn('base-styles', className)}>
      {/* Component content */}
    </div>
  );
}
```

### Component with Zustand Store
```jsx
import { useFeatureStore } from '../store';

export function ComponentName() {
  const { items, addItem } = useFeatureStore(state => ({
    items: state.items,
    addItem: state.addItem,
  }));

  return (
    <div>
      {items.map(item => (
        <div key={item.id}>{item.name}</div>
      ))}
    </div>
  );
}
```

### Component with React Query
```jsx
import { useFeatureData } from '../api/useFeatureData';
import { Spinner } from '~/components/ui/spinner';

export function ComponentName({ id }) {
  const { data, isLoading, error } = useFeatureData(id);

  if (isLoading) return <Spinner />;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <div>
      {/* Render data */}
    </div>
  );
}
```

## Zustand Store Template
```jsx
import { create } from 'zustand';
import { immer } from 'zustand/middleware/immer';

export const useFeatureStore = create(
  immer((set, get) => ({
    // State
    items: [],
    isLoading: false,

    // Actions
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
  }))
);
```

## React Query Hook Template
```jsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import axios from 'axios';

const QUERY_KEY = ['feature', 'items'];

export function useFeatureData(id) {
  return useQuery({
    queryKey: [...QUERY_KEY, id],
    queryFn: async () => {
      const { data } = await axios.get(`/api/feature/${id}`);
      return data;
    },
    staleTime: 5 * 60 * 1000,
  });
}

export function useCreateFeature() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (payload) => {
      const { data } = await axios.post('/api/feature', payload);
      return data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: QUERY_KEY });
    },
  });
}
```

## Barrel Export Template
```jsx
// index.js
export { ComponentName } from './ComponentName';
export { AnotherComponent } from './AnotherComponent';
```

## Naming Conventions

- **Components**: PascalCase (`UserProfile.jsx`)
- **Hooks**: camelCase with `use` prefix (`useUserData.js`)
- **Stores**: camelCase with `use` prefix (`useUserStore.js`)
- **Utils**: camelCase (`formatDate.js`)
- **Constants**: SCREAMING_SNAKE_CASE for values

## Usage

When asked to create a component:
1. Determine if it's a shared component or feature-specific
2. Check existing patterns in the codebase
3. Create files following the templates above
4. Add barrel exports
5. Use Tailwind + cn() for styling
