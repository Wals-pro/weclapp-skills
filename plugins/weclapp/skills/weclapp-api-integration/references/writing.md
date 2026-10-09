# Writing: PUT, items, versions, retries, dry run, actions, statuses

Source labels: **weclapp docs** (sections "Update a specific instance", "Optimistic locking", "Dry-Run", "Create a new instance", "Error reference"), **Lanig talk**, **wals.pro practice**. Base URL: `https://<tenant>.weclapp.com/webapp/api/v2/`, header `AuthenticationToken: <token>`. Run every example against a test tenant first.

## Create

`POST /<entity>` with the writable fields only. Omit `id`, `version`, `createdDate`, `lastModifiedDate`. Read-only and unknown fields give 400, missing rights give 403; success is 201 with a `Location` header. *(weclapp docs)*

## PUT replaces the entity

A plain PUT interprets every property you leave out as `null`. Two safe ways *(weclapp docs)*:

1. **Full read-modify-write:** `GET /<entity>/id/<id>`, change the JSON, PUT the whole object back. New properties that your client does not know yet survive because they travel back unchanged. Writable booleans must be present in a full update.
2. **Partial update:** `PUT /<entity>/id/<id>?ignoreMissingProperties=true` with only the fields you change, **plus `version`**.

```http
PUT /party/id/7992?ignoreMissingProperties=true
Content-Type: application/json

{ "version": "0", "lastName": "Example-New" }
```

Notes:

- `ignoreMissingProperties` is not declared as a parameter on every operation in the spec. A generated client may drop it silently; check the URL actually sent. *(wals.pro practice)*
- If `id` is in the body it must match the URL.
- Properties that cannot be changed because of the entity's status give 403/400. Do not send them "for completeness".
- Send only the minimal payload for what you intend. Fields sent "to be safe" can be re-derived by the server and flip other flags; compare the returned record field by field with the previous state. *(wals.pro practice)*

## Item arrays are replaced, even with `ignoreMissingProperties`

Lists of sub-records (`orderItems`, `shipmentItems`, price lines, `parcels`, and similar): **an existing item you omit is removed or emptied.** `ignoreMissingProperties=true` protects top-level properties, not these lists. *(weclapp docs: "if you omit the id of an existing entry, it will be removed or emptied"; wals.pro practice)*

Treat the update as a merge:

```json
{
  "version": "5",
  "orderItems": [
    { "id": "9001", "version": "3", "quantity": "2" },
    { "id": "9002", "version": "1" },
    { "articleId": "123", "quantity": "1" }
  ]
}
```

- Existing item, unchanged: send `{ "id", "version" }` as a stub.
- Existing item, changed: `id`, `version` and the changed fields.
- New item: no `id`.
- Assert before sending: `existing item IDs − payload item IDs = ∅`, unless removing is the explicit purpose. Remove an item only deliberately.
- Some item fields are guarded (for example manual price flags): sending them back changed gives 400 "cannot be updated". Do not touch them when you only change quantity.
- For `parcels[]` the list is replaced by position; never send `[]` to "keep it".

Lists whose items can be changed but whose list cannot (for example `customAttributes`) behave like normal properties: items you do not send are taken from the existing entity. *(weclapp docs)*

## Optimistic locking: `version`

- With `version` in the body the update fails with an optimistic-lock error if it differs from the current value. Without it the check is off and a concurrent change is overwritten silently. *(weclapp docs)*
- Conflict handling for a side-effect-free update: re-read by ID, reapply **your intended change** to the fresh state, retry a bounded number of times.
- Take the `version` from a **by-ID GET right before the PUT**, not from a list query: lists can return a stale `version` for a short time after a write, which produces the 409 you wanted to avoid. *(wals.pro practice)*
- Recognise the conflict by problem `type` (the part after the last `/` is `optimistic_lock`) or HTTP 409; some paths answer 400. *(weclapp docs: use the last segment of `type`; wals.pro practice)*
- For writes with external side effects (shipping, labels, bookings) do not retry with a re-fetched version blindly; see the next section.

## Read after write, always

- HTTP success is not proof of persistence. Fields can be dropped without an error on some entities, and some combinations are "healed" quietly. Examples seen live *(wals.pro practice)*: a `salesOrderId` in a `salesInvoice` POST is accepted and not stored; BOM lines are discarded when the parent article is not flagged as a production article; a `leadStatus` without the matching customer flag is not kept.
- After each new write path, GET the record and compare the fields you set. Treat constant lists of "supported fields" as claims, not evidence.
- **Eventual consistency:** immediately after a write a follow-up call can return 404 or the old state. Read back with backoff (for example after 0, 2, 5, 10 s) before concluding anything. If a clearing step did not take, repeat it once after the read-back.

## A timeout is not a failure

weclapp queues requests for up to about 30 seconds ([load-management.md](load-management.md)). Your read can time out while the write **commits anyway**. A 409 after a retry can mean "your first attempt worked".

Procedure for any write whose repetition would hurt (create documents, book stock, ship, charge):

1. Classify timeout, connection loss, 409 and optimistic-lock errors as **uncertain**, not as failed.
2. Do not retry automatically.
3. Re-read the record, with backoff such as `0, 2, 5, 10, 20, 30 s`, and check the fields the write was meant to set.
4. If they are there: success (log it as recovered). If they are absent after the window: only now retry or escalate; a very late commit remains possible, so keep the operation marked ambiguous until you can rule it out.

**Idempotency:** the API has no idempotency keys. Make creates idempotent yourself: before a POST, look for an existing record by a stable business key you control (a reference field, a custom attribute, a number you generate) and store the key *before* the call. Check "does it already exist?" first in every handler that can run twice. *(wals.pro practice)*

Automatic retries are for reads. For writes, retry only when the operation is provably repeatable. *(wals.pro practice)*

## dryRun: only for generic writes

`?dryRun=true` on generic `POST`, `PUT` and `DELETE` runs validation and business logic without persisting; success is **200** (not 201) and the response has no `id`, `version` or date meta properties. It does not cover every operation, and "where possible" is the spec's own wording. *(weclapp docs)* It also checks your rights, which makes it a safe probe on a test tenant. *(wals.pro practice)*

```http
POST /salesOrder?dryRun=true
Content-Type: application/json

{ "customerId": "3183" }
```

**It does not protect named actions.** In a live test `POST /salesInvoice/id/<id>/createCreditNote?dryRun=true` answered 200 with an empty body, and a real credit note existed afterwards. *(wals.pro practice)* For actions (`…/id/<id>/create…`, `…/<verb>`) assume `dryRun` is ignored and verify with a read-back on a test tenant.

## Use the native action, do not rebuild it

Linked documents arise only through the action that creates them *(wals.pro practice)*:

- `POST /salesOrder/id/<id>/createSalesInvoice` (not a POST to `/salesInvoice` with an order reference), likewise `createShipment` and similar; the link fields are read-only or ignored when written directly.
- `acceptQuotation` creates a linked order and makes the quotation undeletable.
- Preconditions apply (for example a printed order confirmation); read the action's schema and error text before looping.

If a field "does not exist" on a document, check whether it is a read-only link, a different property name (`recordFreeText`, `recordOpening`, `recordComment` for header texts), or a v3-only field before inventing a workaround.

## Statuses: weclapp does not enforce a graph

- A raw `PUT` with a new `status` is accepted for moves that a user could never make in the UI, including backwards ones such as order confirmation back to draft or a shipped delivery back to new. Only `CANCELLED` was found to be final. *(wals.pro practice)* Do not infer "cannot be changed" from a status name, and do not rely on weclapp to stop an invalid transition: validate the allowed transitions in your own logic.
- Where a status change triggers mail rules or follow-up logic, prefer the action over a raw status PUT; a cancel can fail on an active mailing rule. *(wals.pro practice)*
- Business fields and status in **separate** PUTs: the fields land even if the status change fails validation.
- Include the intermediate statuses in your filters from the start (for example "documents printed" between draft and confirmed on purchase orders), or open items disappear from your list. *(wals.pro practice)*

## Error handling cheat sheet

| Signal | Meaning | Reaction |
|---|---|---|
| 400, `type` ends `validation` | Payload invalid; `validationErrors[].location` is the JSON path | Fix the payload; no blind retry |
| 400 with empty `validationErrors` ("invalid data") | Often a tenant configuration gap (missing default, referenced ID, tax setup) | Log submitted keys; check tenant settings; ask the customer |
| 400, `type` ends `request_timeout` | Server-side request timeout | Retry only if safe; shorten or split the work |
| 401 / 403 | Token invalid, or right/module missing | Not a missing endpoint; check user rights and licence |
| 404 right after a write | Eventual consistency, or truly absent | Read back with backoff first |
| 409 / `optimistic_lock` | Stale `version` (or uncertain commit) | See above |
| 429 | Queue exhausted | Backoff, lower concurrency |

Evaluate `type` by its last path segment, not by message text. *(weclapp docs)*
