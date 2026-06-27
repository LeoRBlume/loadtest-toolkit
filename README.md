# loadtest-toolkit

A Claude Code plugin that turns a cURL command into a complete k6 load-test script, and turns the run's JSON summary into an HTML report. Three slash commands cover the loop.

## The loop

```
/loadtest-toolkit:setup  (checks k6)
       │
       ▼
cURL command  ──►  /loadtest-toolkit:generate  ──►  <name>.test.js
                                                          │
                                                    k6 run <name>.test.js
                                                          │
                                                          ▼
                                                    summary.json
                                                          │
                                                          ▼
                  /loadtest-toolkit:evaluate  ──►  report.html
```

`summary.json` is the contract shared by `generate` and `evaluate`. The generated script writes it, and `evaluate` reads it.

## Prerequisites

Two things on your machine:

1. **Claude Code.** If you don't have it, install from <https://claude.ai/code>.
2. **k6.** Required to run the scripts the plugin generates. The `setup` command will detect your OS and print the install one-liner if k6 is missing. You can also install it manually from <https://k6.io/docs/getting-started/installation/>.

You don't need to install k6 before installing the plugin. The plugin still generates scripts without it, and `setup` will guide you when you're ready to run them.

## Step 1: Install the plugin

Inside any Claude Code session, run these three commands:

```
/plugin marketplace add LeoRBlume/loadtest-toolkit
/plugin install loadtest-toolkit@LeoRBlume
/reload-plugins
```

`LeoRBlume` is the marketplace name (the author's GitHub handle). `loadtest-toolkit` is the plugin inside it. After `/reload-plugins`, three new commands become available:

- `/loadtest-toolkit:setup`
- `/loadtest-toolkit:generate`
- `/loadtest-toolkit:evaluate`

## Step 2: Check your environment

Run:

```
/loadtest-toolkit:setup
```

The command detects your OS, runs `k6 version`, and prints the install one-liner if k6 is missing (winget on Windows, brew on macOS, apt or dnf on Linux). It also checks for two optional companion plugins (`frontend-design` and `no-ai-slop`) that improve the final report. Both are optional. The output looks like this:

```
✓ k6 0.50.0 detected
✓ frontend-design skill available
○ no-ai-slop not installed, optional
✓ Environment ready
```

## Step 3: Generate a test script

Have a cURL command ready. The fastest source: Chrome DevTools > Network tab > right-click the request > "Copy as cURL". Postman and Swagger also export cURL.

Then run:

```
/loadtest-toolkit:generate
```

You can paste the cURL on the same line, or wait for the plugin to ask.

What happens next:

1. The plugin parses your cURL: method, URL, headers, body.
2. It shows a menu of test types (smoke, load, stress, spike, soak) and the question each one answers. You can pick a number, or describe your goal in plain language ("we expect 200 users at launch") and the plugin recommends one.
3. It writes a `<name>.test.js` file in the current directory with `check()` calls, thresholds for the chosen test type, and a `handleSummary` that produces `summary.json` when you run k6.
4. It tells you the exact VU and duration numbers it used and prints the run command.

Auth headers and tokens get routed through `__ENV` variables. The generated script includes a comment with the run command, so secrets stay out of the file.

> Run a **smoke test first** if you're working on a new endpoint. Smoke is a 1-minute, 1-VU sanity check that the script and endpoint actually work. Skip it and you find out your auth header was wrong several minutes into a stress test.

## Step 4: Run k6

This step happens outside Claude Code, in your terminal:

```
k6 run login.test.js
```

If your script uses environment variables for secrets, pass them with `-e`:

```
k6 run -e API_TOKEN=xxx login.test.js
```

When the test finishes, k6 prints a colorized summary in the terminal and writes `summary.json` to the current directory. That JSON is the input for step 5.

## Step 5: Evaluate the results

Back in Claude Code, run:

```
/loadtest-toolkit:evaluate ./summary.json
```

You can also paste the JSON inline, or run with no arguments and the plugin will ask.

The plugin first asks which language to write the report in (Português, English, Español, or other). Then it reads the JSON, judges the run as PASS or FAIL based on thresholds and error rate, and writes `report.html` in the current directory.

The report:

- Opens in any browser, fully offline. No CDN, no external fonts, no JavaScript dependencies. You can email it or attach it to an issue without anything breaking.
- Has a verdict band at the top (green for PASS, red for FAIL) with the main reason.
- Lists the key metrics: total requests, RPS, error rate, average and p95 / p99 latency, checks pass rate, peak VUs, total duration.
- Renders the latency percentiles as horizontal bars.
- Includes a thresholds table with pass / fail per rule.
- Ends with a short paragraph naming the bottleneck and the next thing to look at.

Claude also prints 2 or 3 takeaways in chat: the verdict, the bottleneck, and one concrete next step.

## The test types

Five types are available. The VU counts and durations are starting points. Tune them to your real expected traffic.

- **Smoke**: 1 to 2 VUs, 1 minute. Confirms the script and endpoint work. Run this first.
- **Load**: ramps to your typical concurrent users, holds, ramps down. Answers "how does it behave on a normal day?"
- **Stress**: pushes past normal traffic. Answers "where does it break?"
- **Spike**: sudden surge then drop. Simulates a flash sale or viral moment.
- **Soak**: normal load held for hours. Surfaces memory leaks and slow resource exhaustion.

Full descriptions and the default `options` blocks for each: `references/test-types.md`.

## Tuning the defaults

If you accept the defaults, a Load test ramps to 50 VUs. If your real expected concurrency is 500, that test is meaningless. Two ways to override:

- Describe your real numbers when prompted ("we expect 200 concurrent users for 10 minutes"). The plugin folds them into `stages` directly.
- After the script is generated, edit the `stages` array in the `.test.js` file. Re-run k6.

## Secrets

The generator detects auth headers (`Authorization`, `Cookie`, `X-Api-Key`) and routes them through `__ENV` instead of hardcoding them. Example output in the generated script:

```js
// Run with: k6 run -e API_TOKEN=xxx login.test.js
'Authorization': `Bearer ${__ENV.API_TOKEN}`,
```

Don't commit the generated test files if your URL or body contains anything sensitive.

## Optional companion plugins

Two plugins compose with this one when installed. Both are optional; `evaluate` falls back to internal defaults if either is missing.

- **`frontend-design`** shapes the visual style of the HTML report (typography, hierarchy, color choices).
- **`no-ai-slop`** shapes the prose in the report and the chat takeaways. No filler, no hedges, claim-then-proof.

`/loadtest-toolkit:setup` detects both and tells you which are available.

## Updating and uninstalling

To pull the latest version after a release:

```
/plugin marketplace update LeoRBlume
/reload-plugins
```

To remove the plugin:

```
/plugin uninstall loadtest-toolkit@LeoRBlume
/plugin marketplace remove LeoRBlume
```

## Troubleshooting

**`/loadtest-toolkit:*` commands don't appear after install.**
Run `/reload-plugins`. If they still don't appear, run `/plugin` to confirm the plugin is enabled.

**`k6: command not found` when running the generated script.**
Run `/loadtest-toolkit:setup`. It will print the install command for your OS.

**Report opens but looks bare.**
You probably don't have `frontend-design` installed. The report is using the built-in default styling. Install `frontend-design` and re-run `/loadtest-toolkit:evaluate` for richer styling.

**`summary.json` not generated after `k6 run`.**
Check that the generated script's `handleSummary` function is intact. If you edited the script and removed it, the file will not be produced. Regenerate with `/loadtest-toolkit:generate`.

## Repository

<https://github.com/LeoRBlume/loadtest-toolkit>
