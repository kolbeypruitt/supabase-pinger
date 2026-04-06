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
