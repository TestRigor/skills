---
name: testrigor-dev-loop
description: End-to-end playbook for developing testRigor tests as code with the CLI — build test cases and reusable rules as local files, run them against a local app via the testRigor tunnel, debug from the JUnit report and exit code, and update the remote suite simply by re-running (file mutations). Use this when the goal is to author, run, debug, and iterate on testRigor tests from a repo/terminal (not the web UI). Pulls in `testrigor-write-tests` for the language, `testrigor-cli` for flag details, and `testrigor-api-testing` for API steps.
---

# testRigor dev loop (tests as code)

A repeatable loop for building testRigor tests **from a repo**, driving them with
the `testrigor` CLI, and iterating until green. This is the orchestration layer —
it sequences the other three skills:

- **`testrigor-write-tests`** — the plain-English language (what to put *in* a step).
- **`testrigor-cli`** — every CLI flag and mechanic in detail.
- **`testrigor-api-testing`** — `call api` / `mock api` steps.

## The one mental model that matters

There is **no separate "deploy" step.** When you run with `--test-cases-path` /
`--rules-path`, the CLI parses those local files and sends them to testRigor as part
of triggering the run. So **running = push your files into the remote suite + execute
them.** Your local files are the source of truth; the remote suite is updated to match
on every run.

Identity (how testRigor knows which remote test/rule a file maps to):

| Local file | Becomes | Identified by |
|------------|---------|---------------|
| `test-cases/login.txt` | a test case named `login` | the **filename** (`description`), or `testCaseUuid` inside a `.yaml` |
| `rules/add-to-cart.txt` | a reusable rule named `add-to-cart` | the **filename** (`name`) |

Edit a file → re-run → that same-named test/rule is updated remotely. Add a new
filename → a new test/rule appears. Add `--explicit-mutations` to declare your repo
the authoritative set for the run. (Each new run also auto-cancels the previous one.)

**Use one suite per app / environment.** A run applies a single `--url` to *every*
test in the suite, and `--explicit-mutations` syncs the suite to exactly your files.
So give each app under test its own dedicated suite — don't mix tests for different
sites in one suite, or no single run can satisfy them all and `--explicit-mutations`
would clobber the unrelated ones. (Create the suite once in the UI; point the CLI at
it via `--default`.)

## 0. One-time setup (non-interactive — required for Claude)

`testrigor authenticate` opens an **interactive hidden prompt**, so don't use it in
an agent/CI loop. Authenticate via environment instead (highest-priority token):

```bash
export TESTRIGOR_API_KEY="<YOUR_PERSONAL_AUTH_TOKEN>"   # app → username → API Tokens
export SUITE="<TEST_SUITE_ID>"                          # from the suite's URL
testrigor test-suite config --default "$SUITE"          # optional: lets you omit the ID
testrigor --version                                     # sanity check
```

Get the **Personal Authentication Token** in the app: *username (top-right) → API
Tokens → Generate New Token*. Get the **suite ID** from the suite's URL. Never commit
the token — keep it in the environment.

## 1. Build — write tests and rules as files

Lay the repo out like this (paths are yours; keep filenames stable — they are the
test/rule names):

```
testrigor/
├── test-cases/
│   ├── login.txt
│   └── checkout/place-order.txt
├── rules/
│   ├── login.txt              # reusable rule, invoked by writing `login` in a test
│   └── add-to-cart.txt
├── variables.json             # { "username": "...", "plan": "Pro" }  → stored values
└── settings.yaml              # optional suite settings
```

- A `.txt` file's entire content is the steps; the filename (sans extension) is the
  name. Use `//` for comments.
- A test invokes a reusable rule by writing the rule's **name** (the rule filename) as
  a step — e.g. a step `login` runs `rules/login.txt`.
- Write the steps using `testrigor-write-tests`. For the exact on-disk schema
  (`.txt` vs `.yaml`, labels, datasets, `testCaseUuid`), see that skill's
  "Test cases & reusable rules as files" section.

Example `rules/login.txt`:

```
// the suite's start URL opens automatically — navigate by clicking, never `open url`
click "Sign in"
enter stored value "username" into "Email"
enter stored value "password" into "Password"
click "Log in"
check that page contains "My Account"
```

Example `test-cases/checkout/place-order.txt`:

```
// reuse the login rule, then exercise checkout
login
click "Wireless Mouse"
add-to-cart                       // reuses rules/add-to-cart.txt
click "Checkout"
fill out the rest of the required fields in form
click "Place Order"
check that page contains "Thank you for your order"
```

## 2. Run against localhost (with the tunnel)

Start your local app first, then run. `--localhost` spins up the testRigor tunnel so
the cloud browsers can reach your machine; `--url` points at the local app.

Bring the app up however you normally do (`npm run dev`, etc.). For a static site:

```bash
python3 -m http.server 8421 --directory ./public   # serves ./public at http://localhost:8421/
```

Serve the **same web root the deployed site uses**, so a page lives at the same path
locally and in prod (e.g. serve `public/` so `/shop/index.html` resolves identically).
The tunnel only reaches *your* local app — keep the suite pointed at that one app;
tests that navigate to unrelated external URLs through the tunnel are unreliable.

```bash
mkdir -p testrigor/.run
testrigor test-suite run "$SUITE" \
  --localhost --url "http://localhost:3000" \
  --test-cases-path "testrigor/test-cases/**/*.{txt,yaml,yml}" \
  --rules-path "testrigor/rules/**/*.{txt,yaml,yml}" \
  --variables-path testrigor/variables.json \
  --explicit-mutations \
  --junit-report-save-path testrigor/.run/report.xml
```

This one command pushes the files (updating the suite) **and** runs them against your
local app. First localhost run downloads a small tunnel binary to `~/.testrigor/trtc/`
(needs network). Quote the globs so the CLI expands them, not the shell.

Iterate fast on a single failing test (skip the full suite):

```bash
testrigor test-suite run "$SUITE" --localhost \
  --url "http://localhost:3000" --test-case-uuid "<TEST_CASE_UUID>"
```

## 3. Debug — read the signals

The CLI runs synchronously (don't pass `--async` while debugging) and gives you:

1. **Exit code** — `0` = every test passed; non-zero = something failed/canceled.
   This is your primary pass/fail gate.
2. **The JUnit report** (`testrigor/.run/report.xml`) — machine-readable per-test
   results. Parse it to find which `<testcase>` has a `<failure>` and read the message:
   ```bash
   grep -A2 '<failure' testrigor/.run/report.xml      # quick look
   # or parse properly: xmllint --xpath '//testcase[failure]/@name' report.xml
   ```
3. **The run URL** printed in the output (`https://app.testrigor.com/.../runs/<id>`) —
   open it in a browser for step-level screenshots, video, and the exact failing step.
   Deep visual debugging lives here (the URL is authenticated, so a human opens it).
4. Add `--verbose` for tunnel diagnostics; set `CI=true` to force clean, line-by-line
   log output instead of the spinner renderer.

Then fix the offending `.txt`/`.yaml` file and go back to step 2. Because re-running
re-pushes the files, your fix updates the remote test automatically.

> **Cheap pre-check (runs take minutes).** After changing the app or a test, `grep` the
> page for the exact text and controls your tests assert before burning a full run —
> `grep -F "Thank you for your purchase!" end.html`. Catching a renamed label or a
> removed button statically is far faster than waiting on the cloud run to discover it.

## 4. "Update remote" is just re-running

There is nothing else to do. Editing files and re-running step 2 updates the matching
remote tests/rules (by filename, or by `testCaseUuid` for a pinned `.yaml`).
`--explicit-mutations` keeps the remote suite in sync with exactly what's in your repo.

To run the now-updated suite against the **deployed** site (not localhost), drop
`--localhost` and point `--url` at the real environment — or omit `--url` to use the
suite's configured URL:

```bash
testrigor test-suite run "$SUITE" \
  --test-cases-path "testrigor/test-cases/**/*.{txt,yaml,yml}" \
  --rules-path "testrigor/rules/**/*.{txt,yaml,yml}" \
  --explicit-mutations \
  --labels smoke \
  --junit-report-save-path testrigor/.run/report.xml
```

## Loop summary

```
edit files  →  run (localhost, --test-cases-path/--rules-path)  →  exit code + report.xml
     ▲                                                                      │
     └──────────────────── fix failing steps ◀──────────────────────────────┘
```

Stop when the exit code is `0` and `report.xml` shows no failures.

## Gotchas

- **Don't use `testrigor authenticate` in automation** — it prompts interactively.
  Use `TESTRIGOR_API_KEY` (or `--auth-token`).
- **Quote your globs** (`"...**/*.txt"`) or the shell expands them and the CLI sees
  only the first match.
- **Keep filenames stable** — renaming a file creates a *new* remote test/rule and
  orphans the old one (unless you pin identity with `testCaseUuid` in `.yaml`).
- **JUnit needs sync mode** — `--junit-report-save-path` produces nothing with `--async`.
- **`--localhost` requires `--url`**, and the URL should be the locally reachable one
  (e.g. `http://localhost:3000`).
- **Don't `open url` the app under test.** The suite's URL (or `--url`) opens
  automatically at the start of every test; hard-coding it breaks other environments.
  Navigate by clicking links; select the environment via the suite config or `--url`.
- **Parameterized rules as files work** (a file `search for "query".txt` is invoked as
  `search for "laptop"`), but the quoted/spaced filename is invalid on Windows — for
  cross-platform repos keep file rules fixed-name and define parameterized rules in the
  Reusable Rules UI.
- Make sure the `--junit-report-save-path` parent directory exists (`mkdir -p`).
- **`--labels` runs leave JUnit testcase names blank** (`name="Test Case UUID="`) — when
  debugging a label-filtered run, identify failures by the `<failure>` message and the
  run URL, not the testcase name.
- A forced `--url` applies to *every* test in the run — scope with `--labels` so tests
  written for other URLs don't fail against it.
- **`.txt` test cases can't carry labels** — only `.yaml` can (`labels: [...]`). If you
  need `--labels` to run a subset, use `.yaml`; otherwise give each app its own suite.
