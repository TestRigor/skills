---
name: testrigor-api-testing
description: Write API test steps inside testRigor test cases — call REST endpoints (GET/POST/PUT/DELETE) with headers and bodies, extract response fields by JSONPath, assert on HTTP status codes, save values for later UI steps, and mock API responses. Use when testing an API directly in testRigor, validating a backend response, seeding/reading data via API as part of a UI flow, or stubbing a dependency with a mock. For general UI commands see `testrigor-write-tests`; to run suites see `testrigor-cli`.
---

# API Testing in testRigor

testRigor calls APIs from inside the same plain-English test cases used for UI —
so you can seed data, read it back, assert on responses, and feed values into UI
steps, all in one test. API steps use the `call api` command; stubs use `mock api`.

## Anatomy of `call api`

A `call api` step chains clauses with `and`:

```
call api <method> "<url>"
  with headers "<H1>" and "<H2>" ...
  and body "<payload>"
  and get "<JSONPath>"
  and save it as "<variable>"
  and then check that http code is <code>
```

Every clause after the URL is optional. `<method>` defaults to GET if omitted.

## GET — fetch and extract a field

```
call api get "https://jsonplaceholder.typicode.com/users/10" with headers "Content-Type:application/json" and "Accept:application/json" and get "$..catchPhrase" and save it as "searchWord" and then check that http code is 200
```

- **Headers**: one quoted `"Name:Value"` per header, joined with `and`.
- **`get "<JSONPath>"`**: extracts part of the JSON response. `$` is the root;
  `$.data.name` walks keys; `$..field` finds `field` at any depth;
  `$.items[0].id` indexes arrays. You must `save it as` to keep the value.
- **`check that http code is 200`**: asserts the status.

## POST / PUT / DELETE — send a body

```
call api post "https://api.example.com/users" with headers "Content-Type:application/json" and body "{\"name\":\"James\",\"role\":\"admin\"}" and get "$.id" and save it as "newUserId" and then check that http code is 201
```

```
call api put "https://api.example.com/users/${newUserId}" with headers "Content-Type:application/json" and body "{\"role\":\"editor\"}" and then check that http code is 200
```

```
call api delete "https://api.example.com/users/${newUserId}" and then check that http code is 204
```

Notes:
- Escape inner quotes in the JSON body (`\"`).
- Interpolate saved values into URL/body with `${variableName}`.
- For a body too large to inline, read it from a variable:
  `... and body stored value "requestBody" ...`.

## Asserting on the response

```
... and then check that http code is 200
check that stored value "searchWord" itself contains "Multi-layered"
```

`get "<JSONPath>" and save it as "x"` captures a field; then assert on it with the
standard checks from `testrigor-write-tests`
(`check that stored value "x" itself contains "..."`).

## Chaining API + UI (the common pattern)

Fetch a value via API, then drive the UI with it:

```
// 1. Pull a value straight from the backend
call api get "https://jsonplaceholder.typicode.com/users/10" with headers "Accept:application/json" and get "$..catchPhrase" and save it as "searchWord" and then check that http code is 200

// 2. Use it in the browser
open url "https://example.com"
enter stored value "searchWord" into "Search"
click "Search"
check that page contains "Nothing Found"
```

Or the reverse — set up state via API before a UI test, and verify via API after:

```
call api post "https://api.example.com/cart" with headers "Content-Type:application/json" and body "{\"sku\":\"WM-1\",\"qty\":2}" and then check that http code is 200
open url "https://example.com/cart"
check that page contains "2"
```

## Mocking APIs

Stub a dependency so the UI sees a controlled response (great for error states and
flaky third parties):

```
mock api call "https://example.com/api/profile" returning body "{\"name\":\"Test User\"}"

mock api call POST "https://example.com/todos" with headers "Content-Type:application/json" returning payload "Created" and http code 200

mock api call GET "https://example.com/api/balance" returning body "{\"error\":\"service unavailable\"}" and http code 503
```

After mocking, the matching front-end request receives your stub instead of the real
response — then assert the UI handled it.

## Tips

- Keep secrets (API keys, tokens) in suite variables and reference them as
  `stored value "apiKey"` or inside a header string, never hard-coded.
- `$..field` (recursive) is the most forgiving JSONPath when you don't know the
  exact nesting; tighten to `$.path.to.field` once you do.
- Always assert `http code` on writes (POST/PUT/DELETE) — a 2xx is your proof the
  call landed.
- Run these tests the same way as any other suite — see the `testrigor-cli` skill.

> Official guide: https://testrigor.com/how-to-articles/how-to-do-api-testing-using-testrigor/
