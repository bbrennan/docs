# CFS design review summary

**Updated 2026-10-22.** This directory contains the revised Context Factory Service design for review and signoff. This README records the meeting feedback and subsequent discussion that shaped it; the linked documents describe the current proposal.

The team meeting focused on Policy, APIs, and Overview diagrams. The Data-Model was not reached and became a priority for the following reviews. The main documentation problem was length and density: the design needed a clear presentation path, with enough supporting detail to answer engineering questions.

## Reading the documents

| Document | Purpose |
| --- | --- |
| [Overview](overview.md) | End-to-end presentation: identity, policy, architecture, processing, retention, and decisions. Start here. |
| [Policy](policy.md) | Configuration schema, field meanings, and creation/locking behavior. |
| [Data-Model](data-model.md) | Stored records, keys, lookup paths, processing state, lifecycle, and failure handling. |
| [API](apis.md) | Caller behavior, examples, errors, shared reads, and retries. |
| [Examples](examples.md) | Search Agent demonstration using search and visitor identities only. |
| [OpenAPI](openapi.yaml) | Machine-readable draft of the API contract. |

## Meeting outcomes

- **Shared memory is central.** Multiple applications can contribute events to the same context. Memory can combine those contributions and aggregate across related contexts, such as a visitor's searches.
- **Remove `appId` from the header.** Its necessity was questioned; subsequent design discussion removed it from the public contract and context identity.
- **Make identity ownership explicit.** Explain who supplies business IDs, when additional identities become known, and how CFS derives its internal ID.
- **Require and lock policy at creation.** No context without a policy; no policy PATCH. All writers to the context use its established rules. Business ownership and creation permissions still need agreement.
- **Use `/cfs/v1`.** Replace the original API prefix consistently.
- **Give LTM keys individual rules.** Replace the list of grouping IDs with a label-to-rule dictionary supporting individual TTLs. Preserve rolling summary windows.
- **Focus the demonstration.** Sequence diagrams illustrate the Search Agent scenario; the architecture remains generic.

## Changes from subsequent discussion and reviews

| Area | Result in the revised documents |
| --- | --- |
| Identity and naming | Standardize on `refLabel` and `refId`. The tenant/label/value tuple selects one context; CFS deterministically hashes it into internal `ctxId`. Restore the original hash recipe with explicit inputs, separator, and creation rules in the [Data-Model](data-model.md#ctxid-derivation-and-creation). One application can encounter many IDs under each label. |
| LTM values versus rules | Request `ltm` supplies additional label-to-ID bindings; `policy.ltm` supplies rules for those labels. The context's own value comes from `refLabel` / `refId`. No separate public LTM ID is introduced. |
| Multiple writers | Reads return shared history. A different policy or conflicting binding is rejected rather than creating another context. Apps can consume different authorized summaries; their events still contribute to every bound, eligible scope. |
| Tags and banks | CFS derives internal tags from persisted identities, permitted policy labels, and source eligibility. Payload fields do not establish bindings. A tenant bank with independent tagged scopes is a candidate layout, pending validation. |
| Storage detail | Restore substantive Context/Policy fields, DynamoDB items, Redis behavior, source lineage, work progress, and failure cases. Keep the data-model independent of the Search Agent use case. |
| Background processing | Restore Sweeper, SQS, Extractor, and Cleaner responsibilities. Cover idle/backstop selection, later binding repair, age-driven refresh, retries, and erasure. Initial bindings follow the first extraction's idle/backstop schedule. |
| Write retries | Defer client duplicate-suppression machinery to keep the current API simple. Repeated POSTs may append another event; internal worker retries still reuse stored source IDs. |
| Documentation | Integrate explanations into their relevant sections rather than an alignment Q&A. Keep Overview presentation-focused, Policy concise, and API/Data-Model detailed enough for follow-up questions. Repair and render-check Mermaid diagrams. |
| Contract consistency | Align examples, errors, naming, routes, and schemas with `openapi.yaml`. Mark proposals and implementation gates explicitly. |

The Search Agent example uses only search and visitor identities. Dealer grouping was a hypothetical future use case, not an added demonstration requirement.

## Decisions and validation still open

The review supports shared business identity, immutable context policy, and explicit source lineage. It also identified that tags alone do not establish retention, strict time windows, or complete erasure guarantees.

Remaining decisions include policy ownership/grants; TTL/window clocks; late-binding recovery; overall-plus-window summaries; reconciliation across contributing contexts; metadata retirement; and erasure completion, reopening, and deadlines.

Before implementation commitment, validate Hindsight isolation, independent scope expiry, strict windows, late-binding repair, and derived-memory erasure against a pinned version. Select cross-store recovery and publication-fencing mechanisms, and measure worker freshness and bank scale. The [Overview's decision section](overview.md#8-decisions-and-implementation-gates) is the current reference for these gates.

Document checks cover OpenAPI structure, examples, references, and local links. All four Overview diagrams have been rendered and visually checked. These checks validate the artifacts; backend behavior still requires implementation validation.
