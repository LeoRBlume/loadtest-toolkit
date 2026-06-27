# k6 test types

This file is the source of truth for the five test types `/loadtest-toolkit:generate` offers. Each section gives a short description, the question it answers, and the default `options.stages` and `thresholds` to drop into the generated script.

> **All VU and duration numbers below are starting points.** Tell the user explicitly when generating that they should tune these to their system's expected concurrency and SLOs.

---

## 1. Smoke

A very small run with 1–2 virtual users for about a minute. The goal is not load — it's confirming the script itself works end-to-end (URL, headers, auth, body parse cleanly) and the endpoint responds correctly under almost no pressure. **Always run a smoke test before any larger test.**

**Question it answers:** "Does the script work and does the endpoint respond correctly at all?"

```js
export const options = {
  vus: 1,
  duration: '1m',
  thresholds: {
    http_req_failed: ['rate<0.01'],
    http_req_duration: ['p(95)<500'],
  },
};
```

---

## 2. Load

Simulates the system's expected normal traffic: ramp up to your typical concurrent users, hold there, ramp back down. The numbers should reflect the *real* expected concurrency.

**Question it answers:** "How does the system behave on a normal day?"

```js
// 50 VUs is a placeholder for expected normal concurrency — replace with the user's number.
export const options = {
  stages: [
    { duration: '2m', target: 50 },
    { duration: '5m', target: 50 },
    { duration: '2m', target: 0 },
  ],
  thresholds: {
    http_req_failed: ['rate<0.01'],
    http_req_duration: ['p(95)<500'],
    checks: ['rate>0.99'],
  },
};
```

---

## 3. Stress

Push load progressively well past normal traffic to find the breaking point and observe how the system degrades — does it slow gracefully, time out, or crash?

**Question it answers:** "Where does it break, and how does it fail?"

```js
export const options = {
  stages: [
    { duration: '2m', target: 100 },
    { duration: '5m', target: 200 },
    { duration: '5m', target: 300 },
    { duration: '2m', target: 0 },
  ],
  thresholds: {
    http_req_failed: ['rate<0.05'],
    // Latency is observed rather than hard-capped under stress — degradation is expected.
  },
};
```

---

## 4. Spike

A sudden sharp surge in users followed by a quick drop. Simulates a flash sale, viral moment, or notification blast. The focus is on whether the system survives the surge and how quickly it recovers.

**Question it answers:** "What happens during a sudden surge, and can the system recover?"

```js
export const options = {
  stages: [
    { duration: '30s', target: 500 },
    { duration: '1m', target: 500 },
    { duration: '30s', target: 0 },
  ],
  thresholds: {
    http_req_failed: ['rate<0.10'],
  },
};
```

---

## 5. Soak

Hold normal load for a long time (typically hours). Surfaces slow-burn problems: memory leaks, connection-pool exhaustion, log-disk fill, garbage-collection pauses that only show up after a while.

**Question it answers:** "Does the system stay healthy when load is sustained for hours?"

```js
export const options = {
  stages: [
    { duration: '2m', target: 50 },
    { duration: '2h', target: 50 },
    { duration: '2m', target: 0 },
  ],
  thresholds: {
    http_req_failed: ['rate<0.01'],
    http_req_duration: ['p(95)<500'],
  },
};
```
