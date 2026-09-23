# App Deploy (Dummy) Workflow

A placeholder deploy workflow: `Webhook → Fake Deploy → Respond`. It validates the request and
returns a realistic-looking success response, but **does not actually build or run anything** —
no Docker image, no container, no real server. It exists so the request/response shape of a
"deploy" step is settled before wiring up a real one.

**File:** [`workflows/app-deploy-dummy-workflow.json`](../workflows/app-deploy-dummy-workflow.json)

## Setup

No credentials needed. Import the file, save, toggle **Active**.

## Usage

```bash
curl -X POST http://localhost:5678/webhook/deploy-app \
  -H "Content-Type: application/json" \
  -d '{"outputDir": "todo-api", "hostPort": 4000}'
```

Response:
```json
{
  "status": "ok",
  "note": "DUMMY DEPLOYMENT - this is a placeholder. No Docker image was built, no container was started, and nothing is actually reachable at the URL below.",
  "outputDir": "todo-api",
  "containerName": "generated-todo-api",
  "simulatedUrl": "http://localhost:4000",
  "deployedAt": "2026-09-23T12:00:00.000Z"
}
```

## What a real version would need

This was scoped down from an actual Docker-based deploy after weighing the tradeoff, so the
notes are here for when that's picked back up:

- Building/running a container from *inside* the n8n container means mounting the host's Docker
  socket (`/var/run/docker.sock`) into it and adding the `docker` CLI to the image (the stock
  `n8nio/n8n` image has neither). That combination gives n8n — and anything it executes,
  including AI-generated `npm install` output in the self-testing workflow — root-equivalent
  control over the host. Worth doing only on a disposable/sandboxed machine, never one with
  real credentials or data on it.
- The two app-generator workflows already require generated backend projects to expose a
  `start` script and listen on `process.env.PORT` (defaulting to `3000`), specifically so a real
  deploy step has a predictable convention to build a Dockerfile and port-map around.
- The real version would: write a generic Node Dockerfile into the generated project's
  directory, `docker build` it, `docker rm -f` any previous container with the same name, then
  `docker run -d -p <hostPort>:3000` it — all as one shell script (same pattern as the
  self-testing workflow's `Build Script` → `Run Script` nodes), with the exit code checked before
  reporting success.
