---
name: testrigor-cli
description: Run testRigor test suites from the command line with the `testrigor` CLI — authenticate, set a default suite, run suites (sync or async), filter by labels, override URL, tunnel to localhost, upload mobile app builds, push file-based test cases/rules/variables, and save JUnit reports for CI/CD. Use when asked to run, trigger, or schedule testRigor tests from a terminal or pipeline, set up testRigor CI, or manage testRigor auth/config. To author the tests themselves, use `testrigor-write-tests`.
metadata:
  {
    "openclaw":
      {
        "emoji": "🧪",
        "requires": { "bins": ["testrigor"] },
        "install":
          [
            {
              "id": "node",
              "kind": "node",
              "package": "testrigor-cli",
              "bins": ["testrigor"],
              "label": "Install testRigor CLI (npm)",
            },
          ],
      },
  }
---

# testRigor CLI

`testrigor` triggers testRigor test-suite runs from a terminal or CI pipeline.
The suite lives in testRigor's cloud; a run can **push local test files into it**
(updating the suite) and pull back a JUnit report.

This skill is the flag-level reference. To **write** test cases use
`testrigor-write-tests`; for the full **build → run → debug → update loop** (the
recommended way to develop tests as code) follow `testrigor-dev-loop`.

## Install & verify

Requires Node.js ≥ 18.

```bash
npm install -g testrigor-cli
testrigor --version          # e.g. testrigor-cli/0.0.41-beta
testrigor --help
```

## Authentication

Two token types exist:

- **Personal Authentication Token (PAT)** — your personal token. Get it in the app:
  *username (top-right) → API Tokens → Generate New Token*.
- **Suite CI/CD token** — tied to one suite. Get it from the suite's *CI/CD
  Integration* page.

The CLI resolves a token in this exact priority order (first hit wins):

1. **`TESTRIGOR_API_KEY` env var** ← use this for Claude/CI; fully non-interactive
2. `--auth-token <PAT>` flag
3. `-t, --token <SUITE_TOKEN>` flag
4. `~/.testrigor/testrigor.yml` (written by `testrigor authenticate`)

> ⚠️ `testrigor authenticate` opens an **interactive hidden prompt** — it will hang
> in an agent/CI context. For automation, set the environment variable instead:

```bash
export TESTRIGOR_API_KEY="<YOUR_PAT>"      # non-interactive; highest priority
```

Suite ID resolves from the positional arg, else the stored default. Set a default
(non-interactive) so you can omit it afterward — the ID is in the suite's URL:

```bash
testrigor test-suite config --default <TEST_SUITE_ID>
testrigor test-suite config --show
testrigor test-suite config --delete
```

`TESTRIGOR_CLI_HOME` overrides the config/cache dir (default `~/.testrigor`).

## Running a suite

```bash
# Uses default suite ID + stored PAT
testrigor test-suite run

# Explicit suite + suite CI token
testrigor test-suite run <TEST_SUITE_ID> --token <SUITE_CI_TOKEN>

# Explicit suite + personal token
testrigor test-suite run <TEST_SUITE_ID> --auth-token <PAT>
```

Exit status reflects the run: a **non-zero exit means the run failed** (sync mode),
so the command works directly as a CI gate. Each run also **auto-cancels the previous
run** of the same suite.

## Updating the suite by running (mutations)

There is no separate "deploy" step. The file-push flags below are parsed locally and
sent with the run request, so **running updates the remote suite to match your files**:

| Flag | Sent to the API as | Effect |
|------|--------------------|--------|
| `--test-cases-path "<glob>"` | `baselineMutations[]` | create/update test cases |
| `--rules-path "<glob>"` | `ruleMutations[]` | create/update reusable rules |
| `--variables-path file.json` | `storedValues` | set variables for the run |
| `--settings-path file.{yaml,json}` | `settings` | apply suite settings |
| `--explicit-mutations` | `explicitMutations: true` | treat your files as the authoritative set |

**Identity** — how a file maps to a remote object:

- A test case is matched by its **filename** (→ `description`/name), or by a
  `testCaseUuid` field inside a `.yaml` file (pins to one existing test case).
- A rule is matched by its **filename** (→ rule name).

So: edit a file, re-run → that same-named test/rule is updated remotely. Rename a file
→ a new object is created and the old one is orphaned (unless you used `testCaseUuid`).

Two behaviors confirmed against a live suite:

- **A plain push is additive, and the run executes the *whole* suite.** Without
  `--explicit-mutations`, your files are created/updated by name and any pre-existing
  test cases remain and **also run** — all against the single `--url` you pass, so
  tests authored for a *different* app/URL will fail (and failing steps wait for
  timeouts, making the run much slower). To run only your subset, label your tests and
  pass `--labels`.
- **Additive push can't delete.** To remove a remote test, either delete it in the UI,
  or manage the entire suite as files and run with `--explicit-mutations` (your local
  set becomes authoritative — anything not present is dropped, *including tests you
  didn't author*, so use it deliberately).

For the on-disk file schema, see the `testrigor-write-tests` skill; for the whole
build → run → debug → update loop, see `testrigor-dev-loop`.

## Command tree

```
testrigor authenticate                 # INTERACTIVE prompt — avoid in automation (use TESTRIGOR_API_KEY)
testrigor test-suite config             # --default <id> | --show | --delete
testrigor test-suite run [ID] [flags]   # push files + trigger a run
testrigor plugins                       # list installed plugins
testrigor help [command]
```

## `test-suite run` flags

Authentication / connection:

| Flag | Purpose |
|------|---------|
| `--auth-token <value>` | Personal authentication token (PAT) |
| `-t, --token <value>` | Suite CI/CD authentication token |
| `--base-url-web <value>` | Web base URL (e.g. `https://app.testrigor.com`) — self-hosted/region |
| `--base-url-api <value>` | API base URL (e.g. `https://api2.testrigor.com`) |

Run control:

| Flag | Purpose |
|------|---------|
| `-w, --async` | Start the run and exit immediately (don't wait for results) |
| `--verbose` | Detailed output for debugging |
| `--url <value>` | Override the suite's target URL for this run |
| `--labels <a,b,...>` | Only run test cases with these labels (comma-separated) |
| `--excluded-labels <a,b,...>` | Skip test cases with these labels |
| `--test-case-uuid <uuid>` | Run a single test case (requires `--url`) |

Local tunneling (test a site not reachable from the internet):

| Flag | Purpose |
|------|---------|
| `--localhost` | Tunnel to a local app (requires `--url`, e.g. `http://localhost:3000`) |
| `--min-port <n>` | Min tunnel port (default 40000) |
| `--max-port <n>` | Max tunnel port (default 50000) |

Mobile / desktop app builds:

| Flag | Purpose |
|------|---------|
| `--file-path <file>` | Upload a local build — accepted: `.zip .exe .msi .apk .aab .ipa`; ignored if `--url` is set |

Git context (annotate the run with branch/commit):

| Flag | Purpose |
|------|---------|
| `--branch <name>` | Branch name (requires `--commit`) |
| `--commit <hash>` | Commit hash (requires `--branch`) |

Pushing local test definitions (store tests in your own repo):

| Flag | Purpose |
|------|---------|
| `--test-cases-path <glob>` | Test-case files, e.g. `"test-cases/**/*.{txt,yaml,yml}"` |
| `--rules-path <glob>` | Reusable-rule files, e.g. `"rules/**/*.yaml"` |
| `--variables-path <file>` | Variables JSON for data-driven runs |
| `--settings-path <file>` | Suite settings (`settings.yaml` / `settings.json`) |
| `--explicit-mutations` | Treat the pushed test-case set as authoritative (sync/replace; use with `--test-cases-path`) |
| `--auto-create-ai-rules` | Let AI auto-generate rules for unknown steps |

Reporting:

| Flag | Purpose |
|------|---------|
| `--junit-report-save-path <file>` | Write a JUnit XML report (sync mode only) |

## Localhost debugging (the tunnel)

`--localhost --url http://localhost:3000` lets testRigor's cloud browsers reach an app
running on your machine. Mechanics (so you know what to expect):

- On first use the CLI downloads a small tunnel binary (`trtc`) into
  `~/.testrigor/trtc/` — needs outbound network the first time.
- It opens a tunnel on a port in `--min-port`..`--max-port` (40000–50000) and routes
  the run's traffic through it; the tunnel is torn down when the run ends.
- `--url` must be the locally reachable address (`http://localhost:PORT`,
  `http://127.0.0.1:PORT`). `--localhost` without `--url` is rejected.
- Add `--verbose` to see tunnel steps (download, start, port).

```bash
# start your local app first, then:
testrigor test-suite run "$SUITE" --localhost --url "http://localhost:3000" \
  --test-cases-path "test-cases/**/*.{txt,yaml,yml}" \
  --rules-path "rules/**/*.{txt,yaml,yml}" \
  --junit-report-save-path ./.run/report.xml --verbose
```

## Reading results / debugging a run

In **sync mode** (default — do *not* pass `--async` while debugging) the CLI waits for
completion and surfaces:

- **Exit code** — `0` = all passed; non-zero = a test failed/was canceled. Primary gate.
- **Run URL** printed in the output (`https://app.testrigor.com/.../runs/<id>`) — open
  in a browser for step-level screenshots, video, and the exact failing step. (It's
  authenticated, so a human opens it; the CLI can't fetch its contents.)
- **JUnit XML** (`--junit-report-save-path`, sync only — produced **even when tests
  fail**) — the machine-readable signal. testRigor emits standard JUnit: a failing
  `<testcase status="failure">` carries a `<failure message="..." type="failure"/>`
  whose `message` names the exact failing command, and `<testsuite failures="N">` holds
  the count. Each `<testcase>` also includes its `testCaseUuid` — **except in
  `--labels`-filtered runs**, where (verified) the `<testcase>` name/uuid come back
  empty (`name="Test Case UUID="`); there, identify failures by the `<failure>`
  `message` and the run URL, not the testcase name. Parse it:
  ```bash
  mkdir -p ./.run                                                  # parent dir must exist
  xmllint --xpath 'string(//testsuite/@failures)' ./.run/report.xml   # 0 = all passed
  grep -A1 '<failure' ./.run/report.xml                           # failing cases + reason
  xmllint --xpath '//testcase[failure]/@name' ./.run/report.xml   # list failing test names
  ```
- **Iterate on one test** without re-running the suite: grab the failing test's
  `testCaseUuid` from the report, then `--test-case-uuid <uuid> --url <url>`.
- **Cleaner logs**: set `CI=true` to force the line-by-line (verbose) renderer instead
  of the interactive spinner — useful when capturing output programmatically.

Internally the CLI polls run status every 10s; `200`=passed, `230`=failed,
`229`=canceled, `228/227`=running — these map onto the exit code above.

**Expect minutes, not seconds.** Cloud browsers spin up per run, richer pages take
longer, and **failing tests are the slowest** because testRigor auto-waits/retries
before giving up (a few failing tests can push a run past 5 minutes). When calling the
CLI from an agent or CI step, set a generous timeout (≈10 min) and stay in sync mode so
the exit code gates the job.

## Recipes

Smoke run gated in CI, with JUnit output:

```bash
testrigor test-suite run "$TEST_SUITE_ID" \
  --token "$TESTRIGOR_SUITE_TOKEN" \
  --labels smoke,critical \
  --junit-report-save-path ./testrigor-report.xml
```

Fire-and-forget (async) — kick off and let the pipeline continue:

```bash
testrigor test-suite run --async --token "$TESTRIGOR_SUITE_TOKEN"
```

Test a local dev server via tunnel:

```bash
testrigor test-suite run --localhost --url http://localhost:3000
```

Run a PR build and tag it with git context:

```bash
testrigor test-suite run \
  --branch "$(git rev-parse --abbrev-ref HEAD)" \
  --commit "$(git rev-parse HEAD)" \
  --url "$PREVIEW_URL"
```

Push repo-managed tests, replacing the suite's current set:

```bash
testrigor test-suite run \
  --test-cases-path "test-cases/**/*.txt" \
  --rules-path "rules/**/*.yaml" \
  --variables-path variables.json \
  --settings-path settings.yaml \
  --explicit-mutations
```

Upload and test a mobile build:

```bash
testrigor test-suite run --file-path ./build/app-release.apk
```

## CI/CD notes

- Provide tokens via environment secrets (`$TESTRIGOR_SUITE_TOKEN`), never commit them.
- Use **sync mode** (omit `--async`) when you want the job to pass/fail on results.
- Combine `--junit-report-save-path` with your CI's JUnit ingestion to surface
  per-test results.
- `--labels` / `--excluded-labels` let one suite power several pipelines
  (smoke on every push, full regression nightly).

## Troubleshooting

- `testrigor <command> --help` prints exact flags for the installed version.
- Auth errors → confirm `TESTRIGOR_API_KEY` is exported (or `--auth-token`/`--token`
  is passed) and the token hasn't expired in *API Tokens*; for suite tokens confirm
  the ID matches the suite URL. Don't rely on `testrigor authenticate` in automation —
  it prompts interactively.
- Globs match nothing → quote them (`"test-cases/**/*.txt"`) so the CLI expands them,
  not the shell.
- Self-hosted / non-US region → set `--base-url-web` and `--base-url-api` together.
- `--junit-report-save-path` produces nothing in `--async` mode (no results yet), and
  won't create missing parent directories — `mkdir -p` first.
