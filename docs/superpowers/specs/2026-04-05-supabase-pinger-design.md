# Supabase Pinger — Design Spec

## Problem

Supabase pauses free-tier projects after 7 consecutive days of inactivity. Projects with no API or database requests get shut down, requiring manual restoration from the dashboard.

## Solution

A GitHub Actions workflow that runs daily, reads a list of Supabase projects from a config file, and pings each one's REST API to register activity and prevent pausing.

## Architecture

Pure GitHub Actions — no runtime, no dependencies, no deployment. The workflow uses `curl` to hit each project's built-in PostgREST endpoint.

### Project Structure

```
supabase-pinger/
├── .github/
│   └── workflows/
│       └── ping.yml          # Cron-scheduled workflow
├── config/
│   └── projects.json         # Project list
└── README.md                 # Setup instructions
```

## Config Format

`config/projects.json` — an array of project entries:

```json
[
  {
    "name": "my-app",
    "url": "https://abcdefghij.supabase.co",
    "keySecret": "SUPABASE_ANON_KEY_MY_APP"
  }
]
```

| Field | Description |
|-------|-------------|
| `name` | Human-readable label used in log output |
| `url` | The project's Supabase URL (from Dashboard > Settings > API) |
| `keySecret` | Name of the GitHub repo secret holding the project's anon key |

Actual anon keys are stored as GitHub repo secrets, never in the config file.

## Ping Mechanism

Each project is pinged with:

```
GET {url}/rest/v1/
Headers:
  apikey: {anon-key}
  Authorization: Bearer {anon-key}
```

This hits the PostgREST schema endpoint, which is auto-generated on every Supabase project. No tables, functions, or custom endpoints are required. A 200 response confirms the project is alive and resets the 7-day inactivity timer.

## Workflow Design

**File:** `.github/workflows/ping.yml`

**Triggers:**
- `schedule`: cron — once daily at 08:00 UTC
- `workflow_dispatch`: manual trigger for testing

**Steps:**
1. Checkout the repo to access `config/projects.json`
2. Read the config file with `jq`
3. Loop through each project entry
4. Resolve the anon key from the GitHub secret named in `keySecret`
5. `curl` the REST API endpoint with appropriate headers
6. Log the project name, HTTP status code, and success/failure
7. Track failures; exit non-zero if any ping failed

**Failure behavior:** If any ping returns a non-200 status, the workflow exits with a non-zero code. GitHub's built-in notification system alerts the repo owner on workflow failure.

## Secret Management

Anon keys are stored as GitHub repo secrets. The config file references secrets by name, and the workflow resolves them at runtime. This means:

- Keys never appear in code or logs
- Each project gets its own secret (e.g., `SUPABASE_ANON_KEY_MY_APP`)
- Adding a project requires adding both a config entry and a repo secret

## Adding/Removing Projects

1. Edit `config/projects.json` — add or remove an entry
2. In GitHub repo settings, add or remove the corresponding secret
3. Next scheduled run picks up the change automatically

## Observability

- Each ping logs the project name, HTTP status, and pass/fail
- A summary line reports total projects pinged and any failures
- Results visible in the GitHub Actions tab
- Workflow failure triggers GitHub's default notification (email to repo owner)

## Scope Boundaries

**In scope:**
- Daily cron ping via GitHub Actions
- JSON config for project management
- Per-project logging with pass/fail status

**Out of scope:**
- Retry logic (daily cadence provides a 7x safety margin)
- External notifications (GitHub's built-in failure alerts are sufficient)
- Dashboard or UI
- Automatic project discovery via Supabase Management API
