# AI App Generator (Self-Testing) Workflow

A version of the [app generator workflow](app-generator-workflow.md) that closes the loop
described in [claude-vs-workflow-comparison.md](claude-vs-workflow-comparison.md#what-it-would-take-to-add-testing-to-the-workflow):
it writes the generated code to disk, actually runs `npm install && npm test` against it, and if
that fails, sends the failure output back to Claude and asks it to fix the code — repeating up to
a retry limit before giving up.

**File:** [`workflows/app-generator-with-tests-workflow.json`](../workflows/app-generator-with-tests-workflow.json)

## How it works

```
Webhook ──► Prepare Prompt ──► Call Claude ──► Build Script ──► Run Script ──► Evaluate Result ──► Tests Passed?
              ▲                                                                                      │        │
              │                                                                                    yes│      no
              │                                                                          Summarize ◄──┘        │
              │                                                                              │            Retries Left?
              │                                                                      Respond Success       yes │    │no
              │                                                                                              │    │
              └───────────────────────────────── Prepare Fix Data ◄───────────────────────────────────────────┘    │
                                                                                                                      ▼
                                                                                                                  Give Up ──► Respond Failure
```

- **Build Script** parses Claude's generated files, base64-encodes each file's content, and
  builds a single shell script that writes every file to `generated/<outputDir>/` and then runs
  `npm install && npm test` in that directory.
- **Run Script** (Execute Command node) actually runs it inside the n8n container.
- **Evaluate Result** checks the exit code. `0` = tests passed.
- On failure, **Prepare Fix Data** increments the retry counter and loops back to **Prepare
  Prompt**, which builds a new prompt containing the previous code and the test failure output,
  asking Claude to fix it.
- Once retries are exhausted, it gives up and responds with the last failure instead of looping
  forever.

## Setup

Same as the [non-testing version](app-generator-workflow.md#1-add-your-anthropic-api-key-as-a-credential):

1. Create (or reuse) the **Anthropic API** Header Auth credential (`x-api-key` = your key).
2. Import [`workflows/app-generator-with-tests-workflow.json`](../workflows/app-generator-with-tests-workflow.json).
3. Open the **Call Claude** node and select the Anthropic credential.
4. Save and toggle **Active**.

No extra volume mounts needed — it reuses the same `./generated:/home/node/generated` mount as
the other workflow.

## Usage

```bash
curl -X POST http://localhost:5678/webhook/generate-app-tested \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "A REST API for a todo list with in-memory storage: GET/POST /todos, PATCH and DELETE /todos/:id",
    "target": "backend",
    "outputDir": "todo-api",
    "maxRetries": 3
  }'
```

`maxRetries` is optional (defaults to `3`). Each retry is a full extra Claude API call, so the
worst case for a request is `maxRetries + 1` calls before giving up.

**Success response:**
```json
{
  "status": "ok",
  "outputDir": "todo-api",
  "retriesUsed": 1,
  "filesWritten": 5,
  "files": ["package.json", "server.js", "test/basic.test.js", "..."],
  "testOutput": "...npm test output from the passing run..."
}
```

**Failure response** (retries exhausted):
```json
{
  "status": "failed",
  "reason": "Max retries exceeded - tests still failing.",
  "outputDir": "todo-api",
  "retriesUsed": 3,
  "lastTestOutput": "...the last npm test failure..."
}
```

## What I verified vs. what's unverified

I tested the core write-and-test mechanism locally (outside n8n): given a mocked Claude
response, the **Build Script** node's generated shell script correctly base64-decodes each file
to disk, runs `npm install && npm test`, and produces exit code `0` on a passing test and
non-zero on a failing one, with no shell-injection risk from file contents (they're base64-piped,
never interpolated raw).

What I could **not** verify without a live n8n instance — check these first if something breaks:

- **The `Tests Passed?` / `Retries Left?` IF node conditions.** n8n's IF node condition schema
  has changed across versions; if a branch doesn't route the way you expect, open the node and
  re-set the condition by hand (Passed?: `{{$json.passed}}` is true. Retries Left?:
  `{{$json.retryCount}}` < `{{$json.maxRetries}}`).
- **Execute Command's output field names.** The `Evaluate Result` node expects `exitCode`,
  `stdout`, `stderr` on the item coming out of `Run Script`. If your n8n version names these
  differently, `Evaluate Result` will treat the run as failed (safe default) rather than crash —
  check its output in the execution view if every run "fails" immediately.
- **`npm` and `base64`/`dirname` inside the n8n container.** The official `n8nio/n8n` image is
  Node-based, so `npm` should be present, and its Alpine base includes BusyBox `base64` and
  `dirname`. If `Run Script` errors with "command not found," the container is missing one of
  these and you'd need a custom image.
- **The retry loop's cross-node data lookups** (`$('Prepare Prompt').itemMatching(0)` etc.),
  which recover state that gets discarded when `Call Claude` and `Run Script` replace the item's
  JSON with their own output. This is n8n's documented mechanism for exactly this case, but it's
  the most failure-prone part of any cyclic n8n workflow — if `outputDir`/`retryCount` show up as
  `undefined` partway through, this is where to look.

Use the execution view (click into a run, inspect each node's input/output) to debug any of
these — it'll show exactly which node produced unexpected data.

## Limitations

- **Cost compounds with retries** — a stubborn bug could cost up to `maxRetries + 1` API calls
  per request.
- **Still no human review** — a "passing" result only means the AI-written tests it also wrote
  pass. It doesn't guarantee the tests are meaningful, only that they're self-consistent with the
  generated code.
- **No protection against Claude writing bad-but-passing tests** (e.g. a test that doesn't
  actually assert anything) to make the loop exit early. Review generated code before relying on it.
