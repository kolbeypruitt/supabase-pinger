# Supabase Pinger Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Keep Supabase free-tier projects alive by pinging their REST APIs daily via GitHub Actions.

**Architecture:** A GitHub Actions cron workflow reads project entries from `config/projects.json`, resolves anon keys from repo secrets, and curls each project's PostgREST endpoint. Failures are logged per-project and cause the workflow to exit non-zero.

**Tech Stack:** GitHub Actions, bash, curl, jq

---

## File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `config/projects.json` | Create | Lists Supabase projects with name, URL, and secret reference |
| `.github/workflows/ping.yml` | Create | Cron-scheduled workflow that pings all projects |
| `README.md` | Create | Setup instructions for secrets and config |

---

### Task 1: Create the project config file

**Files:**
- Create: `config/projects.json`

- [ ] **Step 1: Create the example config**

```json
[
  {
    "name": "example-project",
    "url": "https://your-project-ref.supabase.co",
    "keySecret": "SUPABASE_ANON_KEY_EXAMPLE_PROJECT"
  }
]
```

This is a placeholder the user will replace with their actual projects. The `keySecret` field holds the **name** of the GitHub repo secret (not the key itself).

- [ ] **Step 2: Validate the JSON is parseable**

Run: `cat config/projects.json | jq .`
Expected: Pretty-printed JSON output, no errors.

- [ ] **Step 3: Commit**

```bash
git add config/projects.json
git commit -m "feat: add project config with example entry"
```

---

### Task 2: Create the GitHub Actions ping workflow

**Files:**
- Create: `.github/workflows/ping.yml`

- [ ] **Step 1: Create the workflow file**

```yaml
name: Ping Supabase Projects

on:
  schedule:
    - cron: '0 8 * * *' # Daily at 08:00 UTC
  workflow_dispatch: # Manual trigger

jobs:
  ping:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Ping all projects
        env:
          ALL_SECRETS: ${{ toJson(secrets) }}
        run: |
          FAILED=0
          TOTAL=0

          while IFS= read -r project; do
            name=$(echo "$project" | jq -r '.name')
            url=$(echo "$project" | jq -r '.url')
            key_secret=$(echo "$project" | jq -r '.keySecret')

            # Resolve the anon key from secrets
            anon_key=$(echo "$ALL_SECRETS" | jq -r --arg key "$key_secret" '.[$key] // empty')

            if [ -z "$anon_key" ]; then
              echo "::error::[$name] Secret '$key_secret' not found in repo secrets"
              FAILED=$((FAILED + 1))
              TOTAL=$((TOTAL + 1))
              continue
            fi

            # Ping the REST API
            status=$(curl -s -o /dev/null -w "%{http_code}" \
              "${url}/rest/v1/" \
              -H "apikey: ${anon_key}" \
              -H "Authorization: Bearer ${anon_key}")

            TOTAL=$((TOTAL + 1))

            if [ "$status" -eq 200 ]; then
              echo "[$name] OK (HTTP $status)"
            else
              echo "::error::[$name] FAILED (HTTP $status)"
              FAILED=$((FAILED + 1))
            fi
          done < <(jq -c '.[]' config/projects.json)

          echo ""
          echo "--- Summary ---"
          echo "Total: $TOTAL | Passed: $((TOTAL - FAILED)) | Failed: $FAILED"

          if [ "$FAILED" -gt 0 ]; then
            exit 1
          fi
```

Key details:
- `toJson(secrets)` exposes all repo secrets as a JSON object so we can dynamically look up secrets by name from the config
- Each project is processed from the JSON array via `jq -c '.[]'`
- `curl -s -o /dev/null -w "%{http_code}"` returns only the HTTP status code
- GitHub's `::error::` annotation marks failures visibly in the Actions UI
- Non-zero exit triggers GitHub's failure notification

- [ ] **Step 2: Validate the workflow YAML syntax**

Run: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/ping.yml'))"`
Expected: No errors (exits silently on success).

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/ping.yml
git commit -m "feat: add daily ping workflow for Supabase projects"
```

---

### Task 3: Create the README

**Files:**
- Create: `README.md`

- [ ] **Step 1: Write the README**

```markdown
# Supabase Pinger

Keeps your Supabase free-tier projects alive by pinging them daily. Supabase pauses inactive projects after 7 days — this prevents that.

## How It Works

A GitHub Actions workflow runs once daily and sends an HTTP request to each configured Supabase project's REST API. Any successful request resets the inactivity timer.

## Setup

### 1. Fork or clone this repo

### 2. Add your projects to `config/projects.json`

```json
[
  {
    "name": "my-app",
    "url": "https://your-project-ref.supabase.co",
    "keySecret": "SUPABASE_ANON_KEY_MY_APP"
  }
]
```

| Field | Description |
|-------|-------------|
| `name` | A label for log output |
| `url` | Your project URL (Dashboard > Settings > API) |
| `keySecret` | The name of the GitHub secret holding your anon key |

### 3. Add secrets to your GitHub repo

For each project, go to **Settings > Secrets and variables > Actions** and add a secret:
- Name: must match the `keySecret` value in your config (e.g., `SUPABASE_ANON_KEY_MY_APP`)
- Value: your project's anon/public key (Dashboard > Settings > API)

### 4. Enable the workflow

The workflow runs automatically once daily at 08:00 UTC. You can also trigger it manually from the **Actions** tab.

## Adding a Project

1. Add an entry to `config/projects.json`
2. Add the corresponding secret in GitHub repo settings

## Removing a Project

1. Remove the entry from `config/projects.json`
2. Optionally remove the secret from GitHub repo settings

## Monitoring

Check the **Actions** tab to see ping results. Each run logs the HTTP status per project. If any ping fails, the workflow fails and GitHub sends a notification.
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add README with setup instructions"
```

---

### Task 4: Test the workflow locally

**Files:** None (verification only)

- [ ] **Step 1: Validate all files exist and are well-formed**

Run:
```bash
cat config/projects.json | jq .
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/ping.yml'))"
cat README.md | head -5
```
Expected: JSON parses, YAML parses, README header visible.

- [ ] **Step 2: Dry-run the ping logic locally (optional)**

If you have a real Supabase project to test with:
```bash
url="https://your-project-ref.supabase.co"
key="your-anon-key"
status=$(curl -s -o /dev/null -w "%{http_code}" \
  "${url}/rest/v1/" \
  -H "apikey: ${key}" \
  -H "Authorization: Bearer ${key}")
echo "HTTP $status"
```
Expected: `HTTP 200`

- [ ] **Step 3: Update config with real projects**

Replace the example entry in `config/projects.json` with your actual Supabase projects. Each entry needs:
- The project name (your choice)
- The project URL from Dashboard > Settings > API
- A secret name you'll create in GitHub (e.g., `SUPABASE_ANON_KEY_<PROJECT_NAME>`)

- [ ] **Step 4: Commit config updates**

```bash
git add config/projects.json
git commit -m "chore: configure real Supabase projects"
```
