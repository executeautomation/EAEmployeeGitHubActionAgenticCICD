# EAEmployee Playwright framework

This end-to-end framework implements the browser and API priorities from `TestRequirement.md` with Page Objects, a deterministic 12-employee fixture, isolated SQLite state, and multi-format reports.

## Run

```bash
npm install
npm test
```

The default run starts the application at `http://127.0.0.1:5114`, recreates `.playwright-data/e2e.sqlite`, uses the installed Google Chrome channel, executes setup plus the complete suite with one worker, and writes reports.

In GitHub Actions, `.github/workflows/e2e.yml` builds the application in Release mode, installs Playwright's managed Chromium, and runs the same suite. The application starts alongside the tests, applies its Entity Framework migrations, and seeds the isolated SQLite database before Playwright proceeds.

To test an already-running deployment:

```bash
PLAYWRIGHT_BASE_URL=https://example.test npm test
```

Useful commands:

```bash
npm run test:headed
npm run test:debug
npm run report
```

## Structure

- `pages/`: reusable Page Object Models.
- `fixtures/`: deterministic requirement-derived test data.
- `tests/`: traceable tests named with `TR-*` requirement IDs.
- `scripts/start-app.mjs`: isolated local application/database lifecycle.
- `scripts/run-tests.mjs`: complete execution plus Markdown report generation.

## Reports and evidence

- `playwright-report/index.html`: interactive HTML report.
- `test-report/EXECUTION_REPORT.md`: concise human-readable execution report.
- `test-report/results.xml`: JUnit report for CI.
- `test-report/results.json`: machine-readable result data.
- `test-results/artifacts/`: screenshots, videos, and traces retained for failures.

The suite intentionally reports unmet product requirements as failures. It does not weaken assertions to make known application gaps appear green.
