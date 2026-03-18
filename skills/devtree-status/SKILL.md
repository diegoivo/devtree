---
name: devtree status
description: Show all active workstreams with health indicators and resource status
---

# /devtree status

Shows active workstreams from WORKSTREAMS.md with live health checks.

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
6. Continue with status

Parse the config to get `{database}`.

## Execution Steps

### 1. Read WORKSTREAMS.md

Parse the "Active Workstreams" table from `WORKSTREAMS.md` at the project root.

If the file does not exist or has no active rows, print:
```
No active workstreams. Use /devtree start <name> to begin.
```

### 2. Health check each workstream

For each active row, check:

**Worktree exists?**
```bash
test -d .worktrees/<name> && echo "OK" || echo "MISSING"
```

**Git branch exists?**
```bash
git branch --list <branch> | grep -q <branch> && echo "OK" || echo "MISSING"
```

**Port responding?**
```bash
lsof -i :<port> -sTCP:LISTEN 2>/dev/null && echo "RUNNING" || echo "STOPPED"
```

**Database branch exists? (only if `{database}` = `neon`):**
```bash
neonctl branches list --output json 2>/dev/null | grep -q "<db_branch>" && echo "OK" || echo "MISSING"
```

### 3. Display formatted table

Adjust columns based on config — omit "DB Branch" column if `{database}` = `none`.

**With database:**
```
Active Workstreams

| Feature     | Branch           | Port | Server  | DB      | Worktree | Owner | Started    |
|-------------|------------------|------|---------|---------|----------|-------|------------|
| geo-monitor | feat/geo-monitor | 9100 | RUNNING | OK      | OK       | diego | 2026-03-14 |
| fix-scoring | fix/scoring      | 9200 | STOPPED | OK      | OK       | joao  | 2026-03-14 |
```

**Without database:**
```
Active Workstreams

| Feature     | Branch           | Port | Server  | Worktree | Owner | Started    |
|-------------|------------------|------|---------|----------|-------|------------|
| geo-monitor | feat/geo-monitor | 9100 | RUNNING | OK       | diego | 2026-03-14 |
```

### 4. Flag inconsistencies

If any resource is MISSING but the workstream is listed, warn:
```
Warning: <name> has missing resources. Consider running /devtree finish or manually cleaning up.
```
