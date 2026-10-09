# Data types: dates, decimals, IDs, custom attributes

Source labels: **weclapp docs** (section "JSON", "Filter Expressions: Type Coercion", custom attribute filters), **wals.pro practice**. Base URL: `https://<tenant>.weclapp.com/webapp/api/v2/`.

## JSON basics *(weclapp docs)*

| Type | On the wire |
|---|---|
| String | Empty or whitespace-only strings are always read as `null`; a property cannot hold an empty string. |
| Decimal (quantities, prices) | JSON **string**, "." as decimal mark, e.g. `"19.90"`. |
| Integer | JSON number. |
| Float/double | JSON number. |
| Date and timestamp | **Milliseconds since 1970-01-01T00:00:00Z** as a JSON number. |
| Enum | JSON string with the constant name. |
| Missing nullable property | Read as `null`. Nulls are omitted in responses unless you send `serializeNulls=true`. |

Parsing of incoming data is lenient (a number where a string is expected is used as the string), which hides type mistakes; do not rely on it.

## Dates and time zones

Fields such as `orderDate`, `deliveryDate`, `plannedDeliveryDate`, `dueDate`, `invoiceDate`, validity ranges, performance record dates and custom attribute date values are epoch milliseconds. A **pure calendar date** (10 July 2026) is the instant of **Berlin midnight** that day: `2026-07-09T22:00:00Z` in summer, `…T23:00:00Z` in winter. *(wals.pro practice, verified against live data)*

Consequences, all silent:

- Rendering or bucketing such a value in UTC shows the **day before**. This shows up in chat messages, spreadsheets, file names and month boundaries.
- Writing `Date.UTC(y, m, d)` stores the wrong instant for a date.
- A fixed offset (`ms - 2 * 3600 * 1000`) is wrong for half the year.
- In n8n, `settings.timezone` affects only n8n's own expressions and schedule triggers. A plain `new Date()`, `toLocaleDateString()` or `toISOString()` inside a Code node runs in the container's time zone (usually UTC). Always pass `{ timeZone: 'Europe/Berlin' }`.

Rules:

1. **Read:** `new Date(ms).toLocaleDateString('sv-SE', { timeZone: 'Europe/Berlin' })` gives `YYYY-MM-DD`. Never `toISOString().slice(0, 10)` or `getUTC*`.
2. **Write a date:** compute the epoch of Berlin midnight for that calendar day (helper below).
3. **Filter:** suffix filters take epoch-ms **numbers** (`createdDate-gt=1398436281262`). Use half-open intervals at Berlin midnight: `orderDate-ge=<start-of-day-ms>&orderDate-lt=<start-of-next-day-ms>`. In a `filter=` **expression**, a string literal compared to a date is an ISO-8601 point in time **with a required time zone** (`2024-10-13T10:39:12+02:00`, `…Z`), and an integer literal is epoch-ms. *(weclapp docs)* Do not put ISO strings into suffix filters; the result is wrong or empty without an error. *(wals.pro practice)*

### DST-safe helper

The offset has to be taken **at the Berlin midnight in question**, not at midday: on 29 March 2026 the clocks jump at 02:00, so midnight is still +01:00 while midday is already +02:00; on 25 October 2026 the opposite happens. The helper tests the two possible instants and keeps the one that actually reads 00:00:00 in Berlin.

```js
const fmt = new Intl.DateTimeFormat('en-CA', {
  timeZone: 'Europe/Berlin', hourCycle: 'h23',
  year: 'numeric', month: '2-digit', day: '2-digit',
  hour: '2-digit', minute: '2-digit', second: '2-digit',
});

function berlinParts(ms) {
  const p = Object.fromEntries(fmt.formatToParts(new Date(ms)).map((x) => [x.type, x.value]));
  return { date: `${p.year}-${p.month}-${p.day}`, time: `${p.hour}:${p.minute}:${p.second}` };
}

/** 'YYYY-MM-DD' -> epoch ms of Berlin 00:00:00 on that day. */
export function berlinMidnightMs(isoDate) {
  const [y, m, d] = isoDate.split('-').map(Number);
  const utcMidnight = Date.UTC(y, m - 1, d);
  for (const offsetHours of [1, 2]) { // CET or CEST
    const candidate = utcMidnight - offsetHours * 3_600_000;
    const { date, time } = berlinParts(candidate);
    if (date === isoDate && time === '00:00:00') return candidate;
  }
  throw new Error(`No Berlin midnight for ${isoDate}`);
}

/** epoch ms -> 'YYYY-MM-DD' calendar day in Berlin. */
export const berlinDay = (ms) => berlinParts(ms).date;
```

Test it before you trust it; the transition days are the ones that break:

```js
import assert from 'node:assert/strict';
const iso = (ms) => new Date(ms).toISOString();
const cases = [
  ['2026-03-28', '2026-03-27T23:00:00.000Z'],
  ['2026-03-29', '2026-03-28T23:00:00.000Z'], // DST starts this day, midnight still +01:00
  ['2026-03-30', '2026-03-29T22:00:00.000Z'],
  ['2026-07-10', '2026-07-09T22:00:00.000Z'],
  ['2026-10-25', '2026-10-24T22:00:00.000Z'], // DST ends this day, midnight still +02:00
  ['2026-10-26', '2026-10-25T23:00:00.000Z'],
];
for (const [day, expected] of cases) {
  assert.equal(iso(berlinMidnightMs(day)), expected);
  assert.equal(berlinDay(berlinMidnightMs(day)), day);
}
```

Python (`zoneinfo` resolves the offset at the local time itself):

```python
from datetime import datetime
from zoneinfo import ZoneInfo

BERLIN = ZoneInfo("Europe/Berlin")

def berlin_midnight_ms(y: int, m: int, d: int) -> int:
    return int(datetime(y, m, d, tzinfo=BERLIN).timestamp() * 1000)

def berlin_day(ms: int):
    return datetime.fromtimestamp(ms / 1000, BERLIN).date()
```

Timestamps such as `createdDate` and `lastModifiedDate` are real instants; treat them as instants and convert to Berlin only for display or day bucketing. For monthly or daily totals, bucket in Berlin time, not UTC, or entries land in the previous period. *(wals.pro practice)*

## Decimals

- Send decimals as **strings with a dot**: `"19.90"`, not `19.9` and not `"19,90"`.
- Round before sending. A float such as `155.68998629999996` is rejected as invalid; round to the field's scale (two places for money) with `Math.round((v + Number.EPSILON) * 100) / 100` and format as a string. The accepted pattern allows up to 13 integer digits and 5 decimals.
- Never sum amounts of different currencies; prefer the `*InCompanyCurrency` fields where they exist. Do not use floats for sums: use integer cents or a decimal type.
- Do not send empty strings to clear a value; send `null`, or omit the property when using `ignoreMissingProperties` semantics.

*(wals.pro practice; JSON representation from weclapp docs)*

## IDs

IDs are JSON **strings**. Keep them as strings end to end; do not convert to JS numbers. Compare with `String(a) === String(b)`: some fields return an ID as string in one place and number in another. In filter expressions, ID properties accept both integer and string literals. *(weclapp docs, wals.pro practice)*

Reference IDs for tenant data (currencies, units, tax rates, payment methods) differ **per tenant**: never hard-code the ID you saw on your test tenant. Read them from the target tenant. With the wals.pro AI connection, `get_reference_data` returns them; otherwise query the entity. *(wals.pro practice)*

## Custom attributes (Zusatzfelder)

- Definitions live in `customAttributeDefinition` (key, label, type, read-only flag). **Fetch them once per client lifetime and cache them**; the weclapp docs name this as a load recommendation. Entity records carry `attributeDefinitionId` plus the value field, not the attribute key, so resolve the definition to get the key and type. Flattening values without the definition lookup silently produces nothing.
- The value goes into a **typed field**; the wrong field is ignored without an error *(wals.pro practice; confirm names in the `customAttribute` schema of your spec)*:

| Attribute type | Value field |
|---|---|
| Boolean | `booleanValue` |
| Date | `dateValue` (epoch ms, Berlin midnight for a pure date) |
| Decimal, integer | `numberValue` (decimal as string) |
| Single entity | `entityId` |
| Reference | `entityReferences[{ entityId, entityName }]` |
| List | `selectedValueId` |
| Multiselect list | `selectedValues[{ id }]` |
| String, large text, URL | `stringValue` |

- Writing is a round trip: GET the entity, find the entry by `String(attributeDefinitionId)`, set the typed field, PUT the entity back (with `version`). Only definitions that are not read-only are writable.
- Filtering depends on the type *(weclapp docs)*:

```text
GET /party?customAttribute3387.value-eq=OPTION1
GET /party?customAttribute4587.entityReferences.entityId-eq=1234
```

  The number is the **definition ID** of your tenant (the same attribute has a different ID elsewhere). Check the custom attribute section of the spec before choosing `.value`, `.entityReferences.entityId`, `.id` or direct equality, and prove the filter with `/count` ([reading.md](reading.md)).
- Boolean custom attributes work well as **state flags** for event-driven flows ("label created", "synced"): visible to users, resettable by hand, no extra database. See [sync-and-webhooks.md](sync-and-webhooks.md).
