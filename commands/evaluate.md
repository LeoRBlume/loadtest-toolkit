---
description: Turn a k6 run's JSON summary into a standalone HTML performance report.
argument-hint: [paste the summary JSON, or a path to summary.json]
---

You are turning a k6 run's JSON summary into a self-contained HTML report. Follow these steps.

## 1. Get the data

- If `$ARGUMENTS` looks like a file path, read that file.
- If `$ARGUMENTS` looks like JSON (starts with `{`), use it directly.
- Otherwise, ask the user: "Paste the contents of `summary.json`, or give me its path." Then wait.

## 1b. Ask the report language

Ask the user, in the same language they have been talking to you, which language to write the report and the chat takeaways in. Show a numbered menu in the same shape as the test-type menu in `/loadtest-toolkit:generate`:

```
Em qual idioma devo escrever o relatório e os destaques?

1 - Português
2 - English
3 - Español
4 - Outro: me diga qual

Responda com um número.
```

Translate the menu text into the language the user is using, but keep the four option labels as written: `Português`, `English`, `Español`, `Other`/`Outro`/`Otro`.

- Reply is 1, 2, or 3: use that language.
- Reply is 4: ask "Which language?" or "Qual idioma?" and wait.
- Reply is a language name (no number): use it directly.

Use the chosen language for every label, heading, metric name, threshold table, and prose paragraph inside `report.html`, plus the 2 or 3 takeaways printed in chat at step 6.

Numbers, units (ms, %, RPS), HTTP method names, and metric keys from `summary.json` stay in their original form.

## 2. Parse the JSON

Read `${CLAUDE_PLUGIN_ROOT}/references/summary-schema.md` for the field paths. Pull out:

- Total requests: `data.metrics.http_reqs.values.count`
- Requests per second: `data.metrics.http_reqs.values.rate`
- Error rate (0 to 1): `data.metrics.http_req_failed.values.rate`
- Latency in ms: `data.metrics.http_req_duration.values.avg`, `med`, `p(90)`, `p(95)`, `p(99)`
- Checks pass rate (0 to 1): `data.metrics.checks.values.rate`
- Peak VUs: `data.metrics.vus_max.values.value`
- Iterations: `data.metrics.iterations.values.count`
- Total run duration in ms: `data.state.testRunDurationMs`
- Threshold pass / fail: each metric's `thresholds` object

If a field is missing, treat its value as `n/a` and continue. Do not crash on missing fields.

## 3. Judge

Compute the verdict:

- **PASS** if every threshold passed and the error rate is within the configured bound.
- **FAIL** otherwise.

Write a one-line reason naming the main bottleneck, e.g. "p95 latency 820ms exceeded the 500ms threshold" or "error rate 4.2% exceeded the 1% threshold".

## 4. Build the HTML report

Before writing the HTML, try to invoke the **`frontend-design`** skill via the Skill tool to get visual direction (typography, spacing, color hierarchy).

- If `frontend-design` is available, invoke it and apply its guidance to the report.
- If it is not available, use these defaults and continue:
  - One typeface (system stack: `-apple-system, Segoe UI, Roboto, sans-serif`). No decorative fonts.
  - Space between cards, not borders.
  - One accent color: green `#16a34a` for PASS, red `#dc2626` for FAIL. Body text `#1f2937`, background `#fafafa`.
  - Tabular numbers (`font-variant-numeric: tabular-nums`) on every metric so columns align.
  - Headings heavier than body. No all-caps, no shadows, no gradients.

Then write a single self-contained `report.html` to the current working directory. Requirements:

- All CSS inline in a `<style>` tag. No external stylesheets, no CDN scripts, no `<link>` or `<script src="...">` to anything remote. The file opens offline in any modern browser.
- A header band with the verdict (PASS or FAIL) and the one-line reason, color-coded green for PASS, red for FAIL.
- A row of metric cards: total requests, RPS, error rate %, avg latency, p95 latency, p99 latency, checks pass %, peak VUs, total duration.
- A latency percentile breakdown: one horizontal bar per percentile (avg, p90, p95, p99) drawn with plain `<div>` elements and inline CSS widths proportional to value. No chart library.
- A thresholds table: one row per threshold, columns `Threshold` and `Result` (PASS / FAIL), color-coded.
- One short paragraph in the chosen language: what's healthy, what's the bottleneck, what to look at next. Example: "Error rate is fine. p95 latency 820ms is the constraint. Investigate slow DB queries or downstream calls."

Format numbers: latency in ms with 1 decimal, rates as percentages with 2 decimals, durations as `Xm Ys` if over a minute.

## 5. Write the prose without AI tics

The paragraph in the report and the takeaways printed in chat are the only prose you write. Both should read like an engineer wrote them.

Before writing either, try to invoke the **`no-ai-slop`** skill via the Skill tool. The rules apply in every language: write the target language with the same discipline (no filler, no hedges, no clichéd openers).

- If `no-ai-slop` is available, invoke it and follow it for both the report paragraph and the chat takeaways.
- If it is not available, use these minimums and continue:
  - Lead with the actual numbers (`p95 = 820ms`, `error rate = 4.2%`), not adjectives.
  - No filler openers ("It's worth noting", "In summary", "Overall").
  - No hedges ("might potentially", "could possibly"). State it or don't.
  - No em-dashes. Use a period or a comma.
  - Name the bottleneck and the next concrete thing to look at. No marketing tone.

## 6. Hand off

- Save the report and tell the user the path (e.g. `report.html`).
- In chat, print 2 or 3 takeaways: verdict, bottleneck, one concrete next step. Apply the same rules from step 5 to these lines.
