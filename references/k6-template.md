# k6 script template

The canonical shape every script produced by `/loadtest-toolkit:generate` should follow.

## Imports

```js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { textSummary } from 'https://jslib.k6.io/k6-summary/0.0.4/index.js';
```

`textSummary` powers a readable colorized stdout block inside `handleSummary`.

## Options

Fill `stages` and `thresholds` from the chosen test type in `test-types.md`:

```js
export const options = {
  stages: [
    // from test-types.md for the chosen type
  ],
  thresholds: {
    // from test-types.md for the chosen type
  },
};
```

## Default function

Build the request from the parsed cURL: method, URL, headers, body. Run `check()`s on the response. `sleep(1)` between iterations.

```js
export default function () {
  const url = 'https://api.example.com/login';

  const params = {
    headers: {
      'Content-Type': 'application/json',
      // Move secrets to __ENV — never hardcode tokens.
      // Run with: k6 run -e API_TOKEN=xxx script.test.js
      'Authorization': `Bearer ${__ENV.API_TOKEN}`,
    },
  };

  const payload = JSON.stringify({
    email: 'user@example.com',
    password: 'hunter2',
  });

  const res = http.post(url, payload, params);

  check(res, {
    'status is 2xx': (r) => r.status >= 200 && r.status < 300,
    'response time < 500ms': (r) => r.timings.duration < 500,
    'body has token': (r) => r.body && r.body.includes('token'),
  });

  sleep(1);
}
```

### Method variants

- `GET` → `http.get(url, params)`
- `POST` / `PUT` → send body and a `Content-Type` header matching the cURL request
- `DELETE` → `http.del(url, null, params)`
- For form bodies, send a URL-encoded string and set `Content-Type: application/x-www-form-urlencoded`.

## handleSummary

The shared contract with `/loadtest-toolkit:evaluate` — write `summary.json` as the full `data` object, plus a stdout summary.

```js
export function handleSummary(data) {
  return {
    'summary.json': JSON.stringify(data, null, 2),
    stdout: textSummary(data, { indent: ' ', enableColors: true }),
  };
}
```

`data` is whatever k6 passes to `handleSummary` — do not strip fields. The schema in `summary-schema.md` documents which paths `evaluate` reads.

## Secrets and env vars

- Never hardcode bearer tokens, cookies, or API keys.
- Read them from `__ENV.NAME` and leave a comment in the script telling the user how to set them on the `k6 run` command line.
