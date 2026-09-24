# AI App Generator (Self-Testing) Workflow

A version of the [app generator workflow](app-generator-workflow.md) that closes the loop
described in [claude-vs-workflow-comparison.md](claude-vs-workflow-comparison.md#what-it-would-take-to-add-testing-to-the-workflow):
it writes the generated code to disk, actually runs `npm install && npm test` against it, and if
that fails, sends the failure output back to Gemini and asks it to fix the code — repeating up to
a retry limit before giving up.

**File:** [`workflows/app-generator-with-tests-workflow.json`](../workflows/app-generator-with-tests-workflow.json)

## How it works

```
Webhook ──► Prepare Prompt ──► Call Gemini ──► Build Script ──► Run Script ──► Evaluate Result ──► Tests Passed?
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

- **Build Script** parses Gemini's generated files, base64-encodes each file's content, and
  builds a single shell script that writes every file to `generated/<outputDir>/` and then runs
  `npm install && npm test` in that directory. If the response can't be used (not valid JSON,
  cut off at `max_tokens`, or an unsafe file path), the script just prints that problem and
  fails, so it goes through the same fix loop as a failing test instead of stopping the run.
- **Run Script** (Execute Command node) actually runs it inside the n8n container. The script
  always exits `0` and prints its real exit code as a final `__EXIT_CODE__=<n>` line. On a
  non-zero exit, Execute Command drops stdout, which is where npm writes test output.
- **Evaluate Result** reads that marker. `0` = tests passed.
- On failure, **Prepare Fix Data** increments the retry counter and loops back to **Prepare
  Prompt**, which builds a new prompt containing the original request, the previous response,
  and the failure output, asking Gemini to fix it.
- Once retries are exhausted, it gives up and responds with the last failure instead of looping
  forever.

## Setup

Same as the [non-testing version](app-generator-workflow.md#1-add-your-gemini-api-key-as-a-credential):

1. Create (or reuse) the **Gemini API** Header Auth credential (`Authorization` = `Bearer <key>`).
2. Import [`workflows/app-generator-with-tests-workflow.json`](../workflows/app-generator-with-tests-workflow.json).
3. Open the **Call Gemini** node and check the `Gemini API` credential is selected. (Optionally protect the
   webhook with a token — see
   [app-generator-workflow.md](app-generator-workflow.md#optional-protect-the-webhook-with-a-token).)
4. Save and toggle **Active**.

On n8n 2.x the Execute Command node is disabled by default. This repo's `docker-compose.yml`
turns it back on with `NODES_EXCLUDE`; if you run n8n some other way and the node shows as
unknown, set the same variable.

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

`target` is `"backend"` (default), `"frontend"`, or `"static"`. For `"static"` (plain
HTML/CSS/JS that opens straight from `generated/<outputDir>/index.html`), the "test" is a
dependency-free Node script Gemini writes alongside the site. It checks every page exists, every
local `href`/`src` resolves, and each page has the requested sections.

`maxRetries` is optional (defaults to `3`). Each retry is a full extra Gemini API call, so the
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

I ran the workflow's actual Code nodes (`Prepare Prompt`, `Build Script`, `Evaluate Result`,
`Prepare Fix Data`) in Node with mocked model responses (plus one real `gemini-3.6-flash` response, whose generated project installed and passed its tests), and ran the shell scripts they produce
in a `node:20-alpine` container (BusyBox, like the n8n image). Checked cases: a response
wrapped in markdown fences with passing tests, failing tests (the failure text reaches the fix
prompt), invalid JSON, a response cut off at `max_tokens`, and a `../` path. Each one ended
with the expected pass/fail result and a fix prompt that includes the original request. File
contents are base64-piped and never pasted raw into the shell, so they can't inject commands.

What I could **not** verify without a live n8n instance — check these first if something breaks:

- **The `Tests Passed?` / `Retries Left?` IF node conditions.** n8n's IF node condition schema
  has changed across versions; if a branch doesn't route the way you expect, open the node and
  re-set the condition by hand (Passed?: `{{$json.passed}}` is true. Retries Left?:
  `{{$json.retryCount}}` < `{{$json.maxRetries}}`).
- **Execute Command's output field names.** `Evaluate Result` looks for the `__EXIT_CODE__`
  marker in `stdout` (falling back to `stderr`/`error`). If your n8n version names the output
  field differently, the marker isn't found and the run counts as failed (safe default) rather
  than crashing — check `Run Script`'s output in the execution view if every run "fails".
- **`npm` and `base64`/`dirname` inside the n8n container.** The official `n8nio/n8n` image is
  Node-based, so `npm` should be present, and its Alpine base includes BusyBox `base64` and
  `dirname`. If `Run Script` errors with "command not found," the container is missing one of
  these and you'd need a custom image.
- **The retry loop's cross-node data lookups** (`$('Prepare Prompt').itemMatching(0)` etc.),
  which recover state that gets discarded when `Call Gemini` and `Run Script` replace the item's
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
- **No protection against the model writing bad-but-passing tests** (e.g. a test that doesn't
  actually assert anything) to make the loop exit early. Review generated code before relying on it.
