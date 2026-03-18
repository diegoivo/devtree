---
name: start
description: Create a new worktree with isolated database, cache, port, and branch for parallel feature development
---

# /devtree start <name> [--type feat|fix|chore]

Creates a git worktree with fully isolated resources for parallel development.

## Arguments

- `<name>` (required): Feature name (e.g., `geo-monitor-dashboard`)
- `--type` (optional): Branch type prefix. Default: `feat`. Options: `feat`, `fix`, `chore`

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
6. Continue with start

Parse the config into variables: `{base_branch}`, `{app_paths}`, `{database}`, `{prisma}`, `{redis}`, `{redis_prefix_template}`, `{port_range_start}`, `{port_range_end}`, `{port_step}`, `{setup_command}`, `{dev_command}`, `{env_overrides}`.

## Execution Steps

Follow these steps IN ORDER. If any step fails, execute the rollback procedure at the bottom.

### 1. Parse arguments

Extract `<name>` and `--type` (default `feat`) from the user's input. Derive:
- Branch name: `<type>/<name>`
- Database branch name: `<type>-<name>` (no slashes — only used if `{database}` != `none`)
- Worktree path: `.worktrees/<name>`
- Redis prefix: substitute `{name}` into `{redis_prefix_template}`

### 2. Ensure .worktrees directory exists

```bash
mkdir -p .worktrees
```

### 3. Pull latest base branch

```bash
git checkout {base_branch} && git pull --rebase origin {base_branch}
```

### 4. Read WORKSTREAMS.md and allocate resources

Read `WORKSTREAMS.md` from the project root. If the file does not exist, create it with this template:

```markdown
# Workstreams

## Active Workstreams

| Feature | Branch | Port | DB Branch | Redis Prefix | Owner | Status | Started |
|---------|--------|------|-----------|--------------|-------|--------|---------|

## History

| Feature | Branch | Merged | SonarQube Status |
|---------|--------|--------|------------------|
```

Find the next available port by checking which slots (`{port_range_start}`, `{port_range_start} + {port_step}`, ..., `{port_range_end}`) are NOT listed in the active table. Verify the port is free:

```bash
lsof -i :<port> 2>/dev/null
```

If the port is occupied, try the next available range. If ALL slots are used, **STOP** and tell the user to finish a workstream first.

### 5. Create git branch and worktree

```bash
git branch <type>/<name>
git worktree add .worktrees/<name> <type>/<name>
```

### 6. Create database branch (conditional)

**If `{database}` = `neon`:**

```bash
neonctl branches create --name <type>-<name> --output json
```

Parse the JSON output to extract:
- `connectionString` (pooled) → `DATABASE_URL`
- Direct connection string → `DIRECT_DATABASE_URL`

If `neonctl` is not installed, tell the user: `npm i -g neonctl && neonctl auth`

**If `{database}` = `none`:** skip this step entirely.

**If `{database}` = any other value:** **STOP**: "Database type '{database}' is not supported yet. Supported: neon, none."

### 7. Generate .env.local in worktree

For each path in `{app_paths}`:

1. Read `{path}/.env.local` from the MAIN project root (NOT the worktree)
2. Create a NEW `{path}/.env.local` inside the worktree at `.worktrees/<name>/{path}/.env.local` with these changes:
   - `PORT=<allocated_port>`
   - If `{database}` = `neon`: override `DATABASE_URL` and `DIRECT_DATABASE_URL` with the Neon branch values
   - If `{redis}` != `none`: add `UPSTASH_REDIS_PREFIX=<redis_prefix>` (from `{redis_prefix_template}` with `{name}` substituted)
   - Apply each `{env_overrides}` entry, substituting `{port}` with the allocated port and `{name}` with the feature name
3. All other env vars are copied unchanged

### 8. Run project setup

```bash
cd .worktrees/<name> && {setup_command}
```

### 9. Update WORKSTREAMS.md

Add a new row to the "Active Workstreams" table:

```
| <name> | <type>/<name> | <port> | <db_branch_or_-> | <redis_prefix> | <user> | active | <YYYY-MM-DD> |
```

Use `-` for DB Branch if `{database}` = `none`.

### 10. Commit and push WORKSTREAMS.md

```bash
git checkout {base_branch}
git add WORKSTREAMS.md
git commit -m "worktree: start workstream <name>"
git push origin {base_branch}
```

If push fails (conflict), retry:
```bash
git pull --rebase origin {base_branch} && git push origin {base_branch}
```
Max 3 retries.

### 11. Output confirmation

Print:
```
Worktree ready at .worktrees/<name>
Branch: <type>/<name>
Port: <port>
Database branch: <db_branch> (or "none" if database=none)
Redis prefix: <redis_prefix>

Start dev server:
  cd .worktrees/<name> && {dev_command with {port} substituted}
```

## Rollback Procedure

If ANY step fails after resources were created, undo in reverse order — only clean up resources that were actually created based on config:

1. If database branch was created AND `{database}` = `neon`: `neonctl branches delete <type>-<name> --force`
2. If worktree was created: `git worktree remove .worktrees/<name> --force`
3. If git branch was created: `git branch -D <type>/<name>`
4. If WORKSTREAMS.md was modified: revert the change

Always leave the system in a clean state.
