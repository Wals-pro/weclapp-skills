# Reading: filters, projection, joins, pagination, batching

Source labels: **weclapp docs** (sections "Filtering", "Filter Expressions", "Return only specific properties", "Referenced entities", "Additional properties", "Pagination", "Counting"), **Lanig talk**, **wals.pro practice**. Base URL in all examples: `https://<tenant>.weclapp.com/webapp/api/v2/`; every request carries `AuthenticationToken: <token>`.

## Filters: two syntaxes, never mixed

**1. Suffix parameters**: property name, minus, operator.

```text
GET /salesOrder?orderDate-ge=1790805600000&orderDate-lt=1793487600000&properties=id,orderNumber
GET /party?customerNumber-in=["1006","1007"]&properties=id,customerNumber
```

The two epoch values above are Berlin midnight of 1 Oct and 1 Nov 2026 (half-open interval, see [data-types.md](data-types.md)). Date suffix filters take epoch milliseconds as numbers.

Operators: `eq ne lt gt le ge null notnull like notlike ilike notilike in notin`. Several parameters are ANDed; prefix a name with `or-` for OR, or use named groups (`orGroup1-…`, `orGroup2-…`) to AND OR-groups. `like` supports `%` and `_`. `in` takes a **JSON array** as the value. `null` and `notnull` ignore the value. *(weclapp docs)*

**2. Filter expressions** (`filter=…`, marked beta by weclapp): one expression with its own grammar.

```text
GET /party?filter=(salesChannel in ["NET1","NET4"]) and (partyType = "ORGANIZATION")
GET /party?filter=(customerSalesChannel = "NET1") and (not (customerNumber null))
```

`=, !=, <, >, <=, >=, ~` (like), `in [..]`, `contains`, `empty`, `null` (postfix), `and or not`, `trim lower length`, `a ? b : c`, list predicates such as `values[locale = "de"].text = "x"` (no nested predicates). Several `filter` parameters are ANDed with each other and with suffix parameters. *(weclapp docs)*

Do not mix the grammars:

- `filter=(customerNumber-notnull)` and `filter=(customerNumber != null)` fail; the expression form is `not (customerNumber null)`. A suffix filter and an expression are different languages. *(wals.pro practice, verified on a test tenant)*
- **Encoding:** build the filter string unencoded and let the HTTP client encode it once (`curl --get --data-urlencode 'filter=…'`, a query-parameters object). Pre-encoded URLs get encoded twice. *(wals.pro practice)*

## Silent failures: prove every filter

- **Unknown filter properties and properties that do not support filtering are silently ignored**: HTTP 200 with the unfiltered set. *(weclapp docs)* A typo or a wrong suffix (`-gte`, `-lte` do not exist; it is `-ge`, `-le`) turns a filtered read into a full read. In a paging loop that is the whole table. *(wals.pro practice)*
- A filter on the wrong field can also return **zero** rows instead of everything: `customerId` holds the internal ID, not the customer number; the ticket status field is `ticketStatusId`, not `status`. *(wals.pro practice)*
- By contrast `properties` and `sort` with an unknown name return an **error**. Use that to validate field names.

Prove a filter before you trust it:

1. `GET /<entity>/count?<same filters>` and compare with what you expect.
2. Move the boundary (one value above and one below) and check that the count changes accordingly.
3. Check a sample of returned rows against the condition in code. For critical sums, filter again client-side as a guard.

When a "no results" case looks odd, suspect the filter before the data.

## properties: project what you use

```text
GET /party?properties=id,customerNumber,customerSalesChannel,contacts.id,contacts.lastName
```

- Always send `properties`. List every field a later step needs, with dot paths for nested lists.
- An unknown property in `properties` fails the whole request: good during development, a trap if a field disappears between releases. Check projections against your current spec. Writable does not imply readable, and the reverse. *(weclapp docs, wals.pro practice)*
- Not for downloads, `/count` or metadata endpoints.

## Referenced entities instead of N+1

Include by reference ID, project with a colon:

```text
GET /article?includeReferencedEntities=unitId,articleCategoryId&properties=id,articleNumber,unitId,unit:id,unit:name
```

```json
{
  "result": [{ "id": "1", "articleNumber": "A-1", "unitId": "7" }],
  "referencedEntities": { "unit": [{ "id": "7", "name": "Piece" }] }
}
```

- The include parameter names the ID property (`unitId`); the projection uses the referenced entity name plus colon (`unit:name`), **not** `unit.name`. *(weclapp docs)*
- Referenced records are in the top-level `referencedEntities`, grouped by entity type, not inside each row. Join by `id`; take `unit:id` along. *(weclapp docs, wals.pro practice)*
- Several ID properties can land in one bucket (customer, invoice recipient and similar references all resolve to `party`). Resolve by ID across the bucket rather than by property name. Not every collection field can be included; for those, collect the IDs and fetch them once with `id-in`. *(wals.pro practice)*
- This replaces "one call per row to resolve a name" and is one of the most effective load reductions. *(Lanig talk)*

## additionalProperties: computed, index-aligned

```text
GET /article?properties=id,articleNumber&additionalProperties=currentSalesPrice
```

- Computed or complex optional values are returned in a top-level `additionalProperties` object. Each entry is an array whose **index matches the index in `result`**. *(weclapp docs)*
- Merge per page, together with that page; never pool arrays across pages or after a re-sort. *(wals.pro practice)*
- Only a minority of list endpoints declare any, and the names are listed per endpoint in the spec. Some depend on a specific right and answer 403 without it, which is a missing right, not a missing endpoint. *(weclapp docs, wals.pro practice)*
- Do not confuse the three mechanisms: `properties` (fields in `result`), `includeReferencedEntities` (side-loaded records), `additionalProperties` (computed values in their own block).

## Pagination, sorting, counting

- Default page size is 100, usually capped at 1000; exceptions are noted per resource. `page` is one-based; `offset` also exists. *(weclapp docs)*
- Use the largest page size the resource allows together with a tight projection.
- **Stop when a page returns fewer rows than `pageSize`**, not by comparing with `count`. Some endpoints report a `count` that does not match the list. *(wals.pro practice)*
- Offset paging is **not a snapshot.** If records change between pages, rows move across the boundary and you get gaps or duplicates without an error. Sort by a stable field (`sort=id`), detect duplicate IDs across pages, and for reports use keyset paging: `sort=id&pageSize=1000&id-gt=<last id>`. Handle IDs as strings. *(wals.pro practice)*
- Sorting: `sort=name`, `sort=-createdDate`, comma-separated for several; an unknown sort property is an error. `orderBy` is beta; never combine it with `sort`. *(weclapp docs)*
- `GET /<entity>/count` takes the same filters and is cheap. Use it to size a job, to decide whether to read at all, and to validate filters. *(weclapp docs, wals.pro practice)*
- There is no aggregation endpoint beyond count. Build sums with a thin scan (narrow projection, budget check against `count` first), sum per currency, and compare against the system's own totals. *(wals.pro practice)*

## Client-side batching: IDs first, then `id-in`

For "give me these 3,000 records":

1. Collect the IDs (from a narrow first query, a webhook, your own table).
2. Fetch them in chunks with `id-in`, projected:

```text
GET /warehouseStock?articleId-in=["101","102","103"]&properties=id,articleId,quantity&pageSize=1000
```

3. Process each chunk, merge, and keep going on partial failure.

Rules *(wals.pro practice)*:

- The weclapp docs recommend fewer, larger requests such as `filter=id in […]`, but document no limit for URL length. Keep chunks conservative (for example 100 IDs per URL) or move long lists into the request body of `POST /<entity>/query`, see [unofficial-endpoints.md](unofficial-endpoints.md).
- One chunk request replaces dozens of single GETs; it also makes the load predictable.
- Set the chunk size by measured response time, not by habit.

## Entity shapes that differ

- Not every response is `{ "result": [ … ] }`. Some by-ID endpoints return the entity at the top level, and singleton paths such as the current user return `result` as an object. Check the response schema in the spec before writing a generic client. *(wals.pro practice)*
- `serializeNulls=true` adds null properties to the response; by default nulls are omitted. *(weclapp docs)*
