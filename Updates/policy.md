# CFS policy

**Schema and creation reference · design proposal · 2026-10-22.** [Overview](overview.md) presents the design; [API](apis.md) shows complete requests; [OpenAPI](openapi.yaml) supplies machine-readable schemas.

## 1. Schema

The schema accepts business-defined labels. This concrete instance uses the labels from the [Search-Agent example](examples.md).

```json
{
  "stm": {
    "cache": { "ttl": "PT24H" },
    "durability": { "ttl": "P30D" }
  },
  "ltm": {
    "search": { "ttl": "P30D" },
    "visitor": {
      "ttl": "P365D",
      "windows": ["P7D", "P30D", "P90D"]
    }
  }
}
```

| Field | Type / required | Meaning |
| --- | --- | --- |
| `stm` | Object, required | Short-term event storage settings. |
| `stm.durability.ttl` | Duration, required | Raw-event retention in DynamoDB. |
| `stm.cache.ttl` | Duration, optional | Redis cache lifetime; omit `cache` to disable it. |
| `ltm` | Label-to-rule dictionary, optional | Configures long-term memory; omit to disable LTM. |
| `ltm.<label>.ttl` | Duration, required per label | Retention for this context's contribution to that subject's memory. |
| `ltm.<label>.windows` | Unique day-duration list, optional | Additional rolling look-back summaries (`P7D` = preceding seven days, not a calendar week). The proposed contract also provides an overall retained-evidence summary. |

Policy labels describe business subjects, such as `search` and `visitor`. They are not actual IDs. When LTM is enabled, include the context's own `refLabel`; other labels may be configured before their values are known. Request `ltm.visitor` supplies the actual visitor ID, while `policy.ltm.visitor` supplies its rules.

The policy governs grouping and lifetimes, not the fields each app must put inside `data`. Apps may write different event shapes and read different authorized scopes. The CFS Extractor generates the shared summaries; each app does not need to implement its own summarizer. Per-app exclusions or custom summary instructions are not defined by this schema.

Durations use positive whole days, hours, or minutes; windows use whole days. Draft bounds: cache 1 minute–7 days, durability 1–365 days, LTM 1–730 days. At most five LTM labels and five windows per label; each window must not exceed that label's TTL. Unknown fields are rejected. These limits remain proposed contract details.

## 2. How it is created

1. An authorized caller sends the complete `policy` inside the first `POST /cfs/v1/contexts`, with `refLabel`, `refId`, and an event. Creation without policy fails.
2. CFS validates the policy and supplied identity bindings, then creates one Context row and its Policy row atomically in Postgres.
3. The policy is locked permanently. Later writers use that same context and policy. An optionally repeated policy must be equivalent; a different policy conflicts. There is no policy PATCH.
4. `GET /cfs/v1/policies?refLabel=search&refId=search-123` returns that exact context's locked policy.

Every new context receives its own policy copy, even when the same configuration is reused. Actual binding values may be added under permitted labels without changing the policy. The business owner and authenticated principals permitted to establish the first policy still need signoff.

TTL clocks, cross-context aggregation, late-binding recovery, and Sweeper/Extractor behavior are explained in the [Data-Model](data-model.md#6-sweeper-extractor-and-lifecycle).
