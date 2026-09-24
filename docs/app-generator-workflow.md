# AI App Generator Workflow

An n8n workflow that takes a plain-language description of an app and generates a runnable
backend or frontend scaffold using the Gemini API, writing the generated files straight to disk.

**How it works:** `Webhook → Prepare Prompt → Call Gemini → Parse Files → Write to Disk → Respond`

> **Scope check:** this is a one-shot generator, not an autonomous coding agent. The model writes
> the files in a single pass with no chance to test or fix its own output — great for scaffolds,
> boilerplate, and quick prototypes; not a replacement for actually building and iterating on a
> real project.

## Prerequisites

- n8n running via this repo's `docker-compose.yml` (see [n8n-setup-guide.md](n8n-setup-guide.md))
- A [Gemini API key](https://aistudio.google.com/apikey) — this workflow calls the Gemini API
  directly (model `gemini-3.6-flash`), so usage counts against your Google AI quota/billing

## 1. Add your Gemini API key as a credential

The workflow authenticates with an n8n credential rather than a hardcoded key, so the key is
stored encrypted inside n8n and never appears in the workflow JSON or this repo.

1. In n8n, go to **Credentials** → **Add credential** → **Header Auth**.
2. Set:
   - **Name**: `Gemini API`
   - **Header Name**: `Authorization`
   - **Header Value**: `Bearer <your Gemini API key>` (the word `Bearer`, a space, then the key)
3. Save.

The **Call Gemini** node uses Gemini's OpenAI-compatible endpoint
(`/v1beta/openai/chat/completions`). To try another model, change `model` in that node's JSON
body — `gemini-2.5-flash` is no longer offered to new keys, and `gemini-flash-latest` /
`gemini-3.8-flash` often answered `503 high demand` when this was set up.

### Optional: protect the webhook with a token

The generator webhooks accept any request that reaches them. n8n is bound to `127.0.0.1`, so
that means only programs on this machine — fine for testing. Before exposing n8n further, or
if you enable the deploy override, require a secret header:

1. Add a second **Header Auth** credential: **Name** `Webhook Token`, **Header Name**
   `X-Webhook-Token`, **Header Value** a long random string (e.g. `openssl rand -hex 32`).
2. In each generator workflow, open the **Webhook** node, set **Authentication** to
   **Header Auth**, and select `Webhook Token`. Save.
3. Add `-H "X-Webhook-Token: <your token>"` to your requests. Requests without it get a 403.

## 2. Import and configure the workflow

1. **Workflows** → **Add workflow** → **⋯** → **Import from File** →
   [`workflows/app-generator-workflow.json`](../workflows/app-generator-workflow.json).
2. Open the **Call Gemini** node and check that **Credential for Header Auth** is set to
   `Gemini API`. It's pre-selected when the credential exists in your n8n; otherwise pick it.
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
| `target` | no | `"backend"` (default, Node/Express), `"frontend"` (React SPA), or `"static"` (plain HTML/CSS/JS site) |
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

### A plain HTML/CSS website

Use `"target": "static"` for a site that needs no npm, server, or build step:

```bash
curl -X POST http://localhost:5678/webhook/generate-app \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "A one-page portfolio for a web developer: hero, about, 3 project cards, contact section. Responsive, with a dark/light toggle.",
    "target": "static",
    "outputDir": "my-site"
  }'
```

Then open it straight from disk: `open generated/my-site/index.html` (macOS), or double-click
the file. The system prompt requires the site to work from `file://` with relative paths and no
external requests.

The generated project lands on your host machine at `generated/todo-api/` (mounted into the
container at `/home/node/generated/`).

## Notes and limitations

- **Cost**: every call hits the Gemini API and counts against your quota/billing.
- **`max_tokens` is capped at 16000**, and `reasoning_effort` is `"low"`. Gemini counts its hidden
  "thinking" toward `max_tokens`; at the default effort, a one-page portfolio site ran out of
  tokens mid-JSON, and at `"low"` the same request used about 6,800 in the workflow's `Call Gemini` node — large apps may get
  cut off mid-file. Raise it in that node's JSON body if you need more headroom.
- **No validation loop**: the workflow doesn't install dependencies, run, or test the generated
  code. Review it before running.
- **Output is git-ignored by default** (`generated/*` in `.gitignore`) since it's disposable
  scaffolding. If you want to keep a specific generated app, move it out of `generated/` or
  `git add -f` it.

## Troubleshooting

- **"The model did not return valid JSON"** (thrown by the **Parse Files** node) — the model added
  commentary around the JSON (markdown fences on their own are stripped automatically). If the
  error says it hit `max_tokens`, the app was too big for one response. Re-run, or tighten the prompt; the **Prepare
  Prompt** node's system prompt already asks for JSON-only output, but very long prompts can
  occasionally cause drift.
- **401/403 from the webhook** (before anything runs) — only if you turned on the optional
  webhook token: the `X-Webhook-Token` header is missing or wrong.
- **"Unsafe or invalid file path"** — the model returned an absolute path or one containing `..`.
  The workflow refuses to write outside `generated/<outputDir>/`; re-run.
- **401/403 from Call Gemini** — the `Gemini API` credential isn't selected on the node, the
  header value is missing the `Bearer ` prefix, or the key is invalid.
- **503 "high demand" or 429 from Call Gemini** — Google-side capacity or your rate limit. The
  node already retries 3 times, 5 seconds apart; if it still fails, wait and re-run, or switch
  `model` in the node's JSON body.
- **Files not appearing in `generated/`** — confirm `docker-compose.yml` has the
  `./generated:/home/node/generated` volume mount and that you ran `docker compose up -d`
  (or `--force-recreate`) after it was added.
