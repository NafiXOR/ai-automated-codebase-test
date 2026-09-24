# AI App Generator (Test + Deploy) Workflow

The full pipeline: generate → test-and-fix loop → **on success, actually deploy the dev version
as a running Docker container.** This extends
[app-generator-with-tests-workflow.json](../workflows/app-generator-with-tests-workflow.json)
with a real deploy step in place of just reporting "tests passed."

**File:** [`workflows/app-generator-test-deploy-workflow.json`](../workflows/app-generator-test-deploy-workflow.json)

## How it works

```
Webhook ─► Prepare Prompt ─► Call Gemini ─► Build Script ─► Run Script ─► Evaluate Result ─► Tests Passed?
             ▲                                                                                  │      │
             │                                                                                yes│    no
             │                                            Summarize ─► Build Deploy Script ─► Run Deploy ─► Evaluate Deploy ─► Respond Success
             │                                                                                         │
             │                                                                                   Retries Left?
             │                                                                                    yes │  │no
             └──────────────────────────────── Prepare Fix Data ◄─────────────────────────────────────┘  │
                                                                                                            ▼
                                                                                                        Give Up ─► Respond Failure
```

Same generate/test/retry mechanics as the self-testing workflow (see
[docs/app-generator-self-testing-workflow.md](app-generator-self-testing-workflow.md) for how
that loop works). What's new is the success path:

- **Build Deploy Script** writes a generic Node Dockerfile and a `.dockerignore` (which keeps
  the test run's `node_modules`, dev dependencies included, out of the image) into the generated
  project's directory, then constructs a shell script that removes any previous container with the same
  name, builds a fresh image, and runs it with the requested host port mapped to the app's port.
- **Run Deploy** (Execute Command) actually runs that script.
- **Evaluate Deploy** checks the exit code and reports `deployed` or `tests_passed_deploy_failed`.

Deployment failure does **not** loop back into the fix cycle — a failed `docker build`/`docker
run` is an infrastructure problem, not something regenerating the code will fix.

## Why this needed infrastructure changes

Two things were added, both **opt-in** rather than baked into the default setup (see why below):

1. **`Dockerfile`** (repo root) — extends `n8nio/n8n:latest` with the `docker` CLI installed
   (the stock image doesn't have it).
2. **`docker-compose.deploy.yml`** — an override file that builds from that Dockerfile and mounts
   the **host's Docker socket** (`/var/run/docker.sock`) into the n8n container. The base
   `docker-compose.yml` is untouched, so everyone using the starter or self-testing workflows
   isn't silently exposed to this.

### ⚠️ Security tradeoff — read this before enabling it

Mounting the Docker socket into a container gives that container (and anything it can execute)
root-equivalent control over the **host** machine — it can create, inspect, or remove any
container, mount the host filesystem into a new one, etc. This is the standard way to let a
container manage sibling containers ("Docker outside of Docker"), but it means:

- Anything that can get `n8n`'s Execute Command node to run arbitrary shell (which, in this
  repo, the self-testing workflow already does via AI-generated `npm install`/`npm test`) now
  has a path to full host compromise, e.g. via a malicious transitive npm dependency during
  install.
- **Only run this on a disposable or sandboxed machine/VM** — never one with real credentials,
  other production containers, or sensitive data on it.

If that tradeoff isn't acceptable, use
[app-generator-with-tests-workflow.json](app-generator-self-testing-workflow.md) (generate + test,
no deploy) or the placeholder
[app-deploy-dummy-workflow.json](app-deploy-dummy-workflow.md) instead.

## Setup

1. **Start n8n with the deploy override enabled** (builds the Docker-CLI-enabled image and
   mounts the host's Docker socket):
   ```bash
   docker compose -f docker-compose.yml -f docker-compose.deploy.yml up -d --build
   ```
   From now on, use this two-file `-f ... -f ...` form instead of plain `docker compose up -d`
   whenever you want the deploy step available (stop/logs/etc. work with the plain form since
   they don't need the override's config).
2. Create/reuse the **Gemini API** Header Auth credential (`Authorization` = `Bearer <key>`) and
   check it's selected on the **Call Gemini** node, same as the other generator workflows. The webhook has no
   token by default; since this one ends in `docker run` on your host, consider adding one
   ([how](app-generator-workflow.md#optional-protect-the-webhook-with-a-token)) once you're past
   testing.
3. Import [`workflows/app-generator-test-deploy-workflow.json`](../workflows/app-generator-test-deploy-workflow.json),
   save, toggle **Active**.
4. Confirm Docker Desktop (or your Docker daemon) is running on the host — the deploy step needs
   it reachable through the mounted socket.

## Usage

```bash
curl -X POST http://localhost:5678/webhook/generate-app-deployed \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "A REST API for a todo list with in-memory storage: GET/POST /todos, PATCH and DELETE /todos/:id",
    "target": "backend",
    "outputDir": "todo-api",
    "maxRetries": 3,
    "hostPort": 4001
  }'
```

`hostPort` (optional, default `4000`, must be 1024–65535) is the port on your machine the
deployed app will be reachable at. The container's internal port is always `3000` — the
generator's system prompt requires generated backend apps to listen on
`process.env.PORT` (defaulting to `3000`), specifically so this mapping is predictable.

**Success response:**
```json
{
  "status": "deployed",
  "outputDir": "todo-api",
  "retriesUsed": 1,
  "filesWritten": 5,
  "files": ["package.json", "server.js", "..."],
  "testOutput": "...npm test output...",
  "containerName": "generated-todo-api",
  "image": "generated-todo-api:latest",
  "url": "http://localhost:4001",
  "deployOutput": "...docker build/run output..."
}
```

**Tests passed but deploy failed:**
```json
{
  "status": "tests_passed_deploy_failed",
  "outputDir": "todo-api",
  "retriesUsed": 0,
  "deployOutput": "...docker build/run error output..."
}
```

**Gave up (tests never passed):** same `status: "failed"` shape as the self-testing workflow.

## What I actually verified

Unlike guessing at the shell mechanics, I built a mock generated app (a plain Node `http` server
with a `package.json` `start` script) and ran the **exact** shell script `Build Deploy Script`
constructs, for real: `docker build` on a generated Dockerfile, `docker run -d -p <port>:3000`,
then `curl`'d the running container and got the expected JSON response back, then tore down the
container and image. That end-to-end mechanism works. It was re-run after the `.dockerignore`
and exit-code-marker changes, from a `docker:cli` container talking to the host daemon through
the mounted socket (the same setup as the deploy override), and the image contained no
`node_modules` from the test run.

What's still unverified without a live n8n instance (same categories as the self-testing
workflow — see [its doc](app-generator-self-testing-workflow.md#what-i-verified-vs-whats-unverified)):
IF node condition schema, and Execute Command's output field names. n8n 2.x disables Execute
Command by default; `docker-compose.yml` re-enables it with `NODES_EXCLUDE`, so if the node
shows as unknown, check that variable reached the container (`docker compose exec n8n env`).

## Known limitations

- **Containers accumulate.** Nothing here stops or removes a deployed container automatically
  after you're done with it — `docker ps -a` and `docker rm -f <name>` to clean up manually.
  Re-deploying the same `outputDir` does replace its own container (via `docker rm -f` before
  build), but a different `outputDir` on the same `hostPort` will fail with a port-already-in-use
  error rather than resolving the conflict for you.
- **Frontend targets aren't really supported by this deploy step** — the generic Dockerfile
  assumes an `npm start` that boots a long-running server on `process.env.PORT`, which fits a
  backend but not a static frontend build.
- Same generation/testing caveats as the self-testing workflow apply: a "deployed" result only
  means the AI-written tests (which the AI also wrote) passed, not that the app is correct or
  safe to expose beyond your own machine.
