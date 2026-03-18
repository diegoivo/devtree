# devtree

Claude Code plugin for git worktree management with isolated resources and local SonarQube code quality analysis.

## Installation

```
/plugin install devtree@diegoivo
```

## Requirements

- Docker Desktop (for SonarQube)
- sonar-scanner CLI (`brew install sonar-scanner`)

## Commands

| Command | Description |
|---------|-------------|
| `/devtree start <name> [--type feat\|fix\|chore]` | Create worktree with isolated database, cache, port, and branch |
| `/devtree finish` | Run quality gates, merge to base branch, clean up resources |
| `/devtree scan [--full]` | Run local SonarQube analysis (add `--full` for quality gate check) |
| `/devtree status` | Show active workstreams with live health checks |
| `/devtree setup-sonarqube` | Bootstrap local SonarQube (once per machine) |

## Quick Start

1. Install the plugin
2. Run `/devtree setup-sonarqube` to bootstrap local SonarQube
3. Run `source ~/.zshrc` (or your shell rc file) to load the token
4. Ensure `sonar-project.properties` exists in your project root
5. Use `/devtree start my-feature` to create an isolated worktree
6. Work on your feature, then use `/devtree finish` to merge

## Per-Project Config

On first use, devtree auto-detects your project settings and creates `.claude/worktree-config.md`:

```markdown
# Worktree Config
base_branch: main
app_paths:
  - apps/web
database: neon
prisma: true
redis: upstash
redis_prefix_template: "wt:{name}:"
port_range_start: 9100
port_range_end: 9900
port_step: 100
test_command: npx turbo test
coverage_command: cd apps/web && npx vitest run --coverage
dev_command: cd apps/web && npx next dev --port {port}
setup_command: npm install && cd apps/web && npx prisma generate
env_overrides:
  UPSTASH_WORKFLOW_URL: "http://localhost:{port}"
  NEXTAUTH_URL: "http://localhost:{port}"
```

This file is gitignored and project-specific. All fields except `base_branch` and `app_paths` are optional.

## Supported Database Adapters

| Database | Status |
|----------|--------|
| Neon | Supported — creates/deletes branches automatically |
| None | Supported — skips database isolation |
| Supabase | Planned |

## SonarQube

devtree runs SonarQube locally via Docker Compose (`~/.sonarqube/`). Projects can keep their `sonar-project.properties` pointing to SonarCloud for CI — devtree overrides the host URL via CLI flag for local scans.
