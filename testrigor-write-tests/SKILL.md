---
name: testrigor-write-tests
description: Author testRigor automated test cases in plain-English commands — navigation, clicks, typing, assertions, reusable rules, variables, data-driven tests, and login/email/SMS/2FA flows. Use when writing, editing, or converting manual test steps into testRigor test cases (as `test-cases/*.txt` files or steps pasted into the testRigor app), or when asked how to express a step in testRigor's language. To run suites use the `testrigor-cli` skill; for API steps see `testrigor-api-testing`.
---

# Writing testRigor Test Cases

testRigor tests are written in **plain English**, one command per line, from the
perspective of an end user. There are no CSS/XPath selectors — you refer to
elements by the text the user sees ("Sign in", "Email", "Add to Cart").

This skill covers authoring. For the **complete command catalog** read
[reference.md](reference.md). To **execute** what you write, use the
`testrigor-cli` skill.

## The 6 golden rules

1. **One action per line.** Each step is a command on its own line.
2. **Describe what a user sees**, not the DOM. `click "Add to Cart"`, never a selector.
3. **Quote everything the user reads.** Element captions, input text, and expected
   text all go in `"double quotes"`.
4. **Assert often.** After meaningful actions add a `check that ...` so failures are
   pinpointed. A test with no assertions proves nothing.
5. **Prefer built-in reusable rules** (`login`, `fill out form`, `purchase "X"`)
   over re-implementing flows by hand.
6. **Use variables for anything generated or reused** (`save as`, `grab ... and save
   it as`, `stored value "..."`) — never hard-code a value you produced earlier.
7. **Don't `open url` the app under test.** The suite's start URL (or the CLI `--url`)
   opens automatically at the start of every test; hard-coding a URL breaks runs across
   environments (local / staging / prod). Navigate *within* the app by clicking links,
   and choose the environment via the suite config or `--url` — not in the test.

## Most-used commands (the 90% you'll write daily)

```
open url "https://example.com"        // only to reach a NEW page; the suite's URL opens automatically

click "Sign in"
click on the 2nd "Edit"
enter "user@test.com" into "Email"
enter stored value "password" into "Password"
select "United States" from "Country"
type "free text"                      // types into the focused field

check that page contains "Welcome back"
check that page contains "Order #" using OCR
check that "Cart" contains "3 items"
check that button "Checkout" is disabled

scroll down until page contains "Footer"
hover over "Account"
wait 3 seconds                        // max 2 minutes; avoid unless necessary

save value "Acme Corp" as "company"
grab value from "Total" and save it as "orderTotal"
generate unique email, then enter into "Email" and save as "newEmail"
```

## Referring to elements precisely

When a caption is ambiguous, narrow it down — see reference.md for the full grammar:

```
click "Delete" below "Invoice #42"          // relative position
click "Submit" in the context of "Billing"  // scope to a section/container
click the 3rd "Remove"                       // by index
click button "Save"                          // by element type (button/link/input/checkbox/dropdown...)
click exactly "Save"                         // exact, case-sensitive match
click "Continue" if exists                   // conditional — no failure if absent
```

Match form fields by their **visible label** (`into "Email address"`), not a
placeholder — placeholders are often vague or repeated across fields, so they can match
the wrong input.

## Reusable rules

Built-in rules cover common flows — call them by name:

```
login                       // uses the suite's username/password credentials
logout
sign up with email confirmation
purchase "Wireless Mouse"
fill out form               // also: fill out required fields in form / with generated values
```

Define **custom rules** in the suite's *Reusable Rules* section, then call them like
a sentence. Parameters are the quoted words:

```
Rule:  go to checkout for "product"
Steps: login
       search for "product"
       click "product"
       click "Add to Cart"
       click "Checkout"

Use:   go to checkout for "Wireless Mouse"
```

## Data-driven tests

Drive one test with many rows by referencing a table or variables file
(`--variables-path` in the CLI). In the suite, attach a data set and reference
columns as `stored value "columnName"`:

```
enter stored value "email" into "Email"
enter stored value "plan" into "Plan"
check that page contains stored value from "expectedMessage"
```

## A complete example test

```
// Login and place an order, verifying each stage
login
check that page contains "My Account"

click "Catalog"
click "Wireless Mouse"
click "Add to Cart"
check that "Cart" contains "1"

click "Checkout"
generate unique email, then enter into "Email" and save as "buyerEmail"
fill out the rest of the required fields in form
click "Place Order"

check that page contains "Thank you for your order"
grab value of "Order #\d+" and save it as "orderId"
check that email to stored value "buyerEmail" was received
```

## Make a test actually prove something (avoid false passes)

A green run only means *no assertion failed* — not that the feature works. A test can
pass while silently doing nothing useful. Guard against it:

- **Assert the consequence, not just the navigation.** "Reached the Thank-you page" is
  a weak check if the button is really a link — it passes even with an empty/unfilled
  form. Assert the state that *proves* the action worked (an order number, a sent
  email, an updated total).
- **Test the negative, not only the happy path.** To prove a field is required, a
  button disables, or input is rejected, do the *wrong* thing and assert it's blocked:
  ```
  enter "buyer@test.com" into "Email address"
  // deliberately leave "Expiration" empty
  click "Purchase"
  check that page contains "Checkout"                          // still on the form
  check that page doesn't contain "Thank you for your purchase!"
  ```
- **A skipped or mis-targeted step won't always fail.** If an `enter ... into "X"` is
  omitted or matches the wrong field, the test still passes unless a later assertion
  depends on it — so assert intermediate state where it matters.
- **The app must actually enforce what you test.** A test can't catch validation the
  page never performs; if "required" matters, the markup must enforce it *and* the
  negative test above keeps it honest.

## Test cases & reusable rules as files

To keep tests in a repo and drive them with the CLI, store each test case and each
reusable rule as its own file. **The filename (without extension) is the name** — for
a test case that's its title; for a rule that's the keyword you invoke it by. This is
exactly how the CLI's `--test-cases-path` / `--rules-path` parse them.

Accepted extensions: `.txt` and `.yaml`/`.yml` for test cases and rules; `.json` for
variables; `.yaml`/`.yml`/`.json` for settings.

### Test case files

Plain text — the whole file is the steps:

```
# file: test-cases/login.txt   →   test case named "login"
login                              // the suite's start URL opens automatically — no `open url`
check that page contains "My Account"
```

YAML — when you need labels, a dataset, or to pin to an existing test case:

```yaml
# file: test-cases/login.yaml   →   test case named "login"
customSteps: |
  open url "/login"
  login
  check that page contains "My Account"
labels: [smoke, auth]      # optional
datasetName: "Users"       # optional — bind a data set for data-driven runs
testCaseUuid: "..."        # optional — update THIS existing test case instead of matching by name
```

### Reusable rule files

A rule is a named, reusable block of steps. The filename is the rule's name; **invoke
it from any test case by writing that name as a step.**

```
# file: rules/add-to-cart.txt   →   rule named "add-to-cart"
click "Add to Cart"
check that "Cart" contains "1"
```

```yaml
# file: rules/add-to-cart.yaml   →   rule named "add-to-cart"
steps: |
  click "Add to Cart"
  check that "Cart" contains "1"
labels: [cart]             # optional
```

Then in a test case:

```
login
add-to-cart                // runs rules/add-to-cart.txt here
click "Checkout"
```

**Parameterized rules as files** — these *do* work (verified): name the file with the
quoted placeholder and reference it inside via `stored value`. A file
`rules/search for "query".txt` containing
`enter stored value "query" into "Search"` is invoked from a test as
`search for "laptop"`. Caveat: that filename contains `"` and spaces, which is
**invalid on Windows** and awkward in shells — so for cross-platform repos, keep
file-based rules fixed-name (like `open-search-page`) and define parameterized rules
in the suite's *Reusable Rules* UI instead.

### Variables file

A flat JSON object; each key becomes a stored value referenced as `stored value "key"`:

```json
{ "username": "qa@test.com", "password": "secret", "plan": "Pro" }
```

### Running them

Use the `testrigor-cli` skill — running with these flags also **pushes the files into
the remote suite** (see `testrigor-dev-loop` for the full build → run → debug → update
loop):

```bash
testrigor test-suite run "$SUITE" \
  --test-cases-path "test-cases/**/*.{txt,yaml,yml}" \
  --rules-path "rules/**/*.{txt,yaml,yml}" \
  --variables-path variables.json
```

## When you're unsure of the exact keyword

Read [reference.md](reference.md) — it lists every command category with exact
syntax. testRigor is forgiving about phrasing, but using the documented keywords
(`click`, `enter ... into ...`, `check that page contains ...`, `grab ... and save
it as ...`) is the most reliable. If a step is genuinely visual or ambiguous, add
`using AI` (e.g. `check that page "shows a success banner" using ai`).
