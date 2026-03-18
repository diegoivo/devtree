---
name: finish
description: Run quality gates, merge feature branch to main, and clean up all worktree resources
---

# /devtree finish

Finalizes the current workstream: runs tests, coverage delta check, SonarQube gate, merges to base branch, and cleans up all resources.

**IMPORTANT:** This skill STOPS immediately if any verification fails. It never auto-resolves conflicts or bypasses quality gates.

## Step 0: Load project config

Read `.claude/worktree-config.md` from the project root.

If the file does not exist, run the **interactive setup** before proceeding:

1. Auto-detect defaults:
   - `base_branch`: `git symbolic-ref refs/remotes/origin/HEAD | sed 's|refs/remotes/origin/||'`
   - `app_paths`: glob `apps/*/package.json` → extract parent dirs. If none, use `.` (root)
   - `database`: grep for `DATABASE_URL.*neon\.tech` in `.env*` files → `neon`. If not found → `none`
   - `prisma`: check if `prisma/schema.prisma` exists in any app_path → `true`/`false`
   - `redis`: grep for `upstash` in `.env*` → `upstash`. If not found → `none`
   - `test_command`: read root `package.json` → `scripts.test`. Default: `npm test`
   - `dev_command`: read first app_path `package.json` → `scripts.dev`. Default: `npm run dev`
   - `setup_command`: if prisma=true → `npm install && cd {first_app_path} && npx prisma generate`. Else → `npm install`
   - `port_range_start`: 9100, `port_range_end`: 9900, `port_step`: 100
2. Present detected values to user, ask to confirm or adjust
3. Ask for `redis_prefix_template` (default: `"wt:{name}:"`) and `env_overrides`
4. Write `.claude/worktree-config.md`
5. Verify `.gitignore` contains `.claude/worktree-config.md`, add if not
6. Continue with finish

Parse the config into variables: `{base_branch}`, `{app_paths}`, `{database}`, `{prisma}`, `{test_command}`.

## Execution Steps

### 1. Identify current workstream

```bash
git branch --show-current
```

Match the current branch against rows in `WORKSTREAMS.md`. If not in a worktree branch, **STOP** with error.

Extract: `<name>`, `<type>/<branch>`, `<port>`, `<db_branch>`, `<redis_prefix>`.

### 2. Run tests

```bash
{test_command}
```

If tests fail → **STOP**. Print failures. Do NOT proceed.

### 3. Coverage delta check

Verify that new/modified source files have corresponding test files.

```bash
git diff {base_branch}...HEAD --name-only --diff-filter=AM -- '*.ts' '*.tsx'
```

For each file in the output:
1. **Skip if NOT in source directories**: only check files within `{app_path}/src/` for each path in `{app_paths}`
2. **Skip if already a test file**: matches `*.test.ts`, `*.test.tsx`, `*.spec.ts`, `*.spec.tsx`, or is inside `__tests__/`
3. **Skip if excluded pattern**: `*.d.ts`, `types.ts`, `types/*.ts`, `constants.ts`, `index.ts`, `*.config.ts`, `use-*.ts`, `use-*.tsx`, `*-provider.tsx`, `*-context.tsx`
4. **Skip if Next.js route file**: `page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx`, `not-found.tsx`, `route.ts`, `template.tsx`, `global-error.tsx`, `default.tsx`, `opengraph-image.tsx`, `middleware.ts`
5. **Skip if UI component library**: path contains `components/ui/`
6. For remaining files: check if corresponding `__tests__/<name>.test.<ext>` exists

If ANY file has no test → **STOP** with:
```
Coverage gate failed. New/modified files without tests:

  <app_path>/src/lib/new-feature.ts
    Expected: <app_path>/src/lib/__tests__/new-feature.test.ts

Write tests for these files before finishing the workstream.
```

### 4. Run SonarQube quality gate

Execute `/devtree scan --full`.

If quality gate fails → **STOP**. Print failing conditions. Do NOT proceed.

### 5. Merge to base branch

```bash
git checkout {base_branch}
git pull origin {base_branch}
git merge <type>/<name>
```

If merge conflict → **STOP**. Print conflicting files. Tell user to resolve manually, then re-run `/devtree finish`.

### 6. Apply migrations (conditional)

**Only if `{prisma}` = `true` AND `{database}` = `neon`:**

Read the MAIN project's `.env.local` (at the project root, NOT the worktree) to get `DATABASE_URL` and `DIRECT_DATABASE_URL` for the main Neon branch.

```bash
cd {first_app_path} && DATABASE_URL="<main_pooled_url>" DIRECT_DATABASE_URL="<main_direct_url>" npx prisma migrate deploy
```

If migration fails → **STOP**. Print error. The merge is done — tell user to fix migration and push manually.

**If `{prisma}` = `false` or `{database}` = `none`:** skip this step.

### 7. Clean up resources

Execute in order. If any fails, warn but continue:

**If `{database}` = `neon`:**
```bash
neonctl branches delete <db_branch> --force
```

**Always:**
```bash
git worktree remove .worktrees/<name> --force
git branch -D <type>/<name>
```

### 8. Update WORKSTREAMS.md

1. Remove the row from "Active Workstreams" table
2. Add row to "History" table with merge date and SonarQube status

```bash
git add WORKSTREAMS.md
git commit -m "worktree: finish workstream <name>"
```

### 9. Push

```bash
git push origin {base_branch}
```

If push fails, retry with rebase (max 3 retries):
```bash
git pull --rebase origin {base_branch} && git push origin {base_branch}
```

### 10. Output confirmation

Print a summary of what was done. Only list items that actually applied:

```
Workstream <name> completed!

Tests passed
Coverage delta check passed
SonarQube quality gate passed
Merged <type>/<name> to {base_branch}
Prisma migrations applied (only if prisma=true and database=neon)
Database branch deleted (only if database=neon)
Worktree removed
Branch deleted
WORKSTREAMS.md updated
Pushed to origin
```
