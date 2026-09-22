# n8n Automated Workflow Setup Guide

This guide walks through running [n8n](https://n8n.io) locally with Docker and importing the
generic starter workflow included in this repo. A visual, step-by-step version of this same
guide is available at [n8n-setup-guide.html](n8n-setup-guide.html) — open it in any browser.

## What you'll end up with

- n8n running locally in Docker, reachable at `http://localhost:5678`
- Persistent workflow/credential storage in a Docker volume (survives restarts)
- A starter workflow imported and running: **Webhook → Set Data → Respond to Webhook**,
  which you can rename, extend, or use as a template for real automations

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and Docker Compose (bundled with Docker Desktop)
- `curl` (or any HTTP client) to test the webhook later — optional

Verify Docker is installed:

```bash
docker --version
docker compose version
```

## 1. Configure environment variables

Copy the example env file and edit the credentials:

```bash
cp .env.example .env
```

Open `.env` and set a real login for the n8n UI:

```
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=choose-a-strong-password
```

`.env` is git-ignored, so your credentials never get committed.

## 2. Start n8n

From the repo root:

```bash
docker compose up -d
```

Check that the container is healthy:

```bash
docker compose ps
docker compose logs -f n8n
```

Press `Ctrl+C` to stop following logs (this does not stop the container).

## 3. Log in to the n8n editor

Open [http://localhost:5678](http://localhost:5678) in your browser and log in with the
`N8N_BASIC_AUTH_USER` / `N8N_BASIC_AUTH_PASSWORD` you set in `.env`.

## 4. Import the starter workflow

1. In the n8n editor, click **Workflows** in the left sidebar, then **Add workflow**.
2. Click the **⋯** menu (top right) → **Import from File**.
3. Select [`workflows/starter-workflow.json`](../workflows/starter-workflow.json) from this repo.
4. Click **Save**, then toggle **Active** (top right) to turn the workflow on.

The imported workflow has three nodes:

| Node | Purpose |
|---|---|
| **Webhook** | Trigger — listens on `POST /webhook/starter-webhook` |
| **Set Response Data** | Adds a `message` and `receivedAt` timestamp to the payload |
| **Respond to Webhook** | Returns the processed data as JSON |

## 5. Test the workflow

With the workflow active, trigger it:

```bash
curl -X POST http://localhost:5678/webhook/starter-webhook \
  -H "Content-Type: application/json" \
  -d '{"hello": "world"}'
```

You should get back a JSON response containing your original payload plus `message` and
`receivedAt` fields. If nothing responds, open the workflow in the editor and check
**Executions** in the left sidebar for errors.

## 6. Build your own automation

Use the starter workflow as a template:

- Replace the **Webhook** node with whatever should kick off your automation (schedule trigger,
  form trigger, another app's trigger node, etc.)
- Replace **Set Response Data** with the real processing steps (HTTP Request, Function/Code,
  If, database nodes, etc.)
- Replace **Respond to Webhook** with a notification step (Slack, Email, Discord, etc.) if the
  workflow doesn't need to return an HTTP response

Export any workflow you build back into this repo for version control:
**⋯ menu → Download** and save it into `workflows/`.

## Day-to-day operations

| Task | Command |
|---|---|
| Stop n8n | `docker compose stop` |
| Start it again | `docker compose start` |
| Stop and remove the container (data persists) | `docker compose down` |
| View logs | `docker compose logs -f n8n` |
| Update to the latest n8n image | `docker compose pull && docker compose up -d` |
| Full reset (⚠️ deletes all workflows/credentials) | `docker compose down -v` |

## Troubleshooting

- **Port 5678 already in use** — stop whatever else is using it, or change the left-hand port in
  the `ports:` mapping in `docker-compose.yml` (e.g. `"5679:5678"`) and update `WEBHOOK_URL`
  accordingly.
- **Can't log in** — confirm `.env` was created (`cp .env.example .env`) and that
  `docker compose up -d` was run *after* editing it. Restart with `docker compose up -d --force-recreate`
  if you changed `.env` after the container was already running.
- **Webhook returns 404** — make sure the workflow is toggled **Active**, and that you're using
  the production URL (`/webhook/...`) rather than the test URL shown while editing
  (`/webhook-test/...`), which only works while the editor is open and "listening."
- **Losing data after restart** — confirm the `n8n_data` named volume wasn't removed
  (`docker compose down -v` deletes it; plain `docker compose down` does not).
