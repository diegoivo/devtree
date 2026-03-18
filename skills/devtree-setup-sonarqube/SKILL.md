---
name: devtree setup-sonarqube
description: Bootstrap local SonarQube for code quality analysis (once per machine)
---

# /devtree setup-sonarqube

One-time bootstrap of a local SonarQube instance with PostgreSQL via Docker Compose. Run this once per machine.

## Step 1: Check prerequisites

```bash
docker --version 2>/dev/null || echo "MISSING: docker"
docker compose version 2>/dev/null || echo "MISSING: docker compose"
which sonar-scanner 2>/dev/null || echo "MISSING: sonar-scanner"
```

- If Docker is missing → **STOP**: "Install Docker Desktop: https://docker.com/products/docker-desktop"
- If `docker compose` is missing → **STOP**: "Docker Compose V2 required. Update Docker Desktop."
- If sonar-scanner is missing:
  - If `brew` exists → run `brew install sonar-scanner`
  - Else → **STOP**: "Install sonar-scanner manually: https://docs.sonarsource.com/sonarqube/latest/analyzing-source-code/scanners/sonarscanner/"

## Step 2: Create ~/.sonarqube/ and write docker-compose.yml

```bash
mkdir -p ~/.sonarqube
```

Write the following content to `~/.sonarqube/docker-compose.yml` using the Write tool:

```yaml
services:
  sonarqube:
    image: sonarqube:lts-community
    container_name: devtree-sonarqube
    depends_on:
      db:
        condition: service_healthy
    ports:
      - "9000:9000"
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonarqube
      SONAR_JDBC_USERNAME: sonarqube
      SONAR_JDBC_PASSWORD: sonarqube
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs
    deploy:
      resources:
        limits:
          memory: 3g
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "curl -sf http://localhost:9000/api/system/status | grep -q UP"]
      interval: 30s
      timeout: 10s
      retries: 10
      start_period: 120s

  db:
    image: postgres:16-alpine
    container_name: devtree-sonarqube-db
    environment:
      POSTGRES_USER: sonarqube
      POSTGRES_PASSWORD: sonarqube
      POSTGRES_DB: sonarqube
    volumes:
      - sonarqube_pg:/var/lib/postgresql/data
    deploy:
      resources:
        limits:
          memory: 512m
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U sonarqube"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_logs:
  sonarqube_pg:
```

## Step 3: Start containers

```bash
cd ~/.sonarqube && docker compose up -d
```

Check port 9000 availability first:
```bash
lsof -i :9000 -sTCP:LISTEN 2>/dev/null
```
If port 9000 is occupied by something other than `devtree-sonarqube`, **STOP**: "Port 9000 is in use. Stop the conflicting service or adjust the SonarQube port in ~/.sonarqube/docker-compose.yml."

## Step 4: Wait for SonarQube to be ready

Poll the health endpoint with a 3-minute timeout:

```bash
timeout 180 bash -c 'until curl -sf http://localhost:9000/api/system/status | grep -q UP; do sleep 5; done'
```

If timeout → **STOP**: "SonarQube did not start within 3 minutes. Check logs: `cd ~/.sonarqube && docker compose logs sonarqube`"

## Step 5: Change default admin password

Generate a random password and change the default `admin/admin` credentials:

```bash
NEW_PASS=$(openssl rand -hex 16)
RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" -u admin:admin \
  -X POST "http://localhost:9000/api/users/change_password" \
  -d "login=admin&previousPassword=admin&password=$NEW_PASS")
```

**Idempotency handling:**
- HTTP 204 → password changed successfully, use `$NEW_PASS` going forward
- HTTP 401 → password was already changed in a previous run. Read existing `SONAR_ADMIN_PASS` from the shell rc file:
  - If `$SONAR_ADMIN_PASS` exists and authenticates (`curl -sf -u "admin:$SONAR_ADMIN_PASS" http://localhost:9000/api/system/status` returns 200), use it and skip to Step 6
  - If it also fails → **STOP**: "Admin password was changed but SONAR_ADMIN_PASS is not set or incorrect. Reset SonarQube with `cd ~/.sonarqube && docker compose down -v && docker compose up -d` and re-run setup."

## Step 6: Generate API token

```bash
SONAR_TOKEN=$(curl -sf -u "admin:$RESOLVED_PASS" \
  -X POST "http://localhost:9000/api/user_tokens/generate" \
  -d "name=devtree-cli&type=GLOBAL_ANALYSIS" | grep -o '"token":"[^"]*"' | cut -d'"' -f4)
```

Where `$RESOLVED_PASS` is whichever password was resolved in Step 5 (`$NEW_PASS` on fresh run, or `$SONAR_ADMIN_PASS` on re-run).

If token generation fails because `devtree-cli` already exists, revoke it first:
```bash
curl -sf -u "admin:$RESOLVED_PASS" \
  -X POST "http://localhost:9000/api/user_tokens/revoke" \
  -d "name=devtree-cli"
```
Then retry generation.

## Step 7: Save to shell rc file

Detect the user's shell and write to the appropriate rc file:

```bash
RC_FILE="$HOME/.zshrc"  # default
case "$SHELL" in
  */bash) RC_FILE="$HOME/.bashrc" ;;
  */fish) RC_FILE="$HOME/.config/fish/config.fish" ;;
esac
```

Check if `# --- devtree sonarqube ---` block already exists in `$RC_FILE`. If it does, replace the entire block (from `# --- devtree sonarqube ---` to the next blank line or `# ---` marker). If not, append it.

For bash/zsh:
```bash
# --- devtree sonarqube ---
export SONAR_TOKEN="<token>"
export SONAR_HOST_URL="http://localhost:9000"
export SONAR_ADMIN_PASS="<password>"
```

For fish:
```fish
# --- devtree sonarqube ---
set -gx SONAR_TOKEN "<token>"
set -gx SONAR_HOST_URL "http://localhost:9000"
set -gx SONAR_ADMIN_PASS "<password>"
```

## Step 8: Check sonar-project.properties

Check if `sonar-project.properties` exists in the current project root.

- **If missing:** suggest creating a minimal template:
  ```properties
  sonar.projectKey=<org>_<repo>
  sonar.projectName=<project-name>
  sonar.sources=src
  sonar.host.url=http://localhost:9000
  ```

- **If exists but `sonar.host.url` points to `sonarcloud.io`:** inform the user that `/devtree scan` passes `-Dsonar.host.url=$SONAR_HOST_URL` explicitly to the scanner CLI, which overrides the properties file value. No need to edit the file — projects can keep their SonarCloud config for CI while using local SonarQube via devtree.

## Step 9: Output

Print the following, substituting `<RC_FILE>` with the actual path detected in Step 7:

```
SonarQube setup complete!

  Dashboard:  http://localhost:9000
  Admin user: admin (password saved in <RC_FILE>)
  Token:      saved as SONAR_TOKEN in <RC_FILE>

  Run: source <RC_FILE>

  Next steps:
    1. Ensure sonar-project.properties exists in your project root
    2. Run /devtree scan to analyze your code
```
