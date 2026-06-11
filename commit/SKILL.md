---
name: commit
description: Organize staged git changes into logical commits with conventional commit messages. Use when committing code or organizing changes.
allowed-tools: Bash(git:*), Read, Glob, Grep
---

# Commit Agent

Organizes staged git changes into logical, well-structured commits following Conventional Commits format.

## Workflow

1. **Analyze staged changes**
   ```bash
   git diff --cached --name-only
   git diff --cached --stat
   ```

2. **Group changes logically** by:
   - Feature area (e.g., components, utils, hooks, store)
   - Layer (primitives → composites → consumers)
   - Dependency order (base files before dependent files)

3. **Unstage all files** to reorganize:
   ```bash
   git reset HEAD
   ```

4. **Commit in logical order** - stage and commit each group:
   ```bash
   git add <files> && git commit -m "<type>: <description>"
   ```

5. **Verify** with `git log --oneline` and `git status`

## Commit Message Format

```
<type>: <description>
```

### Types
- `feat`: New features or functionality
- `fix`: Bug fixes
- `refactor`: Code restructuring without behavior change
- `style`: Formatting, whitespace (no logic changes)
- `docs`: Documentation only
- `test`: Adding or updating tests
- `perf`: Performance improvements
- `chore`: Maintenance, dependencies
- `ci`: CI/CD configuration

### Rules
- Use imperative mood: "add" not "added" or "adds"
- Keep under 50 characters
- No period at end
- Lowercase after colon
- Be specific about what changed
- NEVER include Co-Authored-By or Claude attribution

## Grouping Strategy

### For New Features
1. Constants and utilities first
2. Primitive/base components
3. Composite components (by category)
4. Higher-level components
5. Store/state management
6. Main feature component
7. Integration/consumer changes (refactor prefix)

### For Bug Fixes
- Single commit per logical fix
- Include related test updates in same commit

### For Refactors
- Group by affected module/area
- Keep related changes together

## Example: Component Library Addition

```
feat: add Reports constants and utility functions
feat: add Reports table primitive cell components
feat: add Reports table metrics cell components
feat: add Reports table name cell components
feat: add Reports table selection cell components
feat: add Reports table workflow cell components
feat: add Reports table cells barrel export
feat: add Reports table column definitions
feat: add Reports table Zustand store
feat: add ReportsList table component
refactor: migrate legacy report list to use new Reports table components
```

## Execution Pattern

For each commit group:
```bash
git add <file1> <file2> ... && git commit -m "<type>: <description>"
```

Always verify final state:
```bash
git log --oneline -<n>  # Show n recent commits
git status              # Confirm clean working tree
```
