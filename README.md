# skills

Portable AI-agent skills for **React 19 + Vite** frontend work. Install them into Claude Code, Codex, opencode, Cursor, and other agents via the open [`skills`](https://github.com/vercel-labs/skills) ecosystem — one source, every agent.

## Install

```bash
# everything, into every detected agent (Claude Code, Codex, opencode, Cursor, …)
npx skills add ashishxcode/skills --all

# or pick skills / agents
npx skills add ashishxcode/skills -s api-migration,review
npx skills add ashishxcode/skills -a claude-code        # single agent
```

Each `<name>/SKILL.md` is the universal source; the CLI translates it into each agent's native format on install (`.claude/skills/`, `.cursor/rules/`, Codex/opencode dirs, …).

## Skills

| Skill | Purpose | Trigger |
| --- | --- | --- |
| `react-perf` | 37 Vercel performance rules (parallel async, bundle size, re-render, …) | auto-applied |
| `review` | Code review — quality, perf, security, a11y | "review this PR" |
| `refactor` | Identify code smells and refactor React code | "refactor this file" |
| `perf` | Bundle / render / Core Web Vitals analysis | "audit performance" |
| `generate` | Generate React components to project conventions | "generate a UserCard" |
| `api-migration` | Migrate legacy imperative API hooks → TanStack Query | "migrate useXxxAPI" |
| `commit` | Organize changes into conventional commits | "organize my commits" |
| `code-check` | Pre-commit gate — lint, format, a11y, security, anti-patterns | "check my code" |
| `test` | Vitest + Testing Library for components, hooks, stores | "write tests for UserCard" |
| `form` | react-hook-form + zod validation patterns | "generate a create form" |
| `errors` | Error boundaries, API error normalization, Sentry | "add error handling" |
| `router` | React Router v7, lazy routes, layout patterns | "add a detail route" |
| `ui` | Shadcn/ui component composition and customization | "create a new Select variant" |
| `build` | Vite, Turborepo, pnpm workspace configuration | "configure Vite for this app" |

## Stack assumptions

React 19 · Vite · TanStack Query v5 · Zustand · Tailwind + Shadcn/ui · react-hook-form + zod · React Router v7 · pnpm · Turborepo (`.jsx`/`.js`, no TypeScript).
The performance rules come from [Vercel's React Best Practices](https://vercel.com/blog/introducing-react-best-practices) — see [`REFERENCE.md`](./REFERENCE.md) for the full guide.

## Authoring

```bash
npx skills init <name>     # scaffold <name>/SKILL.md, then edit it
```
Keep skills focused and example-driven (Bad/Good blocks). Frontmatter: `name`, `description`, optional `allowed-tools`.
