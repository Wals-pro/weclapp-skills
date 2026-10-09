# Unofficial endpoints

Source labels: **weclapp docs** (public spec and its "Change Policy"), **wals.pro practice**. This file describes what you can see for yourself in the spec of your own tenant. It does not cover non-public partner material.

## What "unofficial" means here

An endpoint is unofficial when it is **absent from the public spec of the same API version** (`https://www.weclapp.com/api/openapi_v2.yaml`) but present in your tenant spec fetched with `?includeHidden=true`.

- weclapp makes **no stability promise** for it: no versioning guarantee, no announcement before it changes, and (per the spec's "Change Policy") no liability for the effects. Even the public API can change without advance notice; unofficial endpoints have no changelog protection at all.
- It is **not the same as `deprecated: true`.** Deprecated operations are official and flagged for removal; unofficial operations are not part of the documented contract.
- Documentation can run behind: something may be described in the user docs prose and still be missing from the operation list.
- Availability depends on licence and rights. A 403 means a module or right is missing, not that the path does not exist; a 404 can mean the path is gone in this version. *(wals.pro practice)*

## How to find them in your tenant spec

1. Fetch both specs ([spec-and-versions.md](spec-and-versions.md)): public `openapi_v2.yaml` and the tenant spec with `includeHidden=true`.
2. Diff the **paths and methods** (the Python snippet in `spec-and-versions.md` lists paths that exist only in the tenant spec; extend it to compare `(path, method)` pairs).
3. For each candidate read the operation in the tenant spec: parameters, request schema, response schema, `deprecated`, any description. If the spec says nothing about semantics, you have a signature, not a contract.
4. Probe on a **test tenant** only. Read-only calls first.

Result of such a diff, as of the 24.09.2026 snapshot *(wals.pro practice, re-check on your tenant)*: the bulk is `POST /<entity>/query` and `POST /<entity>/count` for every entity, plus read-only list endpoints for item entities (for example sales order items) and assorted tenant-specific modules.

## `POST /<entity>/query` and `POST /<entity>/count`

The spec prose itself describes the pattern: "POST queries have the advantage to offer greater flexibility, as they have fewer restrictions compared to GET requests", and recommends filter expressions in the request body. The operations are nevertheless not listed per entity in the public spec. *(weclapp docs)*

Why it is useful:

- no URL length limit for long `in` lists;
- no URL encoding of the filter expression;
- the same response shape as the corresponding GET list or count.

Body shape, taken from the request schema in the spec (not verified by us operation by operation, so test it first):

| Field | Notes |
|---|---|
| `filter` | **One** filter-expression string (no suffix filters, no `or-` groups in the body) |
| `properties` | Array of strings, not a comma-separated string |
| `includeReferencedEntities` | Array of ID property names |
| `additionalProperties` | Array of names |
| `orderBy` | Array of ordering expressions; there is no `sort` field |
| `page`, `pageSize`, `offset` | As in the GET list |
| `serializeNulls` | Boolean |

The count variant takes `filter` only. A few entities (comments, documents, archived emails) additionally take `entityId` and `entityName` to scope the query to a record.

```http
POST /party/query
Content-Type: application/json
AuthenticationToken: <token>

{
  "filter": "customerNumber in [\"1006\",\"1007\"]",
  "properties": ["id", "customerNumber"],
  "pageSize": 1000
}
```

```http
POST /party/count
Content-Type: application/json
AuthenticationToken: <token>

{ "filter": "customerNumber in [\"1006\",\"1007\"]" }
```

All the filter pitfalls of [reading.md](reading.md) apply: prove the filter with a count and a boundary check.

## Rules for using any unofficial endpoint

1. **Decide knowingly and write it down.** One comment or decision record per use: why no official route exists, spec version and date, who owns the decision.
2. **Prefer the official route** whenever one gives the same result at acceptable cost. Unofficial is for what official cannot do, or for a measurable load or length problem.
3. **Isolate it** behind a single function or adapter, so a break is a one-place fix.
4. **Have a fallback.** For `POST /<entity>/query` the fallback is the GET list with `id-in` chunks; for others, degrade the feature, do not fail the whole flow.
5. **Monitor it.** Alert on 404, on unexpected 400s and on response-shape changes. A contract test (a handful of calls against a test tenant, run on a schedule and after weclapp releases) catches drift before customers do.
6. **Read-only first.** Mass operations, writes, and anything that acts on a plain GET need explicit sign-off, a test-tenant rehearsal and, for bulk changes, silenced live sync. Do not wire writes you cannot undo.
7. **Never a customer-facing promise.** Do not sell or document a feature as "supported by weclapp" when it rests on an unofficial endpoint.
8. **Business-critical dependency? Ask weclapp** for the intended status of that endpoint before you build on it.
9. Treat the tenant spec as a snapshot too: re-diff after weclapp releases.
