# testRigor skills

Agent skills for working with [testRigor](https://testrigor.com), the AI-based,
plain-English test automation tool. Each subfolder is a self-contained skill
(a `SKILL.md` plus any references) usable by Claude Code / openclaw.

| Skill | Use it to… |
|-------|------------|
| [`testrigor-dev-loop`](testrigor-dev-loop/SKILL.md) | **Start here.** The end-to-end playbook: build tests + reusable rules as files → run against localhost via the tunnel → debug from the JUnit report → update the remote suite by re-running. |
| [`testrigor-write-tests`](testrigor-write-tests/SKILL.md) | Author test cases in testRigor's plain-English language, and the on-disk file formats for test cases / reusable rules. Full command catalog in [`reference.md`](testrigor-write-tests/reference.md). |
| [`testrigor-cli`](testrigor-cli/SKILL.md) | Run/trigger suites with the `testrigor` CLI — auth, the file-mutation (update-remote) model, localhost tunnel, debugging signals, labels, JUnit, mobile builds. |
| [`testrigor-api-testing`](testrigor-api-testing/SKILL.md) | Write `call api` / `mock api` steps — REST calls, JSONPath extraction, status assertions, API+UI chaining. |

Typical flow (all orchestrated by `testrigor-dev-loop`): **build** files
(`testrigor-write-tests`, plus `testrigor-api-testing` for API steps) → **run on
localhost** and **debug** (`testrigor-cli`) → **update remote** by re-running.

## Prerequisite for running

The `testrigor-cli` skill needs the CLI (Node.js ≥ 18):

```bash
npm install -g testrigor-cli
export TESTRIGOR_API_KEY="<YOUR_PERSONAL_AUTH_TOKEN>"   # non-interactive (best for Claude/CI)
```

> `testrigor authenticate` also stores a token, but it prompts interactively — prefer
> the `TESTRIGOR_API_KEY` environment variable for automated/agent use.
