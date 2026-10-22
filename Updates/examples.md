# Search-Agent demonstration

**Design proposal · 2026-10-22.** Tenant: `toyota`. This demonstration uses only the search and visitor identifiers (`searchId` and `visitorId` in the original use case). Their API labels are `search` and `visitor`. All requests below use the revised contract.

## 1. Create a search with its policy

The Search Agent mints `search-123`; its visitor identity integration provides `visitor-42`. It sends `POST /cfs/v1/contexts`, header `X-Tenant-Id: toyota`, and a JSON body:

```json
{
  "refLabel": "search",
  "refId": "search-123",
  "ltm": { "visitor": "visitor-42" },
  "policy": {
    "stm": { "cache": { "ttl": "PT24H" }, "durability": { "ttl": "P30D" } },
    "ltm": {
      "search": { "ttl": "P30D" },
      "visitor": { "ttl": "P365D", "windows": ["P7D", "P30D", "P90D"] }
    }
  },
  "data": {
    "eventType": "searchPerformed",
    "query": "red SUV under 30k near Austin",
    "resultCount": 24
  }
}
```

`201` means the context was created. CFS stores one Policy row and one Context row, durably records the event, and updates the optional shared cache. Repeating the POST can append another event; the current contract does not suppress duplicate client submissions.

## 2. A second app contributes and reads

Another authorized app (`foobar` in the original example) sends the same route/tenant and:

```json
{
  "refLabel": "search",
  "refId": "search-123",
  "data": { "eventType": "priceAlertSet", "priceMax": 30000 }
}
```

`200` means an append to the existing context. The policy and bindings already apply. Both apps read both events with:

```http
GET /cfs/v1/contexts?refLabel=search&refId=search-123&limit=5
```

## 3. A returning shopper starts another search

The Search Agent creates `search-456` with the same policy and `ltm` value but a new query. This creates a second context for the same visitor. A third search, `search-789`, belongs to `visitor-99` and remains separate.

| Summary scope | Contributing searches |
| --- | --- |
| `search / search-123` | First search, both apps |
| `visitor / visitor-42` | Searches 123 and 456 |
| `visitor / visitor-99` | Search 789 only |

After 30 minutes without writes, the Sweeper can select each context on its next five-minute run and queue it through SQS. The six-hour backstop also allows a continuously active context with pending work to be processed. The Extractor reads all writers' events, generates search/visitor tags, retains them in Hindsight, publishes summaries, and records its checkpoint. The demo waits for summary readiness; immediate LTM results are not promised. See the [worker sequence](overview.md#6-asynchronous-ltm-processing).

## 4. Read search and return-shopper memory

```http
# Shopper: this visitor's preceding 30 days
GET /cfs/v1/summaries?refLabel=visitor&refId=visitor-42&window=P30D

# Either authorized app: this search's overall summary
GET /cfs/v1/summaries?refLabel=search&refId=search-123

# Shopper: relevant visitor facts
GET /cfs/v1/memories?refLabel=visitor&refId=visitor-42&q=budget%20and%20body%20style&limit=3
```

The search summary combines events from both apps. The visitor summary combines eligible evidence across that visitor's searches. App names are explanatory and are not fields in these requests.

## 5. Demonstrate the rules

Changing `visitor-42` to another visitor fails with `409 LTM_KEY_CONFLICT`; an unknown key fails with `400 UNKNOWN_LTM_KEY`; a different policy fails with `409 POLICY_CONFLICT`. None creates a parallel context.

Finally, `DELETE /cfs/v1/contexts?refLabel=visitor&refId=visitor-42` returns `202 PENDING`; repeat until `200 DONE`. Both linked searches and affected search/visitor summaries are erased or invalidated. The unrelated visitor-99 search remains. This is a demo acceptance scenario, subject to the lifecycle validation in the [Data-Model](data-model.md).
