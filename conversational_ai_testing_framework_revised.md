# Testing a Deterministic Conversational AI System
## A Software Engineering Strategy for Fuzzy Multi-Turn Inputs Without Overfitting the Tests to the Implementation

**Context:** Conversational shopping assistant for automotive sales  
**Architecture:** Highly deterministic application using LLMs primarily for utterance classification and natural-language response generation; structured catalogs for mappings; graph-based conversational flow; OpenSearch for live inventory/counts; Golden Journeys for end-to-end evaluation.

---

# Executive Summary

The central problem is not simply that conversational systems have a large input space.

The deeper problem is:

> **A conversational application is a stateful program whose input language is effectively unbounded, while its implementation is usually composed of finite rules, graphs, mappings, and heuristics.**

This creates two distinct testing risks:

1. **Coverage risk**  
   Human-authored tests such as Golden Journeys cover only a small subset of possible conversations.

2. **Oracle risk**  
   If the test framework derives expected behavior directly from the implementation, the tests can become circular and self-justifying.

A testing framework can therefore achieve high pass rates while still validating bugs, anti-patterns, or flawed product logic.

The central design principle of this proposal is:

> **The production implementation should be observed by the test framework, but should not be the primary source of truth for behavioral correctness.**

The recommended testing strategy separates:

- **product/domain specification** — what should happen;
- **test generation** — how unexplored cases are created;
- **system under test** — what the implementation actually does;
- **behavioral oracles** — how correctness is determined independently;
- **implementation coverage** — what parts of the code/graph/rules were exercised;
- **semantic evaluation** — where deterministic correctness is insufficient.

The proposed core methods are:

1. Golden Journey regression testing
2. Model-Based Testing
3. Stateful Property-Based Testing
4. Metamorphic Testing
5. Combinatorial / t-way Testing
6. Semantic and coverage-guided fuzzing
7. Mutation testing
8. Production-derived replay and failure mining
9. Independent domain/data oracles
10. Selective rubric-based LLM-as-a-Judge evaluation

The most important architectural change is that these methods should **not all share the same source of truth**.

---

# 1. Introduction: The Problem

The application under test is intentionally highly deterministic.

At a high level:

- LLMs classify user utterances into structured intent/filter representations.
- Structured catalogs map user concepts into business entities.
- Conversation follows graph patterns and explicit navigation logic.
- A live OpenSearch inventory provides authoritative option counts.
- Response generation converts structured results into natural language.
- Golden Journeys encode known critical user experiences.

This design offers important advantages:

- factual control;
- lower hallucination risk;
- predictable business behavior;
- explainability;
- low latency;
- reproducibility;
- traceable state.

However, it introduces a scaling problem.

The implementation is finite.

Human language is not.

---

## 1.1 The unbounded input-space problem

The following utterances may all express nearly the same semantic request:

```text
Show me Toyota SUVs under $40k.
```

```text
I want an SUV from Toyota, max forty grand.
```

```text
Toyota SUV <= 40k.
```

```text
Anything Toyota that's an SUV for less than forty thousand?
```

```text
Let's keep it under 40. Toyota SUV.
```

At the same time, very small linguistic differences may materially change intent:

```text
under $40k
```

versus:

```text
around $40k
```

or:

```text
Toyota
```

versus:

```text
not Toyota
```

or:

```text
show me hybrids
```

versus:

```text
does that one come as a hybrid?
```

This makes natural-language testing fundamentally different from conventional API testing.

---

## 1.2 Multi-turn state makes the problem harder

Consider:

```text
1. Show me SUVs.
2. Toyota.
3. Under $40k.
4. Actually, Honda.
5. Does that come in AWD?
6. No, I meant the Toyota.
```

The correctness of turn 6 depends on:

- previous state;
- entity history;
- active constraints;
- conversation references;
- whether "the Toyota" refers to the prior make or a prior vehicle;
- whether previous filters should be restored;
- whether a pivot should clear incompatible state.

This is not simply an utterance-classification problem.

It is a **stateful behavioral testing problem**.

---

# 2. Why Golden Journeys Are Necessary but Insufficient

Golden Journeys are valuable.

They answer:

> **Does the critical experience we already understand still work?**

They are excellent for:

- release gates;
- known business workflows;
- stakeholder-readable acceptance tests;
- regression protection;
- known defect reproduction;
- deterministic end-to-end evaluation.

But they have a structural limitation:

> **They encode the imagination and historical experience of the test designers.**

If nobody has imagined:

```text
correction after pivot
```

or:

```text
constraint removal after zero results
```

or:

```text
pronoun referring to an entity two turns earlier
```

or:

```text
replace make while preserving price and body style
```

then those behaviors may never be exercised.

A high Golden Journey pass rate therefore does not imply high behavioral coverage.

---

# 3. The More Important Risk: Self-Justifying Tests

Coverage is only one half of the problem.

There is another failure mode:

> **The tests may be overfit to the implementation.**

Suppose the production system has this logic:

```text
Changing make clears model AND price.
```

Now suppose the test model was generated directly from that same rule.

The test executes:

```text
Toyota
RAV4
under $40k
Actually Honda
```

The production system returns:

```text
Honda
no model
no price
```

The test model expects:

```text
Honda
no model
no price
```

The test passes.

But the intended product behavior may have been:

```text
Honda
SUV
under $40k
```

The testing framework has successfully proven that:

> **The implementation behaves like the implementation.**

That is not the same as proving correctness.

---

# 4. The Test Oracle Problem

In software testing, the **test oracle problem** refers to the difficulty of determining what the correct output should be for arbitrary inputs.

This is particularly difficult for conversational systems because:

- exact response wording may vary;
- semantic correctness may not have a single textual answer;
- many inputs have equivalent meanings;
- state changes matter more than wording;
- internal rules may themselves be wrong.

The solution is not to derive all expected behavior from the code.

Instead, different classes of truth should come from different sources.

---

# 5. Separate "What Should Happen?" from "What Did the Code Do?"

The proposed architecture has two distinct views of the system.

## 5.1 Black-box behavioral view

This asks:

> **Given the conversation and the product requirements, did the system do the right thing?**

This view should depend primarily on:

- product requirements;
- business rules;
- domain axioms;
- catalog truth;
- live inventory truth;
- semantic relations between inputs.

It should not need to know:

```text
which helper function executed
which graph node fired
which rule ID matched
which internal pipeline branch was used
```

---

## 5.2 White-box implementation view

This asks:

> **What parts of the implementation did the test exercise?**

This includes:

- graph vertex coverage;
- graph edge coverage;
- graph edge-pair coverage;
- branch coverage;
- rule coverage;
- code coverage;
- classifier-schema validity;
- pipeline traces.

This is valuable, but it is **diagnostic and structural**, not the primary behavioral oracle.

---

# 6. The Three-Model Architecture

A robust testing strategy should maintain three conceptually distinct models.

```text
              PRODUCT / DOMAIN SPECIFICATION
                    "What should happen?"
                              |
                              v
                      BLACK-BOX ORACLE
                              |
                              |
User Conversation ------> SYSTEM UNDER TEST
                              |
                              |
                              v
                     IMPLEMENTATION TRACE
                              |
                              v
                    WHITE-BOX COVERAGE
```

Separately:

```text
                 ADVERSARIAL TEST GENERATOR
                            |
        +-------------------+-------------------+
        |                   |                   |
   Property-based      Metamorphic          LLM generator
    sequences            mutations
```

This separation is intentional.

The implementation should not define all three layers.

---

# 7. What Should Define Correctness?

Correctness should be based on **domain-level behavior**, not internal implementation.

Example requirement:

> When a user changes only the manufacturer, independent constraints should remain unless the user explicitly indicates otherwise.

Example:

```text
Current state:
Toyota
SUV
under $40k
```

User says:

```text
Actually Honda.
```

Expected behavior:

```text
Honda
SUV
under $40k
```

The specification should not say:

```text
Call replace_make()
Take graph edge 17
Clear slot 6
Preserve slot 8
Preserve slot 9
```

Those are implementation details.

A good test should survive implementation changes.

---

# 8. The Refactor Test for Test Quality

For every behavioral test, ask:

> **If we completely replaced the navigation engine tomorrow, should this test still be valid?**

For example:

```text
graph-based navigation
```

could become:

```text
LLM planner
```

or:

```text
finite-state workflow engine
```

or:

```text
constraint solver
```

If the behavioral test remains unchanged, it is likely well specified.

If the expected result must be rewritten because internal code structure changed, the test is probably too implementation-coupled.

---

# 9. Behavioral Contracts Instead of Implementation Contracts

Represent behavior at the domain level.

Example:

```yaml
property: independent_constraint_preservation

given:
  active_constraints:
    body_style: SUV
    make: Toyota
    price_max: 40000

when:
  semantic_operation:
    replace:
      make: Honda

then:
  active_constraints:
    body_style: SUV
    make: Honda
    price_max: 40000
```

Notice what is absent:

```text
graph node
Python class
Bedrock prompt
navigation handler
rule ID
```

That is desirable.

---

# 10. Oracle Provenance Should Be Explicit

Every assertion should record where its expected truth comes from.

Recommended field:

```yaml
oracle_source:
```

Allowed values:

```text
PRODUCT_REQUIREMENT
DOMAIN_AXIOM
CATALOG
INVENTORY
METAMORPHIC_RELATION
HUMAN_LABELED
LLM_JUDGE
IMPLEMENTATION
```

This creates an important governance metric.

Example:

```text
Behavioral assertions by oracle source:

PRODUCT_REQUIREMENT     31%
DOMAIN_AXIOM            24%
CATALOG                 18%
INVENTORY               14%
METAMORPHIC_RELATION     9%
HUMAN_LABELED            3%
LLM_JUDGE                1%
IMPLEMENTATION            0%
```

For behavioral correctness, that is healthy.

By contrast:

```text
IMPLEMENTATION           72%
```

would be a warning that the framework may simply be confirming the existing system.

Implementation-based assertions are still appropriate for:

- schema validity;
- internal structural guarantees;
- execution coverage;
- instrumentation integrity.

But they should not dominate product correctness.

---

# 11. Summary of Testing Methods to Consider

| Method | Primary question | Implementation independence | Priority |
|---|---|---:|---:|
| Golden Journeys | Does this known scenario still work? | High if spec-driven | Core |
| Model-Based Testing | Have we exercised important states/transitions? | Depends on model | Core |
| Stateful Property-Based Testing | Do invariants survive generated sequences? | High | Core |
| Metamorphic Testing | Do related inputs produce appropriately related behavior? | Very high | Core |
| Combinatorial Testing | Have important factor interactions been covered? | High | Core |
| Semantic Fuzzing | Can generated inputs discover unknown behavior? | High | Core |
| Grammar-Based Fuzzing | Can structured linguistic cases be systematically generated? | High | Useful |
| Mutation Testing | Would our tests catch plausible implementation bugs? | Very high | Core validation |
| Differential Testing | Do independent implementations disagree? | High | Situational |
| Formal Model Checking | Do selected abstract properties always hold? | High | Situational |
| Production Replay | What real behavior are we missing? | High | Important |
| LLM-as-a-Judge | Was subjective semantic behavior acceptable? | Moderate | Selective |

---

# 12. Golden Journey Testing

## Purpose

Golden Journeys remain the regression layer.

They should represent:

- critical customer workflows;
- common tasks;
- business-critical paths;
- previously discovered bugs;
- high-value edge cases.

## Important improvement

Golden Journeys should assert **behavioral contracts**, not merely snapshots of current implementation behavior.

For example:

Bad:

```text
Expected graph node = NODE_17
```

Better:

```text
Expected active constraints:
make = Toyota
body_style = SUV
price_max = 40000
```

---

# 13. Model-Based Testing

## Concept

Model-Based Testing represents system behavior using states and transitions and generates tests by traversing the model.

References:

- Microsoft Model-Based Testing / Spec Explorer overview
- GraphWalker model-based testing documentation

## Critical distinction

There should be two models.

### Production graph

Represents actual implementation.

Used for:

```text
coverage
diagnostics
unexpected transition detection
dead state detection
```

### Abstract specification graph

Represents product-level behavior.

Example:

```text
NO_SEARCH
   |
   v
SEARCH_IN_PROGRESS
   |
   +--> RESULTS
   |
   +--> ZERO_RESULTS
   |
   +--> CLARIFICATION_REQUIRED
```

Operations:

```text
REFINE
RELAX
REPLACE
PIVOT
CORRECT
RESET
ASK
```

This model should intentionally be simpler than the production implementation.

## Recommendation

Use both.

Never treat the production graph alone as the behavioral oracle.

---

# 14. Stateful Property-Based Testing

## Concept

Instead of enumerating complete conversations, define:

- actions;
- preconditions;
- invariants;
- generated sequences.

Hypothesis provides `RuleBasedStateMachine` for this purpose.

Possible operations:

```text
select_make()
select_model()
set_price()
add_feature()
remove_constraint()
replace_constraint()
correct_previous_turn()
pivot_vehicle()
reset_search()
```

Generated sequence:

```text
1. Toyota
2. SUV
3. RAV4
4. under $40k
5. Honda instead
6. does it have AWD?
7. remove price limit
8. actually go back to Toyota
```

## Why it is valuable

The sequence does not need to be imagined by a human.

## Shrinking

Property-based testing frameworks attempt to minimize failures.

A 20-turn failure may shrink to:

```text
Toyota
RAV4
Honda
Actually Toyota
```

That makes debugging significantly easier.

---

# 15. Properties and Invariants

Examples should come from product/domain semantics.

## Constraint preservation

Adding an independent constraint should preserve unrelated constraints.

```text
Before:
make = Toyota
body_style = SUV

User:
Under $40k

After:
make = Toyota
body_style = SUV
price_max = 40000
```

---

## Restrictive-filter monotonicity

If a filter is strictly restrictive:

```text
count(SUV)
>= count(Toyota SUV)
>= count(Toyota SUV under $40k)
```

If this fails, likely causes include:

- query construction bug;
- stale state;
- inventory filtering error;
- unexpected constraint reset.

---

## Idempotence

Repeating the same semantic constraint should not create unexpected behavior.

```text
Toyota
Toyota
```

should normally be equivalent to:

```text
Toyota
```

unless repetition itself has explicit semantics.

---

## Reordering equivalence

For independent constraints:

```text
Toyota -> Hybrid -> Under $40k
```

and:

```text
Under $40k -> Toyota -> Hybrid
```

should converge to equivalent final state.

---

## Pivot correctness

Example:

```text
Toyota + SUV + < $40k
```

then:

```text
Actually Honda
```

should usually produce:

```text
Honda + SUV + < $40k
```

with incompatible Toyota-specific state removed.

---

## Reset correctness

After reset:

```text
previous search filters = empty
stale references = cleared
new utterance = interpreted independently
```

---

# 16. Metamorphic Testing

## Concept

Metamorphic Testing is valuable when exact expected outputs are difficult to enumerate.

Instead of specifying exact output text, define relationships between related inputs.

This is especially well suited to conversational AI.

---

## Paraphrase invariance

Input A:

```text
Show me Toyota SUVs under $40k.
```

Input B:

```text
I'm looking for a Toyota SUV and don't want to spend over forty grand.
```

Expected relationship:

```text
same semantic constraints
same eligible inventory
same underlying task
```

The natural-language response does not need to match exactly.

---

## Controlled semantic mutation

Original:

```text
Toyota SUV under $40k
```

Mutation:

```text
Toyota SUV under $30k
```

Expected:

```text
make unchanged
body style unchanged
price changes
result count cannot increase
```

---

## Entity substitution

```text
Toyota -> Honda
```

Expected:

```text
manufacturer changes
independent constraints remain
results recompute
```

---

## Language-noise transformations

Add:

- politeness;
- filler;
- punctuation noise;
- capitalization differences;
- typos;
- slang;
- sentence fragments.

Expected:

```text
core meaning preserved
```

when the transformation is meaning-preserving.

---

# 17. Combinatorial / t-Way Testing

Behavior may depend on:

```text
current state
prior state
operation
entity type
inventory bucket
reference type
ambiguity
negation
correction
pivot
```

Testing the full Cartesian product is usually infeasible.

NIST combinatorial testing work addresses this with covering arrays.

Example factor model:

```text
State:
- START
- MAKE_SELECTED
- MODEL_SELECTED
- RESULTS

Operation:
- REFINE
- REPLACE
- CORRECT
- PIVOT
- QUESTION

Inventory:
- ZERO
- ONE
- FEW
- MANY

Entity:
- MAKE
- MODEL
- TRIM
- PRICE
- FEATURE

Language:
- EXPLICIT
- IMPLICIT
- AMBIGUOUS
- NEGATED
```

Rather than exhaustively testing all combinations, use:

```text
2-way global coverage
3-way coverage for high-risk factors
4-way selectively for critical combinations
```

---

# 18. Semantic Fuzzing

Traditional fuzzing mutates low-level input.

For conversational systems, mutate semantics and language.

Mutation categories:

## Surface

```text
typos
capitalization
punctuation
abbreviations
number formatting
```

## Pragmatic

```text
filler
hesitation
politeness
irrelevant context
fragments
```

## Semantic

```text
negation
constraint addition
constraint removal
replacement
contradiction
comparison
disjunction
```

## Conversational

```text
pronouns
ellipsis
callbacks
"actually"
"instead"
"go back"
"what about..."
multi-intent turns
```

The important principle is:

> Retain generated conversations that discover new behavior.

Possible behavioral coverage signals:

```text
new graph edge
new edge pair
new state/operation combination
new filter-delta signature
new recovery path
new rule branch
new classifier boundary
```

---

# 19. Grammar-Based Fuzzing

A grammar can represent semantic conversation operations.

Example:

```text
<utterance> ::= <refinement>
              | <replacement>
              | <correction>
              | <question>
              | <pivot>

<correction> ::= "actually" <replacement>
               | "I meant" <replacement>
               | "no" <replacement>

<price> ::= "under" <money>
          | "less than" <money>
          | "max" <money>
```

Do not attempt to model English completely.

Model the **business conversation language**.

---

# 20. Mutation Testing: A Key Defense Against Self-Confirmation

Mutation testing deliberately introduces plausible faults into the implementation.

Examples:

```text
Changing make clears price.
```

```text
"not hybrid" is interpreted as hybrid.
```

```text
zero results take the normal-results branch.
```

```text
reset fails to clear one stale constraint.
```

Then ask:

> **Does the evaluation suite catch the defect?**

If not, the testing framework has a blind spot.

This is one of the strongest tools for validating the **testing system itself**.

---

# 21. Deliberate Fault-Challenge Suite

Create a library of seeded defect types.

Example:

```text
Fault A:
forget price after make pivot

Fault B:
retain incompatible model after make pivot

Fault C:
inventory count off by one

Fault D:
negation ignored

Fault E:
latest entity always overrides historical reference

Fault F:
reset leaves stale model

Fault G:
zero-results state skips clarification

Fault H:
multiple constraints treated as replacements
```

Then measure which test families detect each defect.

Example:

| Fault | Golden | Property | Metamorphic | Fuzz | LLM Judge |
|---|---:|---:|---:|---:|---:|
| Preserve-price failure | Sometimes | Yes | Yes | Yes | Maybe |
| Wrong inventory count | Yes | Yes | N/A | Yes | Poor fit |
| Misleading narration | Sometimes | Limited | Sometimes | Sometimes | Yes |
| Negation error | Sometimes | Yes | Yes | Yes | Maybe |
| Conversational incoherence | Weak | Sometimes | Sometimes | Yes | Yes |

This creates a test-quality matrix.

---

# 22. Differential Testing

Differential testing compares two implementations.

Potential uses:

```text
old navigation engine vs new engine
classifier A vs classifier B
deterministic parser vs LLM classifier
production release vs candidate release
```

Disagreement does not automatically indicate which one is correct.

But disagreement is a powerful signal for investigation.

This is particularly useful during:

- migrations;
- model upgrades;
- architectural redesigns.

---

# 23. Formal Model Checking

Formal model checking may be useful for selected abstract properties.

Examples:

```text
A user can always reset to START.
```

```text
ZERO_RESULTS always has at least one recovery transition.
```

```text
A selected model cannot belong to a different make than selected_make.
```

```text
No valid nonterminal state is a dead end.
```

Use this selectively on the abstract specification model.

Avoid attempting to formally model the full natural-language system.

---

# 24. Production-Derived Testing

Real conversations should continuously feed the test corpus.

Useful sources:

- anonymized production transcripts;
- customer-support incidents;
- manually reported bugs;
- unusual correction patterns;
- unusual pivot patterns;
- zero-result sessions;
- abandoned flows.

Every production defect should trigger two actions:

1. Add the exact minimized failure as a regression.
2. Ask what broader property or generator should have found this class of failure.

This prevents the suite from becoming a museum of one-off examples.

---

# 25. Do We Need LLM-as-a-Judge?

## Short answer

**Yes, but only for the portion of correctness that cannot be reliably determined with stronger deterministic oracles.**

For this architecture, LLM-as-a-Judge should be a **small semantic layer**, not the foundation of the evaluation framework.

---

# 26. What an LLM Judge Should NOT Decide

Do not ask an LLM judge whether:

```text
the price filter is correct
```

if structured state can answer it.

Do not ask whether:

```text
inventory count is correct
```

if OpenSearch can answer it.

Do not ask whether:

```text
returned vehicles satisfy filters
```

if the catalog can answer it.

Do not ask whether:

```text
the graph transition was legal
```

if graph rules can answer it.

These have stronger deterministic oracles.

---

# 27. What an LLM Judge IS Useful For

Use LLM judges for subjective semantic qualities.

## Goal understanding

User:

```text
I need something for my teenage daughter.
Safety matters much more than performance,
but I don't want anything enormous.
```

The structured system may produce technically valid filters.

But an LLM judge can help assess:

> Did the interaction meaningfully respond to the user's underlying goal?

---

## Conversational coherence

Example:

```text
No, I meant the other one.
```

Did the response use conversational context sensibly?

---

## Clarification quality

Example:

```text
I want the nicer one.
```

Did the system:

- ask a reasonable clarification;
- avoid inventing unsupported preferences;
- acknowledge ambiguity?

---

## Helpfulness

Did the response technically satisfy the constraints but fail to help the user make progress?

---

## Semantic response equivalence

Two responses may differ in wording while being functionally equivalent.

An LLM judge can help assess whether they communicate the same meaning.

---

## Unsupported narration

Even if structured state is correct, response-generation may introduce wording that:

- overstates;
- implies unsupported facts;
- misrepresents uncertainty;
- confuses the user.

An LLM judge may be useful here.

---

# 28. Treat the LLM Judge as a Fallible Sensor

An LLM judge should not automatically be treated as ground truth.

Its score should often mean:

```text
INVESTIGATE
```

rather than:

```text
FAIL
```

unless the judge has been calibrated against trusted human judgments.

A useful hierarchy:

```text
deterministic invariant violation
    -> hard FAIL

catalog/inventory inconsistency
    -> hard FAIL

metamorphic relation violation
    -> hard or high-confidence FAIL

LLM judge concern
    -> review / secondary signal
```

---

# 29. Use Rubric-Based Judging

Do not ask:

```text
Is this response good?
```

Use explicit criteria:

```text
1. Did the response address the user's latest goal?
2. Did it preserve relevant prior context?
3. Did it avoid contradicting stated constraints?
4. Did it avoid unsupported factual claims?
5. If ambiguity existed, did it avoid unjustified assumptions?
6. Was clarification appropriate?
```

This is more interpretable and more calibratable.

---

# 30. Avoid Correlated LLM Errors

Do not build:

```text
LLM A generates utterance
LLM A classifies utterance
LLM A judges whether classification was correct
```

That creates correlated failure risk.

Better:

```text
Semantic operation defined deterministically
        |
        v
LLM A generates natural-language realization
        |
        v
Application classifier processes it
        |
        v
Deterministic semantic oracle checks result
        |
        v
Optional LLM B judges response quality
```

Where possible:

- use different model families for generation and judging;
- calibrate judges on human-labeled examples;
- periodically inspect disagreement rates.

---

# 31. Four-Oracle Strategy

Recommended architecture:

| Oracle | Source of truth | Best for |
|---|---|---|
| Domain oracle | Product/business specification | Behavioral correctness |
| Data oracle | Catalog + OpenSearch | Facts/counts |
| Metamorphic oracle | Relations between transformed cases | Semantic consistency |
| LLM semantic oracle | Independent rubric | Coherence/helpfulness |

Separately:

| Diagnostic layer | Purpose |
|---|---|
| Implementation traces | Explain why failure occurred |
| Graph coverage | Determine structural execution |
| Rule coverage | Identify untested implementation |
| Code coverage | Identify execution gaps |
| Mutation testing | Validate test sensitivity |

The central distinction is:

> **Implementation traces diagnose correctness. They do not define correctness.**

---

# 32. Proposed Testing Architecture

```text
                         PRODUCT INTENT
                              |
                      DOMAIN SPECIFICATION
                              |
                +-------------+-------------+
                |                           |
        Behavioral Properties        Human Calibration
                |
     +----------+-----------+------------+
     |                      |            |
Property Tests        Metamorphic    LLM Judge
     |                      |            |
     +----------+-----------+------------+
                |
                v
          SYSTEM UNDER TEST
                |
                v
       IMPLEMENTATION TRACE
                |
      +---------+----------+
      |         |          |
   Graph      Rules       Code
 Coverage   Coverage    Coverage
```

Across the entire system:

```text
      ADVERSARIAL / GENERATED TEST INPUTS
                    |
        +-----------+-----------+
        |           |           |
     Stateful    Semantic   Combinatorial
     Generator     Fuzzing     Generator
```

---

# 33. Internal Representation of a Test Case

Do not make English the only representation.

Represent each turn as both:

1. intended semantic operation;
2. linguistic realization.

Example:

```yaml
operation:
  type: ADD_CONSTRAINT
  entity: price
  operator: LESS_THAN_OR_EQUAL
  value: 40000

utterance:
  text: "I'd like to keep it under forty grand."
```

This provides an independent semantic oracle for the generated utterance.

---

# 34. Semantic Operation Taxonomy

Define a finite language of user actions.

## Search construction

```text
ADD_CONSTRAINT
REMOVE_CONSTRAINT
REPLACE_CONSTRAINT
RELAX_CONSTRAINT
TIGHTEN_CONSTRAINT
ADD_ALTERNATIVE
```

## Navigation

```text
SELECT_ENTITY
DRILL_DOWN
GO_BACK
RESET
RESTART
```

## Conversation state

```text
CORRECT_PREVIOUS
REPEAT
CONFIRM
REJECT
PIVOT
RETURN_TO_PREVIOUS
```

## Information requests

```text
ASK_ATTRIBUTE
ASK_AVAILABILITY
ASK_COMPARISON
ASK_EXPLANATION
ASK_RECOMMENDATION
```

## Ambiguity/error

```text
AMBIGUOUS_REFERENCE
UNKNOWN_ENTITY
CONFLICTING_CONSTRAINT
OUT_OF_DOMAIN
MULTIPLE_INTENTS
INCOMPLETE_REQUEST
```

This becomes the common foundation for:

- property-based testing;
- abstract model-based testing;
- combinatorial coverage;
- metamorphic generation;
- semantic fuzzing.

---

# 35. Proposed Property Registry

Create a centralized domain property registry.

Example:

```python
PROPERTY_REGISTRY = [
    valid_catalog_entities,
    legal_domain_state,
    inventory_count_matches_search,
    returned_items_satisfy_filters,
    restrictive_constraint_is_monotonic,
    independent_constraint_is_preserved,
    reset_clears_state,
    repeated_constraint_is_idempotent,
    model_belongs_to_make,
    no_stale_reference_after_reset,
]
```

Each property should document:

```text
Property ID
Business rationale
Source of truth
Applicable states
Applicable operations
Exceptions
Severity
Oracle implementation
```

Example:

```yaml
id: FILTER_PRESERVATION_001

description: >
  Replacing the make constraint preserves independent
  body-style and price constraints.

oracle_source: PRODUCT_REQUIREMENT

applies_when:
  operation: REPLACE_CONSTRAINT
  entity: make

exceptions:
  - incompatible_catalog_mapping

severity: critical
```

---

# 36. Metamorphic Transformation Library

Create reusable transformation operators.

## Meaning-preserving

```text
PARAPHRASE
POLITENESS
ADD_FILLER
CHANGE_WORD_ORDER
NUMBER_FORMAT
ABBREVIATE
CASE_CHANGE
PUNCTUATION_CHANGE
MISSPELLING
FRAGMENT
```

Expected:

```text
semantic representation remains equivalent
```

---

## Meaning-changing with known delta

```text
CHANGE_MAKE
CHANGE_MODEL
INCREASE_PRICE
DECREASE_PRICE
ADD_NEGATION
REMOVE_NEGATION
ADD_CONSTRAINT
REMOVE_CONSTRAINT
REPLACE_CONSTRAINT
```

Expected:

```text
only specified semantic dimensions change
```

---

## Conversation transformations

```text
INSERT_REPEAT
INSERT_IRRELEVANT_TURN
CORRECT_LAST_TURN
REORDER_INDEPENDENT_TURNS
PIVOT_AND_RETURN
RESET_AND_RESTART
```

---

# 37. Combinatorial Coverage Model

Example factors:

```yaml
state:
  - START
  - DISCOVERY
  - MAKE_SELECTED
  - MODEL_SELECTED
  - RESULTS
  - ZERO_RESULTS

operation:
  - ADD
  - REMOVE
  - REPLACE
  - CORRECT
  - PIVOT
  - QUESTION

entity_type:
  - MAKE
  - MODEL
  - TRIM
  - PRICE
  - FEATURE

inventory_bucket:
  - ZERO
  - ONE
  - FEW
  - MANY

reference_style:
  - EXPLICIT
  - PRONOUN
  - ELLIPSIS
  - NONE

language_style:
  - EXPLICIT
  - IMPLICIT
  - AMBIGUOUS
  - NEGATED
```

Recommended targets:

```text
Global:
2-way = 100%

High-risk:
(state, operation, entity_type) = 3-way

Navigation:
(state, operation, inventory_bucket) = 3-way

Reference:
(state, operation, reference_style) = 3-way
```

---

# 38. Coverage Metrics

Do not use raw number of tests as the main confidence metric.

Recommended dashboard:

```text
Golden Journeys
-------------------------
Pass rate                    99.7%
Critical journey pass       100%

Specification Coverage
-------------------------
Domain properties             96%
Critical properties          100%
Behavioral operation types    98%

Graph Coverage
-------------------------
Vertices                     100%
Edges                         99%
Edge pairs                    95%
3-transition sequences        82%

Interaction Coverage
-------------------------
2-way                        100%
3-way                         91%
Critical 3-way               100%

Metamorphic Consistency
-------------------------
Paraphrase invariance         99.4%
Constraint preservation       99.8%
Order invariance              98.9%
Noise robustness              98.1%

Data Correctness
-------------------------
Catalog assertions           100%
Inventory assertions         100%

Mutation Adequacy
-------------------------
Mutation score                87%
Critical-rule mutation        96%

LLM Semantic Evaluation
-------------------------
Goal alignment                94%
Clarification quality         92%
Coherence                     96%
Human/judge agreement         89%
```

---

# 39. Detailed Proposal

## Phase 1 — Write the Independent Behavioral Specification

### Goal

Define what should happen before adding sophisticated generators.

### Work

Document domain-level behaviors such as:

```text
constraint preservation
constraint replacement
reset semantics
pivot semantics
correction semantics
entity compatibility
zero-results recovery
reference resolution expectations
```

Each property should identify its source:

```text
product requirement
business rule
domain axiom
catalog truth
inventory truth
human-labeled standard
```

### Critical rule

Do not automatically derive these properties from the production graph.

---

## Phase 2 — Instrument the Implementation

Expose:

```text
normalized classifier output
active filter state
graph state
transition selected
OpenSearch query
inventory count
rule ID
response metadata
```

These traces are for:

- debugging;
- coverage;
- root-cause analysis.

They should not become the dominant behavioral oracle.

---

## Phase 3 — Attach Properties to Existing Golden Journeys

Before generating new cases, strengthen existing journeys.

After every turn, validate:

```text
domain invariants
catalog validity
inventory correctness
state consistency
```

This immediately increases test power.

---

## Phase 4 — Stateful Property-Based Testing

Use a framework such as Hypothesis.

Generate sequences over:

```text
select
add
remove
replace
correct
pivot
reset
ask
```

Check independent domain properties after every turn.

Persist minimized failures.

---

## Phase 5 — Abstract Model-Based Testing

Build an abstract specification graph separate from the production graph.

Generate high-level behavioral sequences from this model.

Use the production graph only for structural coverage and diagnostics.

---

## Phase 6 — Metamorphic Language Testing

For every semantic operation:

1. Define intended meaning.
2. Generate canonical utterance.
3. Generate paraphrases and controlled mutations.
4. Run through production classifier.
5. Compare resulting semantics.
6. Run downstream system.
7. Validate domain/data properties.

This directly attacks the fuzzy-input problem.

---

## Phase 7 — Combinatorial Coverage

Define high-risk factors.

Generate:

```text
pairwise coverage globally
3-way coverage for critical combinations
```

Turn each abstract combination into:

```text
state setup
semantic operation
language realization
```

---

## Phase 8 — Coverage-Guided Semantic Fuzzing

Seed corpus:

```text
Golden Journeys
production failures
property-test failures
high-value cases
```

Retain generated cases when they produce novel:

```text
state combinations
edge pairs
rule paths
filter-delta signatures
recovery paths
failures
```

---

## Phase 9 — Mutation and Fault Injection

Seed deliberate bugs.

Verify that the evaluation framework detects them.

Create a recurring mutation-quality report.

This guards against the test suite becoming self-congratulatory.

---

## Phase 10 — Add LLM-as-a-Judge Selectively

Use only for:

```text
goal alignment
clarification appropriateness
coherence
subjective helpfulness
unsupported conversational implication
```

Validate judge behavior against human-labeled samples.

Treat low-confidence judge failures as investigation signals rather than automatic hard failures.

---

# 40. Failure Lifecycle

```text
Generated / Production Failure
            |
            v
      Minimize / Shrink
            |
            v
       Root-Cause Class
            |
  +---------+---------+---------+---------+
  |         |         |         |         |
Classifier Mapping   State     Graph    Search/NLG
            |
            v
  Permanent Regression Case
            |
            v
Ask:
"What broader property or generator was missing?"
            |
            v
Update Domain Property / Generator / Coverage Model
```

The goal is not merely to fix examples.

The goal is to learn new **classes of failure**.

---

# 41. Recommended Tooling

## Hypothesis

Use for:

- stateful property-based testing;
- generated action sequences;
- preconditions;
- invariants;
- shrinking.

Reference: Hypothesis stateful testing documentation.

---

## GraphWalker

Use or study for:

- directed graph testing;
- state/transition traversal;
- coverage goals;
- seed-based reproducibility.

Use it primarily for structural graph coverage unless the graph is an independent specification model.

---

## NIST ACTS

Use for:

- pairwise testing;
- t-way covering arrays;
- constrained factor combinations.

---

## Custom Semantic Generator

This is likely one of the highest-value components.

It should understand:

```text
make
model
trim
body style
powertrain
feature
price
inventory
conversation operation
```

The generator should create **meaning first** and language second.

---

## Mutation Testing Framework

Choose language-appropriate mutation tooling.

Use mutation testing primarily on:

```text
navigation
filter preservation
boundary logic
fallback logic
zero-results handling
```

---

# 42. Recommended Priority

## Priority 1

**Independent behavioral specification**

Without this, the framework risks circular validation.

---

## Priority 2

**Deterministic oracles and observability**

Needed for reliable evaluation and debugging.

---

## Priority 3

**Stateful Property-Based Testing**

Best first automated discovery tool for multi-turn failures.

---

## Priority 4

**Metamorphic Testing**

Best attack on linguistic variability.

---

## Priority 5

**Abstract Model-Based Testing**

Provides systematic behavioral traversal independent of implementation detail.

---

## Priority 6

**Combinatorial Coverage**

Provides measurable interaction coverage.

---

## Priority 7

**Semantic Fuzzing**

Continuously searches for unknown behavior.

---

## Priority 8

**Mutation Testing**

Tests the quality of the test suite itself.

---

## Priority 9

**LLM-as-a-Judge**

Adds semantic quality coverage where deterministic methods stop.

---

# 43. Proposed End State

```text
                         PRODUCT INTENT
                              |
                     DOMAIN SPECIFICATION
                              |
          +-------------------+-------------------+
          |                                       |
  Behavioral Properties                    Human Calibration
          |                                       |
    +-----+---------+-------------+---------------+
    |               |             |
Property Tests  Metamorphic   LLM Judge
    |               |             |
    +-------+-------+-------------+
            |
            v
      SYSTEM UNDER TEST
            |
            v
   IMPLEMENTATION TRACE
            |
 +----------+-----------+-----------+
 |          |           |           |
Graph     Rules       Code      Runtime Data
Coverage Coverage   Coverage     Diagnostics
```

Across it all:

```text
        GENERATED TEST SEARCH
                |
   +------------+------------+
   |            |            |
Stateful     Semantic    Combinatorial
Sequences     Fuzzing       Cases
```

And externally:

```text
      MUTATION / FAULT-INJECTION
                |
                v
   "Can the test system detect
       deliberately seeded bugs?"
```

---

# 44. Key Takeaways

The system has two distinct testing challenges:

1. **The conversation space is too large to enumerate manually.**
2. **The testing framework can accidentally overfit to the implementation and validate incorrect behavior.**

The solution is not merely "more test cases."

The solution is architectural.

The testing system should distinguish:

```text
what the product should do
```

from:

```text
how the current code happens to do it
```

The strongest strategy is therefore:

1. Golden Journeys for known critical regressions.
2. Independent domain properties for behavioral truth.
3. Stateful property testing for unexplored multi-turn sequences.
4. Metamorphic testing for linguistic equivalence and controlled semantic changes.
5. Abstract Model-Based Testing for behavioral state coverage.
6. Production graph coverage for implementation coverage only.
7. Combinatorial testing for interaction coverage.
8. Semantic fuzzing for unknown-case discovery.
9. Mutation testing to prove the suite can detect seeded errors.
10. LLM-as-a-Judge only for semantic qualities that stronger deterministic oracles cannot assess.

The central principle is:

> **The production pipeline should be an object of observation, not the source of truth for the behavioral specification.**

And the desired end state is:

> **Generate broad and adversarial conversational behavior, judge it against independent product/domain properties, use deterministic sources wherever possible, use LLM judging only for genuinely subjective semantics, and continuously test whether the test suite itself can detect known classes of failure.**

---

# References

**[1] Microsoft — Model-Based Testing: An Introduction to Model-Based Testing and Spec Explorer**  
Microsoft Learn / MSDN Magazine.  
https://learn.microsoft.com/en-us/archive/msdn-magazine/2013/december/model-based-testing-an-introduction-to-model-based-testing-and-spec-explorer

**[2] GraphWalker — Model-Based Testing Documentation**  
https://graphwalker.github.io/graphwalker-rs/

**[3] Hypothesis — Stateful Testing / RuleBasedStateMachine**  
https://hypothesis.readthedocs.io/en/latest/stateful.html

**[4] Chen, T. Y., Kuo, F.-C., Liu, H., Poon, P.-L., Towey, D., Tse, T. H., & Zhou, Z. Q. — "Metamorphic Testing: A Review of Challenges and Opportunities"**  
ACM Computing Surveys, 2018.  
https://i.cs.hku.hk/~tse/Papers/2010s/hlmtCSUR.html

**[5] Kuhn, D. R., Kacker, R. N., & Lei, Y. — "Combinatorial Testing"**  
NIST.  
https://www.nist.gov/publications/combinatorial-testing

**[6] NIST — Combinatorial Testing / ACTS**  
https://www.nist.gov/programs-projects/combinatorial-testing  
https://csrc.nist.gov/projects/automated-combinatorial-testing-for-software/acts-library/covering-array-library

**[7] LLVM — libFuzzer: Coverage-Guided Fuzz Testing**  
https://llvm.org/docs/LibFuzzer.html

**[8] Zeller, A., Gopinath, R., Böhme, M., Fraser, G., & Holler, C. — The Fuzzing Book**  
https://www.fuzzingbook.org/

**[9] The Fuzzing Book — Fuzzing with Grammars**  
https://www.fuzzingbook.org/html/Grammars

**[10] PIT — Mutation Testing**  
https://pitest.org/  
https://pitest.org/quickstart/basic_concepts/

**[11] McKeeman, W. M. — "Differential Testing for Software"**  
Digital Technical Journal, 1998.  
https://dblp.org/rec/journals/dtj/McKeeman98.html

**[12] Black, P. E., Ammann, P. E., & Ding, W. — "Model Checkers in Software Testing"**  
NISTIR 6777.  
https://www.nist.gov/publications/model-checkers-software-testing

**[13] NIST — Guidelines on Minimum Standards for Developer Verification of Software**  
https://www.nist.gov/publications/guidelines-minimum-standards-developer-verification-software

**[14] NIST — Test Generation Using Model Checking and Specification Mutation**  
https://www.nist.gov/publications/test-generation-using-model-checking-and-specification-mutation

**[15] NIST — Metamorphic Testing for Cybersecurity**  
https://www.nist.gov/publications/metamorphic-testing-cybersecurity

**[16] OpenAI — ARGO: Reward Model Interpretation and Human-Designed Rubrics**  
https://alignment.openai.com/argo/

---

# Suggested Reading Order

For this architecture, the recommended reading sequence is:

1. **Hypothesis Stateful Testing** — practical pattern for generated multi-turn state sequences.
2. **Metamorphic Testing review** — strongest conceptual basis for language equivalence testing.
3. **NIST Combinatorial Testing** — interaction coverage.
4. **GraphWalker** — model/state traversal and structural coverage.
5. **The Fuzzing Book** — semantic and grammar-based generation concepts.
6. **Mutation Testing** — validating the effectiveness of the eval system itself.
7. **NIST developer verification guidance** — useful grounding for combining black-box and structural testing.
8. **LLM rubric-based judging research** — only after the deterministic oracle layers are established.
