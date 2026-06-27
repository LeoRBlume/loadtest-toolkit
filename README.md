# loadtest-toolkit

A Claude Code plugin that wraps the k6 load-testing loop. **generate** a test script from a cURL command, run it, then **evaluate** the JSON summary into an HTML report.

## What it does

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

Three slash commands. The shared contract between `generate` and `evaluate` is `summary.json`.

## Install

Add this repo as a Claude Code marketplace, then install the plugin:

```
/plugin marketplace add LeoRBlume/loadtest-toolkit
/plugin install loadtest-toolkit@LeoRBlume
```

The marketplace is registered as `LeoRBlume` (the author's GitHub handle). The plugin inside it is `loadtest-toolkit`, so the install ID is `loadtest-toolkit@LeoRBlume`.

You will also need [k6](https://k6.io/docs/getting-started/installation/) installed locally to run the generated scripts.

## Commands

### `/loadtest-toolkit:setup`

One-time environment check. Detects your OS, runs `k6 version`, and prints the install one-liner (winget, brew, apt, dnf) if k6 is missing. Also checks for the optional companion skills `frontend-design` and `no-ai-slop` used by `/loadtest-toolkit:evaluate`. Run this first.

### `/loadtest-toolkit:generate`

Generates a k6 script from a cURL command. Walks you through picking a test type (smoke, load, stress, spike, soak), then writes a complete `<name>.test.js` with `check()`s, thresholds, and a `handleSummary` that produces `summary.json`.

**Example:**

```
/loadtest-toolkit:generate curl -X POST https://api.example.com/login \
  -H "Content-Type: application/json" \
  -d '{"email":"u@example.com","password":"hunter2"}'
```

You can also run it with no arguments. It will ask for the cURL.

### `/loadtest-toolkit:evaluate`

Reads a k6 `summary.json`, judges PASS or FAIL based on thresholds and error rate, and writes a self-contained `report.html` (opens offline, no CDN). Asks which language to render the report in (English, Portuguese, Spanish, or other).

If installed, it picks up the **`frontend-design`** skill for the report styling and **`no-ai-slop`** for the report's prose. Both are optional. If either is missing, the command uses built-in defaults.

**Example:**

```
/loadtest-toolkit:evaluate ./summary.json
```

You can also paste the JSON inline, or run with no arguments and it will ask.

## Optional companion plugins

`/loadtest-toolkit:evaluate` uses two other plugins when they are installed:

- **`frontend-design`**: visual direction for the HTML report (typography, hierarchy, color).
- **`no-ai-slop`**: keeps the report's prose and the chat takeaways free of AI tics.

Both are detected automatically. If absent, `evaluate` uses built-in defaults. Nothing breaks.

## Tuning

The VU and duration numbers baked into each test type are starting points, not guarantees. Tune them to your real expected concurrency and SLOs. See `references/test-types.md` for the defaults.

## Secrets

`generate` moves bearer tokens, cookies, and API keys it finds in your cURL to `__ENV` variables and leaves a comment showing the run command, e.g. `k6 run -e API_TOKEN=xxx login.test.js`. Do not commit secrets to the generated script.
