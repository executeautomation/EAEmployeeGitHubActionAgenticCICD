---
description: |
  Runs Playwright CLI smoke checks against EAEmployee when a new
  [smoke-test]-prefixed issue is opened.

on:
  issues:
    types: [opened]

if: github.event_name == 'issues' && startsWith(github.event.issue.title, '[smoke-test]')

permissions:
  contents: read

network:
  allowed: [defaults, playwright]

safe-outputs:
  add-comment:
    max: 1
    target: triggering
    required-title-prefix: "[smoke-test]"

tools:
  playwright:
    browsers: [chromium]
  bash: ["*"]

timeout-minutes: 30

steps:
  - name: Set up .NET 8
    uses: actions/setup-dotnet@v6
    with:
      dotnet-version: 8.0.x
  - name: Publish self-contained smoke-test application
    run: dotnet publish EAEmployee.Net8.csproj --configuration Release --runtime linux-x64 --self-contained true --output .playwright-data/publish

engine:
  id: codex
  env:
    OPENAI_BASE_URL: https://api.minimax.io/v1
    CODEX_API_KEY: ${{ secrets.MINIMAX_API_KEY }}
    OPENAI_API_KEY: ${{ secrets.MINIMAX_API_KEY }}
  config: |
    model_reasoning_effort = "high"
model: MiniMax-M3.1-Flash

excluded-env: [MINIMAX_API_KEY]
sandbox:
  agent: {model-fallback: false, token-steering: false}
models:
  providers:
    openai:
      models:
        MiniMax-M3.1-Flash:
          cost: {input: "0.00000015", output: "0.00000060", cache_read: "0.000000015"}
---

# EAEmployee Playwright CLI Smoke Tests

Run the smoke checks on the checked-out commit. Do not edit application or test
files as part of this workflow. Only execute this task for an issue whose title
starts with `[smoke-test]`. Report each check's result and, on failure, the
failed action and relevant application or CLI output. When the run completes,
always use the `add_comment` safe output to post one concise result comment on
the triggering issue. Include an overall PASS or FAIL, the result of the home,
login, employee CRUD, and dashboard checks, and relevant failure details. Do
not post the result through `gh`, the GitHub API, or any other direct write.

Use the Playwright CLI (`playwright-cli`) for all browser interaction. Do not run
`npm test`, `playwright test`, or use the Playwright Test API. Start the
application first and perform every browser check against that running instance.

The workflow publishes the app as a self-contained Linux executable before the
agent starts. The built-in gh-aw Playwright tool provisions `playwright-cli`,
its skill, and Chromium; do not install them manually.

```bash
set -euo pipefail

mkdir -p /tmp/gh-aw/agent
mkdir -p .playwright-data
rm -f .playwright-data/e2e.sqlite .playwright-data/e2e.sqlite-shm .playwright-data/e2e.sqlite-wal
ASPNETCORE_URLS=http://127.0.0.1:5114 \
ASPNETCORE_ENVIRONMENT=Development \
ConnectionStrings__DefaultConnection="DataSource=$PWD/.playwright-data/e2e.sqlite;Cache=Shared" \
ApiSettings__ApiKey=local-e2e-key \
./.playwright-data/publish/EAEmployee.Net8 > /tmp/gh-aw/agent/eaemployee-smoke-app.log 2>&1 &
server_pid=$!
echo "$server_pid" > /tmp/gh-aw/agent/eaemployee-smoke-app.pid

if ! curl --fail --silent --show-error --retry 10 --retry-connrefused \
  --retry-all-errors --retry-max-time 30 http://127.0.0.1:5114/ >/dev/null; then
  tail -n 100 /tmp/gh-aw/agent/eaemployee-smoke-app.log
  kill "$server_pid"
  exit 1
fi
```

After the health check succeeds, use the built-in Playwright CLI skill and one
browser session for all checks:

1. Run `playwright-cli open --browser=chromium http://127.0.0.1:5114/`, take a
   snapshot, and verify the home page
   renders the Employee Management content.
2. Run `playwright-cli goto http://127.0.0.1:5114/Account/Login`, take a fresh
   snapshot, use its element references to sign in as the seeded administrator
   (`admin` / `password`), and verify the signed-in admin greeting.
3. Use the UI to create an employee with a unique email and verify the employee
   appears in the directory. Run `playwright-cli goto
   http://127.0.0.1:5114/Home/Dashboard` while this employee exists, then verify
   the Workforce Dashboard heading and KPI cards render.
4. Return to the directory, edit the smoke-test employee's name and email, and
   verify the updated values. Delete that employee and verify it is absent from
   the list. Complete all required fields with valid smoke-test values.

Use only current CLI snapshot references for element actions. Inspect the CLI
output after each action and treat missing expected content, navigation errors,
or failed submissions as failures. Capture CLI output and the application log
when a check fails. Close the CLI browser and stop the process recorded in
`/tmp/gh-aw/agent/eaemployee-smoke-app.pid` when finished, whether the checks
pass or fail.
Close the browser with `playwright-cli close`, then read the PID with
`cat /tmp/gh-aw/agent/eaemployee-smoke-app.pid` and stop it with `kill`.
