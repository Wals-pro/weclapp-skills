# Load management: 429, queues, timeouts, parallelism

Source labels: **weclapp docs** (section "Load management" in the spec), **Lanig talk** (Christian Lanig, weclapp, "Jeder Request zählt", Community Day 2026), **wals.pro practice**.

## How weclapp protects itself

- **No fixed rate limit.** weclapp wants to keep it that way: parallel requests are allowed, and normal use should feel no limit. *(weclapp docs, Lanig talk)*
- **Queue first, 429 later.** There is a limit on concurrent active requests per tenant. Requests above it are not rejected; they wait, currently up to about 30 seconds. After that the API answers **HTTP 429 Too Many Requests**. *(weclapp docs)*
- **Sustained load counts too.** weclapp looks at effective request time (request-seconds). Many cheap requests are fine; many expensive ones over minutes are not. *(weclapp docs)*
- **Request-seconds = requests x duration x parallel threads**, evaluated over a time window. *(Lanig talk)* Worked example from the talk, an illustration and not a guaranteed per-tenant value: with a 60-second window and a load factor of 3 (180 available request-seconds), 500 requests x 100 ms = 50 request-seconds is harmless, 200 x 400 ms x 2 threads = 160 is still fine, and 500 x 100 ms x 4 threads = 200 leads to about 20 seconds of waiting.
- **A 429 is a signal, not an error to hammer.** Immediate retries add load; back off. *(Lanig talk, weclapp docs)*
- The spec does not document the exact concurrency limit, a `Retry-After` header, or whether limits apply per token or per tenant (it says "per tenant"). Do not hard-code assumptions; measure with the headers below.

## Reading the queue: response headers

When a request had to wait, the response carries *(weclapp docs)*:

| Header | Meaning |
|---|---|
| `X-Weclapp-Wait-Ms` | Milliseconds the request waited before processing started |
| `X-Weclapp-Wait-Reason` | `concurrency` (too many parallel requests), `load` (overall load too high), or `concurrency, load` |

Present on successful responses **and** on 429; absent when nothing waited. Use them to:

- tell a slow endpoint from a delayed request: own processing time = total time minus wait time;
- throttle before you hit 429.

Log, per request: method, path (no query string with filter values, never the token), status, total duration, wait ms, wait reason, correlation ID if present. *(wals.pro practice)*

## Timeouts

- **Client timeout of at least 60 seconds** for every request. A 30-second client timeout equals the maximum queue time and aborts just before the answer would arrive, then triggers a retry, which adds load. *(weclapp docs, wals.pro practice)*
- Optional request headers tell the server how long you are willing to wait *(weclapp docs)*:

| Header | Meaning |
|---|---|
| `X-Weclapp-Wait-Timeout-Ms` | Maximum wait before processing starts; capped by the server limit |
| `X-Weclapp-Request-Timeout-Ms` | Best-effort limit for the whole request including waiting; error type `request_timeout` |

They can only **shorten** server-side limits, never extend them; zero, negative or invalid values are ignored; with both set, the request timeout also caps the queue wait. Align the client timeout with them: the client must wait at least as long as the headers allow.

```bash
curl --compressed \
  -H "AuthenticationToken: <token>" \
  -H "X-Weclapp-Wait-Timeout-Ms: 5000" \
  -H "X-Weclapp-Request-Timeout-Ms: 15000" \
  "https://<tenant>.weclapp.com/webapp/api/v2/party?pageSize=1&properties=id"
```

Short timeouts suit interactive calls. Background jobs keep the 60-second client timeout and the server defaults.

## The 30-second trap

A request that waited in the queue and then started processing may still **commit** after your client gave up. For reads that only wastes a retry. For writes it creates duplicates or "already done" errors on the retry, see [writing.md](writing.md) ("A timeout is not a failure"). Rule: never let a client timeout decide that a write failed.

## Backoff with jitter

Retry only what is safe: GET, HEAD and OPTIONS, plus writes that are provably idempotent (see [writing.md](writing.md)). Statuses worth retrying: 429, 502, 503, 504, network errors, and the weclapp problem type `request_timeout` (an HTTP 400 whose `type` ends in `/request_timeout`). *(weclapp docs: error types; wals.pro practice)*

```js
const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));
const RETRYABLE = new Set([429, 502, 503, 504]);

export async function weclappGet(url, token, { maxRetries = 5, capMs = 60_000 } = {}) {
  for (let attempt = 0; ; attempt += 1) {
    let res;
    try {
      res = await fetch(url, {
        headers: {
          AuthenticationToken: token,
          Accept: 'application/json',
          'Accept-Encoding': 'gzip',
          'User-Agent': 'example-sync/1.0 (ops@example.com)',
        },
        signal: AbortSignal.timeout(60_000), // never below 60 s
      });
    } catch (error) {
      if (attempt >= maxRetries) throw error; // network error or client timeout
      await sleep(backoffMs(attempt, null, capMs));
      continue;
    }
    const waitMs = Number(res.headers.get('x-weclapp-wait-ms') ?? 0);
    if (waitMs > 0) {
      console.warn('weclapp queue', { waitMs, reason: res.headers.get('x-weclapp-wait-reason') });
    }
    if (!RETRYABLE.has(res.status) || attempt >= maxRetries) return res;
    await sleep(backoffMs(attempt, res.headers.get('retry-after'), capMs));
  }
}

function backoffMs(attempt, retryAfter, capMs) {
  const header = Number(retryAfter);
  if (Number.isFinite(header) && header > 0) return Math.min(header * 1000, capMs); // honour it if present
  const base = Math.min(2000 * 2 ** attempt, capMs); // 2 s, 4 s, 8 s, 16 s, 32 s, capped
  return base / 2 + Math.random() * (base / 2); // jitter, so parallel workers do not retry in step
}
```

Python equivalent of the delay: `base = min(2 * 2 ** attempt, 60); delay = base / 2 + random.random() * base / 2`.

Notes:

- Keep **separate budgets** for network/5xx (few, short retries) and 429 (more attempts, longer waits). A single flat "3 retries" is too few for a busy tenant. *(wals.pro practice)*
- Do not follow redirects on write requests, and never send the token to another host. *(wals.pro practice)*
- Classify errors by the part of `type` after the last `/` (for example `optimistic_lock`, `request_timeout`, `validation`), not by message text; messages are localised and change. *(weclapp docs: error JSON structure)*

## Parallelism that adapts

A fixed worker count is either too slow or provokes 429. A controller that reacts to the headers works better *(wals.pro practice; a starting point, tune it on a test tenant)*:

| Signal | Reaction |
|---|---|
| Start | 2 parallel requests, ceiling about 10 |
| 429 | Drop to 1 and pause all workers for the backoff time |
| `X-Weclapp-Wait-Reason` contains `load`, or wait of 2000 ms or more | Halve the slots |
| `X-Weclapp-Wait-Reason: concurrency`, or wait of 250 ms or more | Remove one slot |
| A full window without waits | Add one slot |

Also: serialise steps that must happen in order, count requests first (`/count`) and read pages in bounded waves, and abort a job before it starts if the planned page count exceeds your budget. *(wals.pro practice)*

## Avoid the load in the first place

Biggest levers, in order *(Lanig talk, weclapp docs, wals.pro practice)*:

1. **Drop calls you do not need.** Count the calls per business transaction; the cheapest request is the one you skip.
2. **Filter and project.** `properties=…`, a precise filter, `pageSize` up to the maximum.
3. **No N+1.** `includeReferencedEntities` for referenced records, one `id-in` request instead of one request per ID ([reading.md](reading.md)).
4. **Delta instead of full sync, webhooks instead of polling** ([sync-and-webhooks.md](sync-and-webhooks.md)).
5. **Cache stable metadata**: custom attribute definitions, units, tax rates, warehouses. The spec names custom attribute definitions explicitly.
6. **Spread peaks.** Do not start eleven schedules at minute :00; offset them. A UI that polls every minute on twenty devices is twenty-fold load.
7. **Measure.** Percentiles and "share of requests over 30 s" tell more than an average. Log duration and the wait headers yourself.

Typical anti-patterns named in the talk: full queries on a schedule, permanent polling, N+1 requests, parallel writers competing for the same records.
