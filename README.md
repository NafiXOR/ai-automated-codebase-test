# ai-automated-codebase-test

Automated workflow setup powered by [n8n](https://n8n.io), self-hosted via Docker.

## Quick start

```bash
cp .env.example .env      # then edit N8N_BASIC_AUTH_USER / PASSWORD
docker compose up -d      # start n8n
```

Open [http://localhost:5678](http://localhost:5678), log in, and import
[`workflows/starter-workflow.json`](workflows/starter-workflow.json) (⋯ menu → Import from File).

For the full walkthrough, see:

- [docs/n8n-setup-guide.md](docs/n8n-setup-guide.md) — step-by-step instructions
- [docs/n8n-setup-guide.html](docs/n8n-setup-guide.html) — the same guide as an interactive page (open directly in a browser)

## Workflows included

| Workflow | What it does |
|---|---|
| [`starter-workflow.json`](workflows/starter-workflow.json) | Minimal example: Webhook → Set → Respond |
| [`app-generator-workflow.json`](workflows/app-generator-workflow.json) | POST a plain-language app description, get a generated backend/frontend scaffold written to `generated/` — see [docs/app-generator-workflow.md](docs/app-generator-workflow.md) |
| [`app-generator-with-tests-workflow.json`](workflows/app-generator-with-tests-workflow.json) | Same idea, but writes the code, runs `npm test` against it, and sends failures back to Claude to fix — retrying until tests pass or a retry limit is hit — see [docs/app-generator-self-testing-workflow.md](docs/app-generator-self-testing-workflow.md) |

Before reaching for the app generator workflow, read
[docs/claude-vs-workflow-comparison.md](docs/claude-vs-workflow-comparison.md)
([PDF](docs/claude-vs-workflow-comparison.pdf)) — it covers what you gain (unattended triggering)
and give up (iteration, repo context, testing) compared to just asking Claude directly.

## Project structure

```
.
├── docker-compose.yml               # n8n service definition
├── .env.example                     # template for local env vars (copy to .env)
├── workflows/
│   ├── starter-workflow.json        # generic starter workflow (Webhook → Set → Respond)
│   ├── app-generator-workflow.json  # AI app generator workflow (needs an Anthropic API key)
│   └── app-generator-with-tests-workflow.json  # app generator with a test-and-retry loop
├── generated/                       # output from the app generator workflows (git-ignored)
└── docs/
    ├── n8n-setup-guide.md           # setup instructions
    ├── n8n-setup-guide.html         # interactive setup guide
    ├── app-generator-workflow.md    # app generator setup + usage
    ├── app-generator-self-testing-workflow.md  # self-testing variant setup + usage
    └── claude-vs-workflow-comparison.md  # when to use the workflow vs. Claude directly
```

## Requirements

- Docker + Docker Compose
