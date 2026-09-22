# AI App Generator Workflow

An n8n workflow that takes a plain-language description of an app and generates a runnable
backend or frontend scaffold using the Claude API, writing the generated files straight to disk.

**How it works:** `Webhook → Prepare Prompt → Call Claude → Parse Files → Write to Disk → Respond`

> **Scope check:** this is a one-shot generator, not an autonomous coding agent. Claude writes
> the files in a single pass with no chance to test or fix its own output — great for scaffolds,
> boilerplate, and quick prototypes; not a replacement for actually building and iterating on a
> real project.

## Prerequisites

- n8n running via this repo's `docker-compose.yml` (see [n8n-setup-guide.md](n8n-setup-guide.md))
- An [Anthropic API key](https://console.anthropic.com/settings/keys) — this workflow calls the
  Claude API directly and billing applies per request

## 1. Add your Anthropic API key as a credential

The workflow authenticates with a generic header credential rather than a hardcoded key, so it's
never stored in the workflow JSON.

1. In n8n, go to **Credentials** → **Add credential** → **Header Auth**.
2. Set:
   - **Name**: `Anthropic API` (any name is fine, you'll select it in step 2 below)
   - **Header Name**: `x-api-key`
   - **Header Value**: your Anthropic API key
3. Save.

## 2. Import and configure the workflow

1. **Workflows** → **Add workflow** → **⋯** → **Import from File** →
   [`workflows/app-generator-workflow.json`](../workflows/app-generator-workflow.json).
2. Open the **Call Claude** node and set its **Credential for Header Auth** to the
   `Anthropic API` credential you just created.
3. Save the workflow, then toggle **Active**.

## 3. Generate an app

POST a description of what you want built to the webhook:

```bash
curl -X POST http://localhost:5678/webhook/generate-app \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "A REST API for a todo list with in-memory storage: GET/POST /todos, PATCH and DELETE /todos/:id",
    "target": "backend",
    "outputDir": "todo-api"
  }'
```

| Field | Required | Description |
|---|---|---|
| `prompt` | yes | Plain-language description of the app to generate |
| `target` | no | `"backend"` (default, Node/Express) or `"frontend"` (React SPA) |
| `outputDir` | no | Subfolder name under `generated/` (defaults to `generated-app`) |

The response is a JSON summary:

```json
{
  "status": "ok",
  "outputDir": "todo-api",
  "filesWritten": 5,
  "files": ["package.json", "server.js", "routes/todos.js", "README.md", ".gitignore"]
}
```

The generated project lands on your host machine at `generated/todo-api/` (mounted into the
container at `/home/node/generated/`).

## Notes and limitations

- **Cost**: every call hits the Anthropic API and is billed on your account.
- **`max_tokens` is capped at 8000** in the workflow's `Call Claude` node — large apps may get
  cut off mid-file. Raise it in that node's JSON body if you need more headroom.
- **No validation loop**: the workflow doesn't install dependencies, run, or test the generated
  code. Review it before running.
- **Output is git-ignored by default** (`generated/*` in `.gitignore`) since it's disposable
  scaffolding. If you want to keep a specific generated app, move it out of `generated/` or
  `git add -f` it.

## Troubleshooting

- **"Claude did not return valid JSON"** (thrown by the **Parse Files** node) — the model added
  commentary or markdown fences around the JSON. Re-run, or tighten the prompt; the **Prepare
  Prompt** node's system prompt already asks for JSON-only output, but very long prompts can
  occasionally cause drift.
- **401/403 from Call Claude** — the `Anthropic API` credential isn't selected on the node, or
  the key is invalid/out of quota.
- **Files not appearing in `generated/`** — confirm `docker-compose.yml` has the
  `./generated:/home/node/generated` volume mount and that you ran `docker compose up -d`
  (or `--force-recreate`) after it was added.
