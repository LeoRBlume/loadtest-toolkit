---
description: Generate a k6 load-test script from a cURL command, with checks, thresholds, and a JSON summary export.
argument-hint: [paste a cURL command, or leave empty to be prompted]
---

You are generating a k6 load-test script from a cURL command. Follow these steps in order.

## 0. Precondition check (non-blocking)

Run `k6 version` once via Bash. If it fails or `k6` is not on PATH, print this one-line notice:

> `k6` is not installed locally. You can still generate the script; you will need k6 to run it. Run `/loadtest-toolkit:setup` for install instructions.

Do not block. The user may want to generate the script on one machine and run it elsewhere. Move on to step 1.

## 1. Get the request

- If `$ARGUMENTS` contains a cURL command, use it.
- Otherwise, ask the user: "Paste the cURL command of the endpoint you want to load-test." Then wait for their reply.

## 2. Parse the cURL

Extract:
- HTTP method (default `GET` if `-X` is absent)
- Full URL (base + path, including query string)
- Headers (every `-H` / `--header`), keeping `Authorization`, `Cookie`, `X-Api-Key` verbatim
- Body / payload (from `-d`, `--data`, `--data-raw`, `--data-binary`, `--data-urlencode`)

Record which headers carry secrets. You will route those through `__ENV` in the generated script.

## 3. Show the test type menu

Read `${CLAUDE_PLUGIN_ROOT}/references/test-types.md` for the full descriptions, then show this menu:

```
Which kind of load test do you want?

1 - Smoke   1 to 2 VUs for 1 minute. Verifies the script works and the
            endpoint responds correctly under almost no load. Run this first.
2 - Load    Expected normal traffic. Ramps up to typical concurrent users,
            holds, ramps down. Answers "how does it behave on a normal day?"
3 - Stress  Beyond normal. Pushes load higher and higher to find the
            breaking point and watch how the system degrades.
4 - Spike   Sudden surge then drop. Simulates a flash sale or viral moment.
5 - Soak    Normal load held for hours. Surfaces memory leaks and slow
            resource exhaustion.

Reply with a number, or describe your goal (e.g. "we expect 200 users at
launch and want to know if we will hold up") and I will pick one with you.
```

## 4. Decide together

- If the user gives a number 1 to 5, use that test type.
- If they describe a goal in plain language, recommend one type, explain why in one or two sentences, and confirm before generating.
- If they give expected concurrency, peak users, or duration, fold those numbers into `options.stages` (overriding the defaults from `test-types.md`).

## 5. Generate the script

Read `${CLAUDE_PLUGIN_ROOT}/references/k6-template.md` and `${CLAUDE_PLUGIN_ROOT}/references/test-types.md`. Then build a complete k6 script saved as `<name>.test.js` (pick a name from the URL path, e.g. `login.test.js` for `/login`). The script must:

- Issue the parsed request with the correct method, full URL, all headers, and body if present.
- Route any auth tokens or secrets through `__ENV` variables. Leave a comment showing the run command, e.g. `// run with: k6 run -e API_TOKEN=xxx login.test.js`.
- Set `export const options = { stages: [...], thresholds: {...} }` using the defaults for the chosen test type from `references/test-types.md`. After generating, tell the user the exact VU and duration numbers you used and that they are starting points to tune.
- Add `check()` calls on the response. At minimum: status is 2xx. Add response-time and body-shape checks where the response gives you something to assert on.
- `sleep(1)` between iterations inside the default function.
- Export `handleSummary(data)` that writes `summary.json` in the shape documented in `${CLAUDE_PLUGIN_ROOT}/references/summary-schema.md`, plus a colorized stdout summary using `textSummary` from `https://jslib.k6.io/k6-summary/0.0.4/index.js`.

## 6. Hand off

- Save the script in the current working directory.
- Print the exact command to run it, e.g.:
  ```
  k6 run login.test.js
  ```
  Include any `-e VAR=value` flags needed for the `__ENV` secrets.
- Tell the user that after the run, `summary.json` will be produced in the working directory and they can pass it to `/loadtest-toolkit:evaluate` to get an HTML report.
