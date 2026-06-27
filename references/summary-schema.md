# summary.json schema

The contract shared by both commands:
- `/loadtest-toolkit:generate` writes it via `handleSummary` (full `data` object, unmodified).
- `/loadtest-toolkit:evaluate` reads it to build the HTML report.

`summary.json` is the object k6 passes to `handleSummary(data)`. See the [k6 end-of-test summary docs](https://k6.io/docs/results-output/end-of-test/custom-summary/) for the upstream definition.

## Fields `evaluate` reads

| Metric                  | Path                                             | Notes                              |
|-------------------------|--------------------------------------------------|------------------------------------|
| Total requests          | `data.metrics.http_reqs.values.count`            | integer                            |
| Requests per second     | `data.metrics.http_reqs.values.rate`             | float, req/s                       |
| Error rate              | `data.metrics.http_req_failed.values.rate`       | 0 to 1                             |
| Latency avg             | `data.metrics.http_req_duration.values.avg`      | ms                                 |
| Latency median          | `data.metrics.http_req_duration.values.med`      | ms                                 |
| Latency p90             | `data.metrics.http_req_duration.values["p(90)"]` | ms                                 |
| Latency p95             | `data.metrics.http_req_duration.values["p(95)"]` | ms                                 |
| Latency p99             | `data.metrics.http_req_duration.values["p(99)"]` | ms                                 |
| Checks pass rate        | `data.metrics.checks.values.rate`                | 0 to 1                             |
| Peak VUs                | `data.metrics.vus_max.values.value`              | integer                            |
| Iterations              | `data.metrics.iterations.values.count`           | integer                            |
| Total run duration (ms) | `data.state.testRunDurationMs`                   | ms                                 |
| Thresholds              | Each `data.metrics.<name>.thresholds`            | object: `{ <expr>: { ok: bool } }` |

## Threshold shape

For each metric that has thresholds defined in the script, k6 attaches a `thresholds` object:

```json
"http_req_duration": {
  "thresholds": {
    "p(95)<500": { "ok": true }
  },
  "values": { "avg": 123.4, "p(95)": 410.2, "...": "..." }
}
```

`evaluate` iterates every metric in `data.metrics`, and for each one that has a `thresholds` object, lists each entry's expression and its `ok` boolean.

## Graceful degradation

Not every k6 run emits every metric. For example, `checks.values.rate` is only present if the script called `check()`. When a field is missing:

- Show `n/a` in the report instead of `0`, `NaN`, or `undefined`.
- Skip the corresponding card, bar, or row. Do not crash.

The verdict (PASS or FAIL) uses only the thresholds present, plus the error rate when available.
