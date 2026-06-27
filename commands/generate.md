---
description: Generate a k6 load-test script from a cURL command, with checks, thresholds, and a JSON summary export.
argument-hint: [paste a cURL command, or leave empty to be prompted]
---

You are generating a k6 load-test script from a cURL command. Follow these steps in order.

## 0. Quick precondition check (non-blocking)

Run `k6 version` once via Bash. If it fails or `k6` is not on PATH, print a one-line notice:

> Heads up: `k6` isn't installed locally. You can still generate the script, but you'll need k6 to run it. Run `/loadtest-toolkit:setup` for install instructions.

Do **not** block on this — the user may want to generate the script on one machine and run it elsewhere. Move on to step 1.

## 1. Get the request

- If `$ARGUMENTS` contains a cURL command, use it.
- Otherwise, ask the user: "Paste the cURL command of the endpoint you want to load-test." Then wait for their reply before proceeding.

## 2. Parse the cURL

Extract:
- HTTP method (default `GET` if none given via `-X`)
- Full URL (base + path, including query string)
- Headers (every `-H` / `--header`), preserving auth headers (`Authorization`, `Cookie`, `X-Api-Key`, etc.) verbatim
- Body / payload (from `-d`, `--data`, `--data-raw`, `--data-binary`, or `--data-urlencode`)

Note any auth headers — you will move secrets to `__ENV` variables in the generated script.

## 3. Explain the test types

Before generating anything, read `${CLAUDE_PLUGIN_ROOT}/references/test-types.md` for the full descriptions, then present this menu to the user:

```
Which kind of load test do you want?

1 - Smoke   — minimal load (1–2 virtual users, ~1 min). Verifies the script works and the
              endpoint responds correctly under almost no load. Always run this first.
2 - Load    — expected/normal traffic. Ramps up to your typical concurrent users, holds,
              ramps down. Answers "how does it behave on a normal day?"
3 - Stress  — beyond normal. Pushes load higher and higher to find the breaking point and
              watch how the system degrades.
4 - Spike   — a sudden, sharp surge then drop. Simulates a flash sale or viral moment.
5 - Soak    — normal load held for a long time (hours). Surfaces memory leaks and slow
              resource exhaustion.

Reply with a number, or just describe your goal (e.g. "we expect 200 users at launch and
want to know if we'll hold up") and I'll pick the right one with you.
```

## 4. Decide together

- If the user gives a number (1–5), use that test type.
- If they describe a goal in plain language, recommend the best-fit type and explain why in one or two sentences. Confirm with the user before generating.
- If they describe expected concurrency, peak users, or duration, fold those numbers into the `options.stages` you generate (overriding the defaults).

## 5. Generate the script

Read `${CLAUDE_PLUGIN_ROOT}/references/k6-template.md` and `${CLAUDE_PLUGIN_ROOT}/references/test-types.md`. Then build a complete k6 script saved as `<name>.test.js` (pick a sensible name from the URL path, e.g. `login.test.js` for `/login`). The script must:

- Issue the parsed request: correct method, full URL, all headers, body if present.
- Move any auth tokens or secrets to `__ENV` variables and leave a comment explaining how to set them (e.g. `# k6 run -e API_TOKEN=xxx login.test.js`).
- Set `export const options = { stages: [...], thresholds: {...} }` using the defaults for the chosen test type from `references/test-types.md`. After generating, tell the user the exact VU/duration numbers you used and that they are starting points to tune.
- Add `check()` calls on the response. At minimum: status is 2xx. Add response-time and body-shape checks where sensible (e.g. response contains an expected field).
- `sleep(1)` between iterations inside the default function.
- Export `handleSummary(data)` that writes `summary.json` in the exact shape documented in `${CLAUDE_PLUGIN_ROOT}/references/summary-schema.md`, plus a readable colorized stdout summary using `textSummary` from `https://jslib.k6.io/k6-summary/0.0.4/index.js`.

## 6. Hand off

- Save the script to the current working directory.
- Print the exact command to run it, e.g.:
  ```
  k6 run login.test.js
  ```
  Include any `-e VAR=value` flags needed for the `__ENV` secrets.
- Remind the user that after the run, `summary.json` will be produced in the working directory and they can pass it to `/loadtest-toolkit:evaluate` to get an HTML report.
