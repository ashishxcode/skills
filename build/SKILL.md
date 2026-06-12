---
name: build
description: Vite, Turborepo, pnpm workspace configuration — build pipelines, ESLint, Prettier, project scaffolding. Use when configuring build tooling, debugging build failures, or setting up a new app/package.
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# Build & CI

Build configuration patterns for Turborepo monorepos with pnpm workspaces, Vite, and React 19.

## Monorepo Structure

```
├── turbo.json              # Turborepo pipeline config
├── pnpm-workspace.yaml     # Workspace definition
├── package.json            # Root scripts and devDependencies
├── apps/
│   └── dashboard/
│       ├── vite.config.js
│       ├── package.json
│       └── src/
└── packages/
    └── ui/                 # Shared UI component library
        ├── package.json
        └── src/
```

## pnpm Workspace

```yaml
# pnpm-workspace.yaml
packages:
  - "apps/*"
  - "packages/*"
```

## Turborepo Pipeline

```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "typecheck": {
      "dependsOn": ["^build"]
    },
    "lint": {
      "dependsOn": ["^build"]
    },
    "test": {
      "dependsOn": ["^build"]
    },
    "format": {
      "cache": false
    },
    "//#format": {
      "cache": false
    }
  }
}
```

## Vite Configuration

```js
// apps/dashboard/vite.config.js
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import path from "path";

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      "~": path.resolve(__dirname, "./src"),
    },
  },
  server: {
    port: 3002,
    proxy: {
      "/api": {
        target: "http://localhost:4000",
        changeOrigin: true,
      },
    },
  },
  build: {
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ["react", "react-dom", "react-router-dom"],
          ui: [
            "@radix-ui/react-dialog",
            "@radix-ui/react-dropdown-menu",
            "@radix-ui/react-select",
          ],
        },
      },
    },
  },
});
```

## Package Configuration

### App Package

```json
{
  "name": "@workspace/dashboard",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "lint": "eslint src/ --max-warnings 0",
    "typecheck": "vite build --noEmit || echo 'No TS in this project'",
    "format": "prettier --write \"src/**/*.{js,jsx,css,json}\"",
    "format:check": "prettier --check \"src/**/*.{js,jsx,css,json}\""
  },
  "dependencies": {
    "@tanstack/react-query": "^5",
    "@workspace/ui": "workspace:*",
    "react": "^19",
    "react-dom": "^19",
    "react-hook-form": "^7",
    "react-router-dom": "^7",
    "zustand": "^5",
    "zod": "^3",
    "@hookform/resolvers": "^4",
    "axios": "^1",
    "lucide-react": "^0.400",
    "clsx": "^2",
    "tailwind-merge": "^3"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4",
    "vite": "^7",
    "eslint": "^9",
    "prettier": "^3",
    "tailwindcss": "^4",
    "@tailwindcss/vite": "^4",
    "autoprefixer": "^10",
    "postcss": "^8"
  }
}
```

### Shared UI Package

```json
{
  "name": "@workspace/ui",
  "private": true,
  "type": "module",
  "exports": {
    "./components/*": "./src/components/*",
    "./lib/utils": "./src/lib/utils.js"
  },
  "scripts": {
    "lint": "eslint src/",
    "format": "prettier --write \"src/**/*.{js,jsx,css}\""
  },
  "dependencies": {
    "clsx": "^2",
    "tailwind-merge": "^3",
    "lucide-react": "^0.400"
  },
  "peerDependencies": {
    "react": "^19",
    "react-dom": "^19"
  }
}
```

## ESLint

```js
// eslint.config.js
import js from "@eslint/js";
import reactHooks from "eslint-plugin-react-hooks";
import reactRefresh from "eslint-plugin-react-refresh";

export default [
  js.configs.recommended,
  {
    plugins: {
      "react-hooks": reactHooks,
      "react-refresh": reactRefresh,
    },
    rules: {
      ...reactHooks.configs.recommended.rules,
      "react-refresh/only-export-components": ["warn", { allowConstantExport: true }],
      "no-unused-vars": ["warn", { argsIgnorePattern: "^_" }],
    },
  },
];
```

## Prettier

```json
{
  "semi": true,
  "singleQuote": false,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2
}
```

## Common Tasks

### Add a new app

```bash
mkdir -p apps/new-app/src
cd apps/new-app
pnpm init
# Add package.json, vite.config.js, etc.
pnpm install
```

### Add a shared package dependency

```bash
pnpm --filter @workspace/dashboard add @workspace/ui
```

### Run commands across packages

```bash
pnpm --filter @workspace/dashboard dev
pnpm --filter @workspace/dashboard build
pnpm --filter @workspace/dashboard lint

# Across all
pnpm build         # turbo build
pnpm lint          # turbo lint
pnpm format        # prettier --write
```

### Analyze bundle

```bash
pnpm --filter @workspace/dashboard build
# Install visualizer
pnpm --filter @workspace/dashboard add -D rollup-plugin-visualizer
```

```js
// vite.config.js — add visualizer conditionally
import { visualizer } from "rollup-plugin-visualizer";

export default defineConfig({
  plugins: [
    react(),
    process.env.ANALYZE && visualizer({ open: true }),
  ],
});
```

## Troubleshooting

### ESLint resolution issues in monorepo

```bash
# Ensure eslint is installed at root
pnpm add -w -D eslint
# Run lint from root
pnpm lint
```

### Build cache invalidation

```bash
# Clear turbo cache for specific package
rm -rf apps/dashboard/.turbo
# Or full clean
pnpm turbo clean
```

### Port conflicts

```json
// Change dev port in vite.config.js
server: { port: 3003 }
```

## Verification

```bash
pnpm format:check
pnpm lint
pnpm build
```
