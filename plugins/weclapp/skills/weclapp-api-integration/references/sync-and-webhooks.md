# Sync and webhooks: delta, events, idempotency

Source labels: **weclapp docs**, **Lanig talk**, **wals.pro practice**. Base URL: `https://<tenant>.weclapp.com/webapp/api/v2/`, header `AuthenticationToken: <token>`.

The talk's symptom-to-cause-to-fix table: load peaks come from full syncs (fix: delta sync), repetitive identical calls from polling (fix: webhooks), many single requests from N+1 (fix: `referencedEntities`, see [reading.md](reading.md)). *(Lanig talk)*

## Delta sync with `lastModifiedDate`

There is no change feed, `ETag` or `If-Modified-Since`; the incremental read is a filter on `lastModifiedDate`. *(weclapp docs: no such mechanism documented; wals.pro practice)*

```text
GET /article?lastModifiedDate-ge=<cursor-ms>&sort=lastModifiedDate&properties=id,articleNumber,name,lastModifiedDate&pageSize=1000
```

Cursor rules *(wals.pro practice, design rules to prove on a test tenant: filter effectiveness, millisecond resolution, clock skew)*:

1. **Window, not "since now".** Define the window `[W, U]`. `U` is the **start** time of this run, `W` the `U` of the last fully successful run. Do not set the next cursor to the run's *finish* time: changes made while the run was reading (minutes, for big sets) would fall through the gap.
2. **Inclusive lower bound, plus overlap.** Use `-ge`, and subtract a safety overlap (for example a few minutes) for clock skew and late commits. Overlap re-delivers records, so the processing step must be idempotent (upsert by `id`, compare `version` or a content hash, skip no-op writes).
3. **Advance the cursor only after the whole window succeeded.** Any failed page or failed write: keep `W`, run again.
4. **Distrust the filter.** If a response contains rows outside the window, or the filter evidently did nothing (see [reading.md](reading.md): unknown filters are ignored), fail the run; do not filter locally and carry on.
5. Project narrowly; count first with `/count` if the window might be large.

## A delta is not a delete signal

`lastModifiedDate-ge` returns changed records; deleted records appear nowhere and there is no tombstone documented. *(wals.pro practice)*

- Detect deletions with a deliberately infrequent **full ID reconcile**: read only `id` (`properties=id`, `sort=id`, all pages), diff against your store. Delete locally **only if every page succeeded**; any error means zero deletes.
- Or subscribe to `atDelete` webhooks and keep the reconcile as the safety net.
- General rule: an empty or failed response is not an empty truth. Never "delete what the source did not return". A reconciliation that ran on an empty fetch after an outage has removed live reference data in production before. *(wals.pro practice)*
- Keep a targeted by-ID read as the freshness fallback for risk-relevant paths instead of a hidden full sync.

## Webhooks

weclapp webhooks are an official resource: `/webhook` (list, create) and `/webhook/id/<id>` (read, update, delete). Properties from the schema *(spec)*: `entityName`, `atCreate`, `atUpdate`, `atDelete`, `requestMethod` (`GET` or `POST`), `url`, plus read-side `deactivatedDate` and `errorMessage`.

```http
POST /webhook
Content-Type: application/json

{
  "entityName": "salesOrder",
  "atCreate": true,
  "atUpdate": true,
  "atDelete": false,
  "requestMethod": "POST",
  "url": "https://integration.example.com/hooks/weclapp?k=<secret-path-token>"
}
```

What the spec does **not** document: payload format, signing, retry policy, auto-deactivation rules, ordering, duplicates. What was observed in production *(wals.pro practice, verify on a test tenant)*:

- The notification is a **minimal payload**: roughly entity ID, entity name and event type, **not the record**. There are three events per entity (create, update, delete), no finer events, and no event for specific business actions such as stock bookings.
- A webhook has exactly **one target URL**. Registrations are not removed when you delete the flow behind them: clean up the webhook itself.
- Listing may report `count: 0` while `result` holds entries; read `result` and page by length.
- To manage webhooks, the role right on the webhook entity (update) covers create, read and delete; other webhook rights are rejected in role definitions.
- `deactivatedDate` and `errorMessage` suggest that failing targets get deactivated. Monitor them: a webhook that switched itself off looks like "no events".

### Receiver pattern

1. **Acknowledge fast** (2xx), do the work asynchronously (queue). Do not call weclapp synchronously inside the request unless it is trivial.
2. **Never trust the payload as truth.** It is a hint. GET the entity by ID with your own token and decide from the fresh state; an unauthenticated URL can be called by anyone, so also protect the URL (long random path token, allow-listing where possible, no secrets in logs) and treat it as an untrusted trigger.
3. **Exit early** on the fresh state: wrong status, wrong document type, already processed.
4. **State in the record, not in your memory.** Custom-attribute checkboxes such as "label created" make processing resumable and visible; a human can reset one to trigger reprocessing. *(wals.pro practice)*
5. **Lock per entity** with a lease (for example five minutes), re-read after acquiring it (double check), release in `finally`.
6. **At-least-once thinking.** The same event can arrive twice, in parallel, or late. Make every handler idempotent.

### Storms and self-triggering

One business transaction commonly produces one create plus several updates within seconds, and **your own writes (status PUTs, created follow-up documents) change the entity and fire the webhook again.** Observed: two dozen orders causing about twice as many flow runs, and parallel runs seeing different states and failing on race conditions. *(wals.pro practice)*

- **Debounce per entity ID** (coalesce events for a few seconds into one run).
- **Check what already exists before creating** (`GET /shipment?salesOrderId-eq=<id>`), then act. A fixed wait is only a stop-gap.
- Options and their risks: concurrency 1 on the trigger (serialises, lowers throughput); a dedupe gate on `lastModifiedDate` or `version` (too coarse and it drops a real follow-up change); treating "already happened" as success (hides real errors). Pick deliberately and monitor.
- Gate on fields of the entity, not on the outputs of skipped branches.
- Never run two systems on the same webhook (for example the old and the new flow platform during a migration): switch the URL, deactivate the old flow, observe a few days, then delete the old flow **and** its registration.

### Schedule as the safety net

Events get lost (deactivated hook, outage on your side). Run a low-frequency **delta sweep** (see above) in addition to the webhook. And alert on silence: "no events for N hours although open candidates exist" is a failure that "no errors" does not show; error workflows that continue on failure never fire. *(wals.pro practice)*

## Polling you cannot avoid

- Filter and project, never unfiltered; `lastModifiedDate-ge` or a status filter, not a full list.
- Offset schedules (not all at minute :00).
- Debounce alert chains (one alert per fingerprint and day, not one per run).
- UIs that poll multiply with the number of open screens: lengthen the interval, pause hidden tabs, share one cache. *(wals.pro practice)*
