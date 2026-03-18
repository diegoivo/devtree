---
name: scan
description: Run local SonarQube code quality analysis - quick scan or full quality gate
---

# /devtree scan [--full]

Runs local SonarQube analysis on the current codebase.

## Arguments

- `--full` (optional): After scanning, check the quality gate. Without this flag, runs a quick local scan only.

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
6. Continue with scan

Parse the config into variables: `{coverage_command}`, `{app_paths}`, etc.

## Step 1: Prerequisites check

```bash
which sonar-scanner 2>/dev/null || echo "MISSING: sonar-scanner. Run: brew install sonar-scanner"
echo $SONAR_TOKEN | head -c 5 || echo "SONAR_TOKEN not set. Run /devtree setup-sonarqube first."
curl -sf http://localhost:9000/api/system/status | grep -q UP || echo "SonarQube not running. Run /devtree setup-sonarqube first."
```

If any prerequisite is missing → **STOP** with the appropriate message.

## Step 2: Generate coverage

If `{coverage_command}` is set in the config:

```bash
{coverage_command} 2>&1 | tail -5
```

Then remove HTML reports to avoid polluting the scan:
```bash
find . -path "*/coverage/lcov-report" -type d -exec rm -rf {} + 2>/dev/null
```

If `{coverage_command}` is NOT set, skip and warn: "No coverage_command configured. Coverage metrics will not be available in the scan."

## Step 3: Read sonar-project.properties

Read `sonar-project.properties` from the project root. Extract:
- `sonar.projectKey` → used for quality gate API call
- `sonar.javascript.lcov.reportPaths` or `sonar.typescript.lcov.reportPaths` → used as coverage path

If the file does not exist → **STOP**: "sonar-project.properties not found. Create one or run /devtree setup-sonarqube for a template."

## Step 4: Run scanner

```bash
sonar-scanner \
  -Dsonar.token=$SONAR_TOKEN \
  -Dsonar.host.url=$SONAR_HOST_URL \
  -Dsonar.javascript.lcov.reportPaths=<coverage_path_from_properties>
```

The `-Dsonar.host.url=$SONAR_HOST_URL` flag overrides whatever `sonar.host.url` value is in the properties file. This allows projects to keep their SonarCloud config for CI while using local SonarQube via devtree.

Parse the scanner output and present: total issues by severity, bugs, code smells, security hotspots, coverage.

## Step 5: Full scan (--full only)

After the scan completes, check the quality gate:

```bash
curl -sf -u "$SONAR_TOKEN:" \
  "http://localhost:9000/api/qualitygates/project_status?projectKey=<projectKey>"
```

Parse the JSON response:
- `projectStatus.status` = `OK` → "Quality Gate: PASSED"
- `projectStatus.status` = `ERROR` → "Quality Gate: FAILED" (show which conditions failed from `projectStatus.conditions`)

## Output Format

```
SonarQube Analysis Results

Quality Gate: PASSED / FAILED (only shown with --full)

Bugs:            0 (A)
Vulnerabilities: 0 (A)
Code Smells:     15 (A)
Coverage:        82.3%
Duplications:    3.1%

View full report: http://localhost:9000/dashboard?id=<projectKey>
```
