# Specification and API versions

Source labels: **weclapp docs** = user documentation inside the OpenAPI YAML; **wals.pro practice** = observed in production integrations (spec snapshot of 24.09.2026, verify against your own spec).

## Two specifications, two purposes

| Spec | URL | Contains |
|---|---|---|
| Public spec with user docs | `https://www.weclapp.com/api/openapi_v2.yaml` (also `openapi_v1.yaml`, `openapi_v3.yaml`) | The documented, versioned surface. Same for everyone. Ignores `includeHidden`. |
| Tenant spec | `GET https://<tenant>.weclapp.com/webapp/api/v2/meta/openapi.yaml` (JSON: `meta/openapi.json`) | What your tenant actually serves. Authenticated. With `?includeHidden=true` it also lists endpoints that the public spec leaves out. |

The tenant spec depends on the licence and modules of that tenant: an endpoint missing there can still exist elsewhere, and an endpoint present there may answer 403 for your user (module or right missing, not "endpoint missing"). *(wals.pro practice)*

Fetch the tenant spec for the version you call (swap `v2` for `v1` or `v3` in the path) and keep it next to your code:

```bash
curl -sS --compressed \
  -H "AuthenticationToken: <token>" \
  -H "User-Agent: <your-integration>/1.0" \
  -o tenant-openapi.yaml \
  "https://<tenant>.weclapp.com/webapp/api/v2/meta/openapi.yaml?includeHidden=true"

curl -sS --compressed -O https://www.weclapp.com/api/openapi_v2.yaml
```

Rules:

- **Pull a fresh spec when you start a task and when something "should exist but 404s".** A local copy is a snapshot; fields and endpoints appear and disappear between releases. *(wals.pro practice)*
- **Use a test tenant's token** to fetch it. The spec is large (several MB): search it, do not paste it into a prompt.
- Do not guess where an endpoint is documented: look it up (below).

## Find the endpoint and the prose docs

Endpoints and schemas:

```bash
rg -n '^  /salesOrder:' tenant-openapi.yaml            # path entry
rg -n 'includeReferencedEntities|name: additionalProperties' tenant-openapi.yaml
rg -n '^    salesOrder:' tenant-openapi.yaml           # schema (components)
```

Prose documentation (the user docs live in `info.description`, headings are indented by four spaces):

```bash
rg -n '^    ## ' tenant-openapi.yaml
```

Read from the matching heading to the next `## ` heading. Typical sections: *Load management*, *Pagination*, *Filtering*, *Filter Expressions*, *Return only specific properties*, *Referenced entities*, *Additional properties*, *Dry-Run*, *Optimistic locking*, *Update a specific instance*, *Error reference*. Do not hard-code line ranges; they shift with every spec update. *(weclapp docs structure; wals.pro practice for the search method)*

List what the tenant offers beyond the public spec (needs PyYAML; the files are big, so this takes a moment):

```python
import yaml

def paths(file):
    with open(file, encoding="utf-8") as f:
        return set(yaml.safe_load(f)["paths"])

extra = sorted(paths("tenant-openapi.yaml") - paths("openapi_v2.yaml"))
print(len(extra), "paths only in the tenant spec")
for path in extra:
    print(path)
```

Operations whose public counterpart is missing are the candidates for [unofficial-endpoints.md](unofficial-endpoints.md).

## Versions

| Version | Use it when | Notes |
|---|---|---|
| **v2** | Default for every new integration. | Current standard version. weclapp docs: improved POST/PUT, consistent custom attributes, cleaned-up property names, better error status codes, obsolete endpoints removed. |
| **v1** | Only to keep existing code running until you migrate. | The v1 spec carries a deprecation notice: removal no earlier than August 2026, and an "earliest end-of-life" line naming November 2026. That is what weclapp's v1 spec says; confirm the actual date with weclapp before planning around it. |
| **v3** | Deliberately, for a capability v2 cannot express. | Few path changes, bigger schema changes. Reads and writes may be split across versions. |

Facts that decide a migration or a version choice *(wals.pro practice, from a comparison of the public v1, v2 and v3 specs on 24.09.2026; re-check in your spec)*:

- **v2 knows only `/party`.** `/customer`, `/supplier`, `/lead` and `/contact` exist in v1 and return 404 in v2. Customers, suppliers, leads and contacts are `party` records told apart by flags such as `customer`, `customerNumber`, `supplierNumber`, `partyType`.
- **Denormalised `*Name` and `*Number` fields of v1 are gone in v2.** Read the reference ID and resolve it with `includeReferencedEntities`.
- **Item relation names differ** (for example `salesInvoiceItemRelationship` in v1 against `salesInvoiceItemRelationships` in v2). A matcher that "finds nothing" raises no error.
- **v3 changes shipments and goods receipt:** the single-package fields on `shipment` are removed in favour of `parcels`, incoming bookings move to a `bookIntoWarehouse` action, and `paid` becomes `paidStatus` on orders. Some write-only capabilities, such as the component factor on production order items, exist only in v3.
- Choose the version **per workflow, deliberately.** Never copy a base URL from a neighbouring node or script. Debug against the same version production uses.

Deprecation signals:

- `deprecated: true` on an operation in the spec is the **only** signal. weclapp sends no `Deprecation` or `Sunset` response headers and does not always name a successor or a date. *(wals.pro practice)*
- weclapp writes new major versions in parallel and keeps the previous version available for at least one year after the release of a new one. *(weclapp docs, "API lifecycle")*
- Subscribe to the API newsletter linked in the spec and read the changelog before upgrading. Changes are mostly additive; breaking changes are documented in the changelog. *(weclapp docs)*
- weclapp may change attributes, resources and policies without advance notice and is not liable for the effects (spec section "Change Policy"). Design for that: tolerate unknown fields, assert what you depend on, monitor failures.
