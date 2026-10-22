# CFS API

**Engineering reference · contract draft · 2026-10-22.** [Overview](overview.md) is the presentation entry point. [OpenAPI 3.1.1](openapi.yaml) is the machine-readable contract; [Data-Model](data-model.md) explains persistence and processing. The contract accepts business-defined labels; wire examples use search and visitor identities from the [demonstration](examples.md).

## 1. Conventions and identity

The base URL is `/cfs/v1`. Every operation requires AWS IAM/SigV4 authentication; all except health also require `X-Tenant-Id`. CFS authorizes the authenticated principal for the tenant, subject, and operation. Policy creation and erasure require their own permissions. No `appId` header, body field, or writer selector is used.

| Name | Meaning |
| --- | --- |
| `refLabel` | Business subject label, such as `search` or `visitor`. |
| `refId` | Actual identifier value, such as `search-123` or `visitor-42`. |
| Request `ltm` | Additional labels mapped to actual ID values. |
| `policy.ltm` | Labels mapped to immutable retention/summary rules. |
| Internal `ctxId` | Deterministic hash of the tenant/label/value tuple, computed by CFS; never a public input. [Derivation](data-model.md#ctxid-derivation-and-creation). |

A context is unique by `(tenantId, refLabel, refId)`. Linked-identity reads use the same label/value vocabulary but do not require a standalone Context row. Unknown request fields are rejected; application event fields belong inside `data`. Access logs must avoid business subject/ID values, query text, and payload values; correlation request/trace IDs remain usable.

Linked-identity aggregation is provided by `/summaries` and `/memories`. `/contexts` reads one directly keyed raw history; it does not fan out across contexts associated with a visitor.

Success is `{data, meta}`; `meta` contains `traceId` and a response `timestamp`. CFS errors are `{error, meta}`. Gateway-generated errors may instead contain a native `message` body. Limits, retry rules, and lifecycle choices remain draft contract details, not behavior verified against a deployed endpoint.

## 2. Operations

| Method and path | Purpose | Main success response |
| --- | --- | --- |
| `POST /contexts` | Create with policy or append to shared history | `201` created; `200` appended |
| `GET /contexts` | Read recent live events from all writers | `200`, `data.entries` |
| `GET /policies` | Inspect an exact context's locked policy | `200`, `data.policy` |
| `GET /summaries` | Read one precomputed overall/window summary | `200`, status and items |
| `GET /memories` | Recall relevant facts/observations | `200`, `data.memories` |
| `DELETE /contexts` | Erase a direct/linked identity and poll completion | `202 PENDING`; `200 DONE` |
| `GET /health` | Check process readiness | `200 OK` |

Paths in the table are relative to `/cfs/v1`. Policy mutation routes do not exist.

## 3. POST /contexts — create or append

Send `X-Tenant-Id` and a JSON body, with SigV4 signing. `Content-Type: application/json` is the standard HTTP header that tells CFS the body is JSON. HTTP clients commonly set it when their JSON-body option is used; it is not an additional CFS business field.

| Body field | Required | Meaning |
| --- | --- | --- |
| `refLabel`, `refId` | Yes | Select the shared business context. |
| `data` | Yes | JSON object representing one event; at most 65,536 encoded UTF-8 bytes. |
| `occurredAt` | No | Source event timestamp; CFS separately assigns `recordedAt`. |
| `ltm` | No | Additional identity values; labels must be permitted by the locked policy and differ from `refLabel`. |
| `policy` | On creation | Complete policy. May be omitted for an existing context; a repeated policy must match. |

Full request limit: 131,072 UTF-8 bytes. Labels follow `^[a-z][A-Za-z0-9]{0,63}$`; identifier values contain 1–128 characters from letters, digits, `.`, `_`, `:`, and `-`. The selected subject's own LTM value comes from `refLabel`/`refId`, so it is not repeated in request `ltm`.

Request `ltm` may contain several distinct permitted labels, each mapped to one ID; arrays of IDs under one label are not supported. Identity-looking fields inside `data` are ordinary event content and establish no binding. Only the top-level `ltm` map supplies additional bindings. A context may start with unknown additional identities and bind them later under its already configured policy.

```http
POST /cfs/v1/contexts
X-Tenant-Id: toyota
Content-Type: application/json
```

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
  "data": { "eventType": "searchPerformed", "query": "red SUV under 30k near Austin" }
}
```

A new context returns `201` with:

```json
{
  "data": {
    "entryId": "20000000-0000-4000-8000-000000000001",
    "recordedAt": "2026-10-21T16:00:00Z"
  },
  "meta": { "traceId": "trace-demo-001", "timestamp": "2026-10-21T16:00:01Z" }
}
```

Another authorized app sends the same route/tenant with this body:

```json
{
  "refLabel": "search",
  "refId": "search-123",
  "data": { "eventType": "priceAlertSet", "priceMax": 30000 }
}
```

It receives `200` with a new entry ID. The existing policy and visitor binding apply. A supplied different policy causes `409 POLICY_CONFLICT`; a different visitor value causes `409 LTM_KEY_CONFLICT`; an unknown label causes `400 UNKNOWN_LTM_KEY`. None creates an alternate context. Omission preserves established bindings; a permitted unset binding can be added and triggers recoverable backfill.

Policy comparison ignores object order, normalizes durations to seconds, and compares windows as sets. Creation requirements and label/policy relationships depend on server state and are runtime checks, not merely JSON Schema validation.

### Shared context and binding rules

These rules apply to every business label, regardless of which application created the context:

| App B's request | Outcome |
| --- | --- |
| Same tenant, `refLabel`, and `refId`; omit `ltm` | Append to the same context and inherit its bindings. |
| Same tuple; supply an unset binding already permitted by the locked policy | Bind the subject and append to the same context. Earlier recoverable eligible history joins that scope too. |
| Same tuple; supply a label the locked policy does not permit | `400 UNKNOWN_LTM_KEY`; no second context. |
| Same tuple; supply a different policy to enable another label | `409 POLICY_CONFLICT`; the existing policy is not extended or merged. |
| Same tuple; replace an established identity value | `409 LTM_KEY_CONFLICT`. |
| Genuinely different business tuple | A new context, requiring its own creation policy. |

Every writer accepts the existing retention/grouping rules. A writer need not populate or read every configured scope, but cannot apply a private policy to the shared context. Future contexts can use a newly agreed configuration; minting another ID for the same business activity is not an implicit policy-upgrade mechanism.

### Retry behavior

Each successful POST records a new event. Retrying after a timeout or lost response may append another event; duplicate suppression is outside the current contract.

## 4. GET /contexts — shared recent history

```http
GET /cfs/v1/contexts?refLabel=search&refId=search-123&limit=5
X-Tenant-Id: toyota
```

`refLabel` and `refId` are required. `limit` defaults to 20 and allows 1–64. All authorized writers are included, ordered by `recordedAt` descending, then entry ID descending. This is a bounded recent-history API, not paginated full-history export.

Both apps use the same request to read that shared history. This draft provides no per-writer stream or writer filter. Application payloads may have different shapes; consumers interpret their agreed event types. For visitor memory across searches, use `/summaries` or `/memories` with `refLabel=visitor` rather than expecting `/contexts` to aggregate linked searches.

```json
{
  "data": {
    "entries": [
      {
        "entryId": "20000000-0000-4000-8000-000000000001",
        "recordedAt": "2026-10-21T16:00:00Z",
        "data": { "eventType": "searchPerformed", "query": "red SUV under 30k near Austin" }
      }
    ]
  },
  "meta": { "traceId": "trace-demo-002", "timestamp": "2026-10-21T16:01:00Z" }
}
```

Entries optionally include `occurredAt`. Unknown/empty contexts return `entries: []` and create no rows. CFS uses complete Redis cache or DynamoDB fallback, excludes expired events, and returns `503` on dependency failure rather than fabricating empty history.

## 5. GET /policies — inspect the locked rules

```http
GET /cfs/v1/policies?refLabel=search&refId=search-123
X-Tenant-Id: toyota
```

Returns `data.refLabel`, `refId`, complete `policy`, `locked: true`, and `createdAt`. Missing context returns `404 CONTEXT_NOT_FOUND`. This selects one context's policy, not a global policy for every search. Reading visitor summaries does not imply that `visitor / visitor-42` has its own policy row.

## 6. GET /summaries — precomputed LTM

```http
GET /cfs/v1/summaries?refLabel=visitor&refId=visitor-42&window=P30D
X-Tenant-Id: toyota
```

The tenant and subject pair are required. Omit `window` for the overall retained-evidence summary; supply a configured positive whole-day duration for a rolling window. The endpoint reads a materialization; it does not synchronously invoke summary generation.

| Status | `data.items` | Meaning |
| --- | --- | --- |
| `READY` | Exactly one item | Valid requested summary is available. |
| `PENDING` | Empty | Eligible work awaits extraction, repair, or regeneration. |
| `EMPTY` | Empty | Unknown subject, disabled LTM, or no eligible evidence. |

A known subject with eligible evidence but an unsupported window returns `400 UNSUPPORTED_WINDOW`. Backend failure returns `503`, not `EMPTY`. Proposed shared-identity configuration preserves each contribution's own expiry and unions windows across active contributors.

```json
{
  "data": {
    "status": "READY",
    "items": [
      {
        "kind": "WINDOW",
        "window": "P30D",
        "windowStart": "2026-09-21T16:00:00Z",
        "windowEnd": "2026-10-21T16:00:00Z",
        "text": "Recent searches focus on red SUVs under $30,000 near Austin.",
        "asOf": "2026-10-21T16:00:00Z",
        "computedAt": "2026-10-21T16:00:01Z",
        "sourceThrough": "2026-10-21T15:59:00Z",
        "validUntil": "2026-10-21T16:05:00Z"
      }
    ]
  },
  "meta": { "traceId": "trace-demo-003", "timestamp": "2026-10-21T16:02:00Z" }
}
```

Overall items use `kind: ALL` without window fields. `asOf` anchors the evidence boundary; `computedAt` records generation; `sourceThrough` communicates processing coverage; `validUntil` bounds serving validity. Source/binding changes may invalidate earlier. LTM may lag raw history. Strict windows, age-driven refresh, and per-key expiry require the backend validation described in the Data-Model.

## 7. GET /memories — relevant facts and observations

```http
GET /cfs/v1/memories?refLabel=visitor&refId=visitor-42&q=budget%20and%20body%20style&limit=3
X-Tenant-Id: toyota
```

`q` is required, 1–1,000 characters; `limit` defaults to 5 and allows 1–20. Returns `data.memories`, each containing an opaque `memoryId`, `text`, `type` (`FACT` or `OBSERVATION`), and optional `occurredAt`. Missing scope/no matches returns an empty list. CFS never substitutes an unscoped Hindsight query when identity resolution fails.

This route requests relevant-memory retrieval without generative answer synthesis; backend retrieval can still use embeddings and reranking. Banks, tags, documents, and backend scores are not public fields. Outage is `503` rather than successful emptiness.

## 8. DELETE /contexts — erasure and polling

```http
DELETE /cfs/v1/contexts?refLabel=visitor&refId=visitor-42
X-Tenant-Id: toyota
```

An authorized erasure request finds both a directly keyed context and all contexts linked to the identity. For visitor-42, that includes the visitor's searches and every app's contributions within them. The cleanup covers raw events, Redis cache, Hindsight evidence/observations, and affected summaries. Other subjects' summaries are invalidated when they used deleted evidence; unrelated contexts remain.

`202` returns `data.status: PENDING` and a suggested `Retry-After` polling delay. Repeat the same DELETE; `200` returns `data.status: DONE` only after cleanup/invalidation completes. An unknown subject is already `DONE`. Repeating the request does not restart or postpone erasure. Proposed serving guard: affected data reads and writes return `409 ERASURE_PENDING` during cleanup.

Completion records must outlive Context removal so polling remains meaningful. Their lifetime, authorized reopening of an erased identity, and operational deletion deadlines still require policy.

## 9. Errors and readiness

| HTTP | CFS code / condition |
| --- | --- |
| `400` | `VALIDATION_ERROR`, `INVALID_POLICY`, `POLICY_REQUIRED`, `UNKNOWN_LTM_KEY`, `UNSUPPORTED_WINDOW` |
| `401` / `403` | Gateway authentication or authorization failure; CFS may return `FORBIDDEN` for tenant/subject/operation denial. |
| `404` | `CONTEXT_NOT_FOUND` on policy inspection. |
| `409` | `POLICY_CONFLICT`, `LTM_KEY_CONFLICT`, `ERASURE_PENDING` |
| `413` | `REQUEST_TOO_LARGE` |
| `429` | `RATE_LIMITED`; respect supplied retry delay. |
| `503` | `SERVICE_UNAVAILABLE`; a write may have committed before the failure response or timeout. See [retry behavior](#retry-behavior). |

CFS errors contain `error.code`, `message`, optional `details: [{path, message}]`, and `meta`. Clients should branch on codes, not message text. Proposed limits and runtime checks are detailed in OpenAPI; the file is a contract draft, not an API Gateway deployment template.

`GET /cfs/v1/health` requires IAM but no tenant header. `200` returns `data.status: OK`; `503` means process unavailable. This is process readiness, not proof that every backend is healthy.

## 10. Artifact validation

OpenAPI structure, examples, references, Markdown JSON/HTTP examples, and relative links are checked together. Rejection cases cover wrong fields, policy/value confusion, and invalid status combinations. The Overview's four Mermaid diagrams were rendered and visually checked with Mermaid 12.2.0. These are document checks, not implementation tests. With `openapi-spec-validator` 0.9.0 installed, rerun from the CFS root using `python3 -m openapi_spec_validator Updates/openapi.yaml`.
