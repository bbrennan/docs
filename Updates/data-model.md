# CFS data model

**Engineering reference · design proposal · 2026-10-22.** [Overview](overview.md) presents the end-to-end design; [Policy](policy.md) defines the configuration; [API](apis.md) and [OpenAPI](openapi.yaml) define caller behavior. Physical keys and worker fields below are proposed, not deployed DDL.

## 1. Identity and invariants

A context is unique by **`(tenantId, refLabel, refId)`**. Labels name business subjects; IDs identify individual instances. Apps encounter many values under the same label. Different authorized apps using the same tuple share one history and one locked policy.

| Value | Set by | Purpose and location |
| --- | --- | --- |
| `tenantId` | Platform, selected through an authorized request | Tenant isolation in every lookup. |
| `refLabel` / `refId` | Calling app | Business context identity, persisted in Postgres. Labels use business nouns; values are opaque IDs. |
| Request `ltm` | Calling app's identity integrations | Additional identity values, e.g. `{"labelB":"subject-42"}`; persisted on the context. |
| Internal `ctxId` | CFS, deterministically from the business tuple | Stable storage/queue reference, persisted on creation. Exact derivation below; not a public request field. |
| `entryId` | CFS | UUID for one stored event, returned to callers and reused internally for source traceability. |
| Internal bank/document IDs | CFS | Hindsight routing and source traceability; not public API inputs. |

Creation requires a policy. Postgres creates the Context and Policy records atomically; every subsequent writer accepts that immutable policy. Actual LTM values are bindings, not policy rules. An unset permitted binding may be added; an established value cannot change. Binding scope is the entire context, across writers.

Workflow owners agree the labels. The originating application obtains or creates the business ID and shares it with collaborating applications; subsequent writes/reads reuse that value. Identity integrations supply additional subject IDs, which callers pass in top-level request `ltm` when known, on creation or a later write. CFS does not mint separate business “LTM IDs” or infer associations from event content. The same business tuple always derives the same `ctx_id`.

There is no app dimension in context identity, event keys, cache keys, or summary selection. Tenant and subject authorization uses the authenticated principal, not a claimed application name. Treat business IDs, payloads, and query text as restricted: avoid them in logs and queue messages; an internal hash is not an authorization mechanism.

### ctxId derivation and creation

`ctxId` is the API/architecture name for the value stored as Postgres `ctx_id`. CFS computes it; the caller supplies only the business identity:

```text
identity = tenantId + "|" + refLabel + "|" + refId
digest   = lowercase_hex(SHA256(UTF8(identity)))
ctxId    = "c_" + digest[0:32]
```

The result is `c_` followed by 32 lowercase hexadecimal characters. This preserves the original design's hash recipe, with `refLabel` replacing `refType`.

- Inputs are the authorized tenant selector and validated `refLabel` / `refId`, in that order. Values are case-sensitive and used exactly as validated, without trimming or case conversion.
- The separator is the literal pipe character `|`. The API's allowed characters exclude it from all three inputs, so boundaries are unambiguous. Use this separator rather than `::`, because identifier values may contain colons.
- `appId`, request `ltm`, policy, event content, and timestamps are excluded. Another writer or additional binding does not change the context ID.

CFS validates and authorizes the request, computes the ID, and resolves the stored Context. If absent, a policy-bearing write creates the Context and Policy together using that ID. If present, it reuses the records and checks the locked policy/bindings. Reads never create records. The unique business-tuple constraint still applies; concurrent first writes must converge on the same pair.

For the generic example below, `tenant-1|labelA|ref-123` produces `c_5346011545625e2f6473c4b573aa9f22`. If an existing hash belongs to a different tuple, fail without reading, appending, or merging its data; do not silently assign an alternate ID. Changing this recipe requires a storage migration. Erasure/recreation permissions remain separate from hash computation.

## 2. Postgres: authoritative metadata

### Policy — one immutable row per context

| Column | Proposed type | Required | Meaning |
| --- | --- | --- | --- |
| `ctx_id` | text, PK/FK association with Context | Yes | Identifies the governed context; not a global policy per label. |
| `rules` | jsonb | Yes | Complete validated policy, including `stm` and optional `ltm` label-to-rule dictionary. |
| `created_at` | timestamptz | Yes | Policy establishment time, exposed by `GET /policies`. |

Enforce one-to-one association and the invariant that no committed Context lacks a Policy. Create both in the same Postgres transaction; the final DDL must enforce this invariant at commit. Equivalent policy bodies may be supplied on later writes, but they never update this row. Equivalence ignores object order, normalizes durations, and treats windows as sets.

### Context — identity, bindings, and work progress

| Column | Proposed type | Meaning |
| --- | --- | --- |
| `ctx_id` | text, PK | Deterministic `c_` + 32-hex hash defined above; also referenced by Policy. |
| `tenant_id`, `ref_label`, `ref_id` | text, NOT NULL | Unique business tuple; own-context LTM value comes from these fields. |
| `ltm` | jsonb, NOT NULL, default `{}` | Additional label-to-ID bindings, e.g. `{"labelB":"subject-42"}`. |
| `created_at` | timestamptz, NOT NULL | Metadata creation time. |
| `updated_at` | timestamptz, nullable before first event | Latest accepted event write; used for idle selection. |
| `committed_through` | ordered internal position, nullable | Monotonic boundary covering durably committed event work. |
| `extracted_through` | ordered internal position, nullable | Commit boundary successfully processed into LTM. |
| `pending_since` | timestamptz, nullable | Oldest unprocessed event work; six-hour backstop clock. |
| `binding_revision` | bigint, NOT NULL, default 0 | Advances when a permitted binding is added. |
| `extracted_binding_revision` | bigint, NOT NULL, default 0 | Binding revision reconciled by the Extractor. |
| `summary_due_at` | timestamptz, nullable | Next summary refresh/invalidation deadline, including periods without writes. |
| `erase_requested_at` | timestamptz, nullable | Excludes context from new writes/extraction while erasing. |

Worker claims/fencing and cross-store repair state are also required; their final representation depends on the write-handoff design described below. Read order `(recordedAt, UUID)` is not a safe commit cursor: a later UUID can sort behind an earlier one at the same timestamp. Extraction needs a monotonic commit sequence or equivalent allocation invariant, with no progress beyond unresolved earlier work.

### Concrete creation and lookup

For illustration, an application creates an activity identified by `labelA / ref-123`, linked to `labelB / subject-42`, within `tenant-1`. Its Context row includes:

```json
{
  "ctx_id": "c_5346011545625e2f6473c4b573aa9f22",
  "tenant_id": "tenant-1",
  "ref_label": "labelA",
  "ref_id": "ref-123",
  "ltm": { "labelB": "subject-42" },
  "binding_revision": 1,
  "extracted_binding_revision": 0,
  "extracted_through": null,
  "erase_requested_at": null
}
```

Its Policy row has the same `ctx_id` and `rules` equal to the request's complete `policy` object. Linked-subject memory does not require a separate Context/Policy row for that subject. A later write to the same business tuple adds an event, not another metadata pair. `labelA` and `labelB` are illustrative labels; storage does not prescribe a business taxonomy.

Required access paths:

| Lookup | Predicate / index requirement |
| --- | --- |
| Direct context | Unique `(tenant_id, ref_label, ref_id)`. |
| Linked-subject contexts | Tenant predicate plus `ltm @> '{"labelB":"subject-42"}'::jsonb`; tenant B-tree and containment-capable JSONB GIN indexes are candidates. |
| Extraction work | LTM-enabled, non-erasing contexts with pending events/revisions or due summaries; index due-work timestamps/state. |
| Erasure work | Direct tuple lookup plus linked-context lookup; maintain a subject guard to catch racing new bindings. |

## 3. DynamoDB: events and derived state

Proposed single-table layout uses `pk` / `sk`. `H(tenant, label, value)` below is the full 64-character lowercase SHA-256 hex digest of `UTF8(tenant + "|" + label + "|" + value)`, with the same validation/separator rules as above. It identifies a summary/erasure subject, which need not have a Context row. `ctxId` uses the first 32 characters of that digest plus `c_`; neither hash is a public identifier or authorization credential.

| Item | Partition key | Sort key |
| --- | --- | --- |
| Event | `CTX#{ctxId}` | `E#{recordedAt}#{entryId}` |
| Retained-source manifest | `CTX#{ctxId}` | `M#{entryId}` |
| Summary | `SUM#{H(tenant,label,value)}` | `ALL` or `WINDOW#{duration}` |
| Erasure guard/completion | `ERASE#{H(tenant,label,value)}` | `STATE` |

Stored event timestamps use normalized fixed-width UTC precision so sort-key order matches `(recordedAt, entryId)`. Reads reverse that order. No per-app items or APPS set are needed. Extraction additionally needs a committed-order access path, such as an ordered journal or indexed commit position; selecting that path is part of the write-handoff design, not solved by the time-ordered event key.

### Event fields

| Event field | Type | Meaning |
| --- | --- | --- |
| `entryId` | string UUID | Stable source ID. |
| `recordedAt` | timestamp string | CFS acceptance timestamp; retention clock. |
| `occurredAt` | timestamp string, optional | Caller-supplied event time; window clock when present. |
| `data` | JSON object/document | Application event, at most 65,536 encoded UTF-8 bytes. |
| `expiresAt` | epoch seconds | `recordedAt + stm.durability.ttl`. |

Each successful POST creates a new event and CFS-assigned `entryId`. The current contract does not store client retry keys or successful-result replay records; a repeated POST may append another event. Internal worker retries reuse the stored event ID to avoid retaining that same source twice.

Expired items are logically absent even before DynamoDB physically removes them. Recent-history queries filter expiry and continue paging until they collect the requested live entries or exhaust the range; filtering occurs after DynamoDB's query-page limit. [TTL](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html), [query filtering](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Query.FilterExpression.html).

### Retained-source manifest

Each manifest entry records its Hindsight bank/document reference, source event times, applied binding revision, per-bound-label expiry deadlines, and retain/delete progress. Persist durable intent before calling Hindsight, then reconcile the outcome; a successful engine write followed by failed bookkeeping must not leave undiscoverable evidence. The manifest survives raw-event expiry while LTM evidence exists, supporting retagging, selective expiry, and erasure. Retire it only after external cleanup is confirmed. Its final layout must preserve lineage and support bounded scans by context.

### Summary fields

| Field | Meaning |
| --- | --- |
| `text`, `kind`, optional `window` | Materialized overall/window result. |
| `asOf`, `windowStart`, `windowEnd` | Evidence evaluation time; window interval is `[start, end)`. |
| `computedAt` | Generation completion time, not a concurrency token. |
| `sourceThrough` | Latest recorded source timestamp considered; not proof that every new write is included. |
| `validUntil` | Hard serving deadline; changes may invalidate earlier. |
| Generation/coverage reference | Internal source/binding snapshots and publication fence; prevents stale workers replacing newer results. |
| `expiresAt` | Physical cleanup deadline, separate from serving validity. |

Publish only against a still-valid generation. Multiple contexts can contribute to one shared-subject summary, so publication fencing must coordinate at the shared subject scope as well as the individual context. Pending/invalidated results must not be served as `READY`.

The erasure control item stores the restricted subject selector, `PENDING`/`DONE`, request/completion times, and bounded cleanup progress. It protects identities without a directly keyed Context and preserves repeat-DELETE results after metadata removal. Completion retention and authorized reopening remain operational decisions.

## 4. Redis: shared recent-history cache

**Proposed UUID-compatible layout:** a sorted set orders entry IDs by recorded time, a hash stores event payloads by entry ID, and cache metadata records the covered durable watermark. Keys share a cluster hash slot: `ctx:{ctxId}:order`, `ctx:{ctxId}:entries`, `ctx:{ctxId}:meta`. For the example, the order key is `ctx:{c_5346011545625e2f6473c4b573aa9f22}:order`. Equal timestamps order by entry ID. Event UUIDs are sorted-set members rather than Redis stream IDs. [Redis sorted sets](https://redis.io/docs/latest/develop/data-types/sorted-sets/).

Keep the newest 64 live entries. Atomic update/refill merges by entry ID, trims both structures together, and refreshes their common cache TTL. Reads filter each event's source expiry; Redis TTL never extends raw retention.

A cache hit is usable only if its coverage matches the required durable snapshot. Missing/short/incomplete cache falls back to DynamoDB and refills without replacing newer data. Cache-write failure must invalidate or bypass an incomplete cache, not leave it silently authoritative. Erasure fencing must prevent late refills restoring readable deleted data. Exact atomic scripts and write-handoff integration require implementation review.

## 5. Hindsight: shared memory with source traceability

**Candidate:** one bank per tenant, rather than per application or individual context. A bank is the storage/isolation container; tags select business subject scopes within it. “Multiple LTM levels” means parallel groupings of eligible source evidence: one event can contribute to its own context's subject and additional bound subjects. Other contexts can contribute to those same subjects. These are not nested banks; a shared subject aggregates eligible evidence across linked contexts. [Retain](https://hindsight.vectorize.io/developer/api/retain).

### How CFS derives tags

The Extractor uses persisted metadata and source-event eligibility, not whichever fields happen to appear in the latest request:

1. Load the Context and its locked Policy; resolve the tenant's bank internally. No caller supplies a bank or Hindsight tag array.
2. For each label in `policy.rules.ltm`, use `context.ref_id` if the label equals `context.ref_label`; otherwise look up `context.ltm[label]`.
3. For each source event, include the label only if its value is bound and that event is still eligible under that label's retention rule. Unbound labels produce no tag. If LTM is disabled or no eligible scope remains, perform required cleanup without an unscoped retain.
4. Generate the internal `label:value` tags, use stable document IDs derived from `ctxId` and `entryId`, and retain with `observation_scopes: "per_tag"`. Event content supplies facts; explicit tags supply identity grouping. As eligibility changes, reconcile tags and derived state.

| Locked policy label | Stored value source | Initial tag for the example |
| --- | --- | --- |
| `labelA` (own subject) | `context.ref_label` / `context.ref_id` | `labelA:ref-123` |
| `labelB` (additional subject) | `context.ltm.labelB` | `labelB:subject-42` |

The own-subject value is automatic; additional values must have been explicitly bound. Tags may differ per event as individual contribution deadlines pass. Summary windows select evidence within a subject scope; they do not create additional permanent identity tags.

Non-empty, server-derived strict scope filters exclude untagged/global memories. Apply the same isolation to summary generation and any generated state consulted by it. Missing scope resolution must never become an unfiltered bank query. [Recall](https://hindsight.vectorize.io/developer/api/recall).

A new permitted identity binding reconciles earlier eligible evidence only while it is recoverable in STM or retained LTM. Deleted history cannot be recreated; an unbound policy label does not reserve a historical copy. Hindsight document tag updates replace the full tag set and trigger observation re-consolidation; CFS must also invalidate its summary copies. [Documents](https://hindsight.vectorize.io/developer/api/documents#update-document).

## 6. Sweeper, Extractor, and lifecycle

Every five minutes the Sweeper queues non-erasing, LTM-enabled contexts with due work: events idle 30 minutes; work pending six hours despite ongoing writes; an unreconciled binding revision on a previously extracted context; or summary deadlines. Initial bindings follow the first event extraction's idle/backstop schedule. Revision-triggered repair requires `extracted_through` to be non-null and `binding_revision > extracted_binding_revision`, so a creation-time binding does not bypass that schedule.

Jobs carry `{ctxId, upTo, bindingRevision, reason}`, not business ID values. The Extractor rechecks state, reads all writers' committed events through the snapshot, retains/repairs evidence, publishes valid summaries, and records completed progress. Summary-only jobs need not re-retain unchanged events. SQS retries failed jobs and sends exhausted work to a dead-letter queue. See [processing coordination](overview.md#6-asynchronous-ltm-processing).

**Proposed lifecycle rules:** raw-event and per-label LTM expiry use original `recordedAt`; later writes/bindings do not restart them. Windows use `occurredAt`, otherwise `recordedAt`. Shared-subject memory preserves each source contribution's expiry and uses the union of windows across active contributions. Overall summaries cover retained eligible evidence. An empty eligible set invalidates obsolete summaries and completes the job.

Independent expiry means removing a contribution from one expired scope while preserving it in another still-eligible scope. Strict windows must exclude older source evidence and observations derived from it; Hindsight `temporal_window` ranks rather than hard-filters. Neither behavior is solved by a tag string. Validate these against a pinned Hindsight version; separate scope materializations/banks remain the fallback. [Temporal semantics](https://hindsight.vectorize.io/developer/api/recall#temporal_window).

## 7. Reads, failures, and erasure

| Operation | Authoritative lookup |
| --- | --- |
| Context read | Resolve Postgres identity and erasure state; use complete Redis cache or query live DynamoDB events. |
| Policy read | Join the exact Context to its immutable Policy. |
| Summary read | Resolve authorized subject/configuration, then read a valid subject/window materialization. |
| Memory read | Resolve authorized subject and recall eligible Hindsight evidence in its strict scope. |
| Erasure | Establish subject guard, enumerate direct/linked contexts, then clean all associated stores. |

Proposed serving guard: affected data reads and writes return `409 ERASURE_PENDING` while cleanup is active. Policy inspection is metadata-only. Guard checks and cleanup fencing must also cover in-flight requests and worker results.

| Race / failure | Required outcome |
| --- | --- |
| Concurrent first writes | One metadata pair; compatible writer joins it, incompatible policy fails. |
| Metadata commits, event write fails | No successful event acknowledgment; reconcile pending metadata/work rather than leaving a phantom processed event. |
| Event commits, response/progress update fails | Repair internal processing progress from durable state. The client has an uncertain outcome; repeating POST may append another event. |
| Concurrent/out-of-order writes | Never advance extraction beyond unresolved earlier work. Timestamp ordering alone cannot prove this. |
| Duplicate/stale SQS delivery | Check progress/revisions and publication fence; no duplicate source or stale summary overwrite. |
| Hindsight outage | Retry without advancing completed work; observe backlog versus source retention. |
| Erasure races with write/extraction/refill | Subject guard and generation checks prevent publication/resurrection; cleanup converges before `DONE`. |

The cross-store handoff/repair protocol and generation-fencing mechanism must be selected before implementation; Postgres, DynamoDB, and Hindsight are not one transaction. This includes both metadata/event handoff and manifest/engine handoff. Cache consistency and checkpoint safety depend on that protocol.

Cleaner removes linked contexts' raw events, caches, manifests/Hindsight evidence, and affected summary copies. Invalidate other subjects' summaries when they used deleted sources. Delete Context/Policy metadata last, after cleanup is confirmed; retain the independent completion record. Unrelated contexts remain, and a shared tenant bank is not deleted to erase one subject.

**Ordinary expiry is different:** proposed default is to retain compact Context/Policy metadata after evidence expires, so later appends to the same business identity still use its locked policy. Empty source manifests can retire after confirmed cleanup. A finite metadata-retention period and corresponding recreation rules require agreement; event TTL does not implicitly unlock or delete the policy.

Implementation gates are independent-scope expiry, strict rolling windows, recoverable late-binding repair, cross-store/fencing correctness, and complete derived-state erasure. The model is independent of application-specific subject labels.
