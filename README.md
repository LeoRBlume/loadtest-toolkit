# loadtest-toolkit

A Claude Code plugin that covers the full k6 load-testing loop: **generate** a test script from a cURL command, run it, then **evaluate** the JSON summary into a readable HTML report.

## What it does

```
/loadtest-toolkit:setup  (verifica k6)
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

Three slash commands, one shared `summary.json` contract between generate and evaluate.

## Install

Add this repo as a Claude Code marketplace, then install the plugin:

```
/plugin marketplace add LeoRBlume/loadtest-toolkit
/plugin install loadtest-toolkit@LeoRBlume
```

The marketplace is registered as `LeoRBlume` (the author's GitHub handle), and the plugin inside it is `loadtest-toolkit` — hence `loadtest-toolkit@LeoRBlume`.

You will also need [k6](https://k6.io/docs/getting-started/installation/) installed locally to actually run the generated scripts.

## Commands

### `/loadtest-toolkit:setup`

One-time environment check. Detects your OS, verifies `k6` is on PATH, and prints the right install command (winget / brew / apt / dnf) if it's missing. Also checks for the optional companion skills `frontend-design` and `no-ai-slop` used by `/loadtest-toolkit:evaluate`. Run this first.

### `/loadtest-toolkit:generate`

Generates a k6 script from a cURL command. Walks you through picking a test type (smoke / load / stress / spike / soak), then writes a complete `<name>.test.js` with `check()`s, thresholds, and a `handleSummary` that produces `summary.json`.

**Example:**

```
/loadtest-toolkit:generate curl -X POST https://api.example.com/login \
  -H "Content-Type: application/json" \
  -d '{"email":"u@example.com","password":"hunter2"}'
```

You can also run it with no arguments — it will ask for the cURL.

### `/loadtest-toolkit:evaluate`

Reads a k6 `summary.json`, judges PASS / FAIL based on thresholds and error rate, and writes a self-contained `report.html` (opens offline, no CDN). Asks which language to render the report in (English, Portuguese, etc.).

If installed, it picks up the **`frontend-design`** skill for richer styling and **`no-ai-slop`** for cleaner prose. Both are optional — the command falls back to built-in defaults if either is missing.

**Example:**

```
/loadtest-toolkit:evaluate ./summary.json
```

You can also paste the JSON inline, or run with no arguments and it will ask.

## Optional companion plugins

`/loadtest-toolkit:evaluate` can leverage two other plugins if they're installed:

- **`frontend-design`** — visual direction for the HTML report (typography, hierarchy, color).
- **`no-ai-slop`** — keeps the report's prose and chat takeaways free of AI tics.

Both are detected automatically. If absent, `evaluate` falls back to minimal built-in defaults — nothing breaks.

## Tuning

The VU and duration numbers baked into each test type are **starting points**, not guarantees. Tune them to your actual expected concurrency and SLOs. See `references/test-types.md` for the defaults.

## Secrets

`generate` will move bearer tokens, cookies, and API keys it finds in your cURL to `__ENV` variables and leave a comment on how to run with them — e.g. `k6 run -e API_TOKEN=xxx login.test.js`. Don't commit secrets to the generated script.
