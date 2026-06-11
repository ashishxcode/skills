# skills

Portable AI-agent skills for **React 19 + Vite** frontend work. Install them into Claude Code, Codex, opencode, Cursor, and other agents via the open [`skills`](https://github.com/vercel-labs/skills) ecosystem — one source, every agent.

## Install

```bash
# everything, into every detected agent (Claude Code, Codex, opencode, Cursor, …)
npx skills add ashishxcode/skills --all

# or pick skills / agents
npx skills add ashishxcode/skills -s migrate-api,review
npx skills add ashishxcode/skills -a claude-code        # single agent
```

Each `<name>/SKILL.md` is the universal source; the CLI translates it into each agent's native format on install (`.claude/skills/`, `.cursor/rules/`, Codex/opencode dirs, …).

## Skills

| Skill | Purpose | Trigger |
| --- | --- | --- |
| `react-best-practices` | 37 Vercel performance rules (parallel async, bundle size, re-render, …) | auto-applied |
| `review` | Frontend code review — quality, perf, security, a11y | "review this PR" |
| `refactor` | Identify code smells and refactor React code | "refactor this file" |
| `perf-audit` | Bundle / render / Core Web Vitals analysis | "audit performance" |
| `component-gen` | Generate React components to project conventions | "generate a UserCard" |
| `migrate-api` | Migrate legacy imperative API hooks → feature `api/` (TanStack Query) | "migrate useXxxAPI" |
| `commit` | Organize changes into conventional commits | "organize my commits" |

## Stack assumptions

React 19 · Vite · TanStack Query v5 · Zustand · Tailwind + Shadcn/ui · pnpm.
The performance rules come from [Vercel's React Best Practices](https://vercel.com/blog/introducing-react-best-practices) — see [`REFERENCE.md`](./REFERENCE.md) for the full guide.

## Authoring

```bash
npx skills init <name>     # scaffold <name>/SKILL.md, then edit it
```
Keep skills focused and example-driven (Bad/Good blocks). Frontmatter: `name`, `description`, optional `allowed-tools`.
