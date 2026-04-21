# GBPS Charge Service Implementation

## Solution Architecture and Integration Note for SCB

## 1. Executive Summary

This document presents the proposed architecture for the new **GBPS Charge Service**, which is intended to replace the current **eDMI-based charge derivation approach** used in the RPE flow.

At a high level, the new service is designed to make charge evaluation:

* easier to understand,
* easier to audit,
* faster to execute at scale, and
* easier to evolve as GBPS charge rules change over time.

Instead of relying on logic embedded in or tightly coupled with eDMI, the proposed design introduces a dedicated microservice that:

* receives and stores charge rules published from GBPS,
* evaluates incoming transaction contexts against those rules using an indexed rule engine, and
* calculates the final charge using a controlled and extensible calculation framework.

For SCB, this creates a more transparent and supportable operating model. It also positions the platform for future charge-rule growth without repeatedly reworking the runtime integration pattern.

## 2. Background and Problem Statement

The current charge derivation approach depends on eDMI. While this may have been sufficient in the initial stages, it creates several practical issues as the volume of rules, change requests, and operational expectations increase.

### Current limitations

* The charge logic is tightly coupled with an external component, which reduces flexibility.
* Rule execution is not sufficiently transparent for troubleshooting and support teams.
* Auditing a charge outcome can be difficult because the evaluation path is not easily visible.
* Performance can become a concern when the number of rules increases.
* Introducing new rule patterns or exceptions becomes harder over time.

From an SCB architecture perspective, the main concern is not only technical replacement, but also whether the new solution improves operational clarity, resilience, and maintainability. This proposed design aims to address those concerns directly.

## 3. Proposed Target Architecture

The target solution introduces a standalone **Charge Evaluation Microservice** between the RPE transaction flow and the persisted GBPS rule set.

### 3.1 High-level architecture

Rule synchronization flow:

`GBPS -> Solace Topic -> Charge Service -> rpe_charge_db`

Runtime evaluation flow:

`RPE TT -> Charge Service -> Rule Evaluation -> Charge Calculation -> Response`

### 3.2 Core components and responsibilities

| Component | Responsibility |
| --- | --- |
| GBPS | Publishes charge rule data as JSON events |
| Solace | Transports rule change events reliably to downstream consumers |
| Charge Service | Ingests rules, validates payloads, stores rule data, evaluates requests, and computes charges |
| RPE TT | Invokes the charge evaluation service during transaction processing |
| rpe_charge_db | Stores synchronized charge rules and supporting metadata |

### 3.3 Architectural intent

The design separates rule lifecycle management from runtime transaction handling. This is important because rule ingestion and rule execution have very different characteristics:

* ingestion must be reliable, idempotent, and traceable,
* runtime evaluation must be fast, deterministic, and easy to explain.

By separating these concerns, the solution becomes easier to operate and easier to scale.

## 4. Key Architecture Decisions

## 4.1 Decision 1: Use a GBPS-aligned data model

### Decision

The Charge Service database will largely mirror the GBPS rule structure, while introducing surrogate technical identifiers where necessary for internal processing.

### Why this was chosen

This approach keeps the persisted model close to the upstream source structure. That reduces transformation effort and makes it easier for support teams, developers, and analysts to reconcile what GBPS sent versus what the Charge Service stored and evaluated.

### Benefits

* Lower transformation complexity during ingestion
* Easier reconciliation between GBPS payloads and database records
* Reduced risk of semantic drift between source and target
* Better debugging experience when investigating mismatches

### Trade-offs

| Benefit | Trade-off |
| --- | --- |
| Simplified ingestion | Lower freedom to redesign the schema for internal optimization |
| Easier debugging | Composite business keys may still be complex |
| Strong alignment with GBPS | Some queries may require additional joins or indexing care |

### Architecture note

For SCB review, this decision is sensible if operational traceability is valued more highly than aggressive schema normalization. That appears to be the right trade-off for this use case.

## 4.2 Decision 2: Use a custom indexed rule evaluation engine

### Options considered

Rejected option:

* Spring Expression Language (SpEL)

Selected option:

* Custom manual indexed rule engine

### Why SpEL was not preferred

Although SpEL offers flexibility, it introduces avoidable risks for this use case:

* more runtime overhead,
* less predictable optimization opportunities,
* weaker control over supported operators,
* greater debugging complexity for production support.

### Why a custom engine was selected

The charge-evaluation problem is bounded and domain-specific. Because of that, a controlled custom evaluator can deliver better performance and better operational transparency than a general-purpose expression engine.

| Factor | SpEL | Manual Indexed Engine |
| --- | --- | --- |
| Performance | Moderate | High |
| Scalability | Limited at high rule volume | Strong |
| Debuggability | Limited | Detailed and controllable |
| Security | Broader execution surface | Restricted operator model |
| Optimization potential | Low | High |

### Architecture note

This is one of the strongest decisions in the document. For an SCB enterprise platform, the ability to explain exactly why a rule matched or failed is as important as raw speed.

## 5. Rule Evaluation Engine Design

## 5.1 Request context model

Incoming requests are transformed into a flattened evaluation context before rule matching begins.

Example:

`header.pmtTp = TT`

`data.chrgsInf.chrgBr = DEBT`

This flattened representation provides a consistent structure for indexing, comparison, and logging.

## 5.2 Index structure

The proposed primary index model is:

`Map<path, Map<value, List<ruleId>>>`

Example:

`header.pmtTp -> TT -> [rule1, rule2]`

This allows the engine to quickly identify candidate rules based on known field-value combinations before performing deeper checks.

## 5.3 Evaluation flow

The evaluation flow is expected to work as follows:

1. Extract and flatten the incoming request context.
2. Use indexed fields to retrieve candidate rule sets.
3. Intersect those rule sets for mandatory AND-based conditions.
4. Perform exact evaluation for conditions that cannot be fully indexed.
5. Select the final matching rule or rules.

### Conditions that need final-stage evaluation

* OR conditions
* `!=` conditions
* cross-field comparisons
* dynamic or derived values

### Why this matters

This two-stage approach avoids evaluating every rule for every request. Instead, the engine narrows the search space first and then performs precise validation only on likely candidates.

That is the main reason the design is expected to scale better than a naive rule-by-rule evaluator.

## 6. Advanced Rule Handling

Real-world charge rules rarely stay limited to simple equality checks. The design already anticipates this and introduces handling for more complex patterns.

## 6.1 Nested path handling

Example:

`data.cbs.enqryInf.CHRG.casaSgmntIdr`

Approach:

* normalize nested JSON paths into a consistent dot-notation model

This is necessary so both ingestion and evaluation use the same rule-path vocabulary.

## 6.2 List handling

Example:

`data.txSttImInf.dbtSttLmAcct[0].ccy`

Approach:

* flatten indexed list elements explicitly

Example flattened value:

`dbtSttLmAcct[0].ccy = LKR`

Future enhancement:

`dbtSttLmAcct[*].ccy`

### Architecture note

This is a good starting point, but list semantics often become a source of ambiguity in enterprise rule engines. If wildcard support is eventually added, the team should clearly define whether matching means:

* any element matches,
* all elements match, or
* first matching element wins.

## 6.3 Cross-field comparison

Example:

`dbt.ccy != cdt.ccy`

Approach:

* resolve both paths dynamically at evaluation time
* compare the resolved values using supported operators

This is important for rules where the charge depends on relationships between fields rather than fixed literal values.

## 6.4 OR-condition handling

The current design proposes that indexing is primarily applied to AND-based conditions, while OR clauses are evaluated in the final stage.

This is a practical decision because OR-heavy logic is harder to index cleanly without increasing index complexity and memory consumption.

## 6.5 Non-indexable conditions

The following condition types will be handled in the final validation stage:

* `!=`
* cross-field comparisons
* dynamic values
* conditions requiring runtime interpretation

## 7. Charge Calculation Engine

After rule selection, the service will calculate the charge amount using a dedicated calculation layer.

### Design pattern

The proposed approach is a **Strategy Pattern** with factory-based resolution.

### Supported calculation types

* Fixed charge
* Percentage-based charge
* Tiered or slab-based charge

### Calculation flow

`Rule -> Strategy Resolver -> Calculation -> Result`

### Why this is appropriate

This keeps charge calculation logic separate from rule matching logic. That separation improves maintainability because new calculation types can be added without changing the evaluation engine itself.

For SCB, this also makes testing easier because rule selection and charge computation can be validated independently.

## 8. Data Ingestion and Synchronization

## 8.1 Source and transport

* Source system: GBPS
* Event transport: Solace
* Payload format: JSON

## 8.2 Supported event operations

* Insert
* Update
* Delete

## 8.3 Ingestion expectations

The ingestion flow should support:

* schema validation,
* idempotent event handling,
* safe replay,
* retry handling,
* operational logging.

### Architecture note

This section is directionally correct, but it would benefit from more detail for formal sign-off. In particular, SCB architects are likely to ask how the service behaves during:

* duplicate event delivery,
* out-of-order events,
* temporary database unavailability,
* partial ingestion failure,
* Solace consumer restart or replay scenarios.

Those behaviors should be made explicit in the next revision.

## 9. Non-Functional Considerations

## 9.1 Performance

The main performance advantage comes from indexed filtering, which reduces the evaluation scope from all rules to a much smaller candidate set.

Conceptually:

`O(N) -> O(K)` where `K << N`

That said, the document would be stronger if it included target non-functional numbers such as:

* expected maximum rule count,
* target average evaluation latency,
* target p95 and p99 latency,
* maximum acceptable synchronization lag.

## 9.2 Scalability

The service is expected to be stateless from a runtime processing perspective, which supports horizontal scaling. This is a good fit for containerized deployment and microservice-based operating models.

## 9.3 Reliability

Reliability is expected to come from:

* event-driven rule synchronization,
* retry handling,
* idempotent processing,
* graceful degradation where possible.

However, graceful degradation should be defined more precisely. For example, the document should state whether charge evaluation should:

* fail closed,
* return a default response,
* fall back to a previous rule snapshot,
* or route the transaction for manual exception handling.

## 9.4 Observability

The proposed observability model is good and should include:

Logs:

* matched rule identifiers,
* rejected conditions,
* processing errors,
* rule version references.

Metrics:

* evaluation latency,
* candidate rule count,
* rule-ingestion success and failure counts,
* synchronization lag,
* cache or index rebuild duration if applicable.

## 9.5 Security

The design intentionally avoids dynamic execution and limits the allowed operator set. This is a strong design choice because it keeps the execution surface controlled and predictable.

For SCB review, this should also mention:

* service-to-service authentication,
* payload validation,
* audit logging retention,
* access control for operational users,
* data classification impact if sensitive fields are logged.

## 10. Risks and Mitigations

| Risk | Mitigation |
| --- | --- |
| Rule complexity grows over time | Keep a controlled rule DSL and governance process |
| Index memory overhead becomes significant | Use selective indexing and monitor rule distribution |
| List handling becomes ambiguous | Define strict semantics for indexed and wildcard list evaluation |
| Synchronization inconsistencies occur | Use idempotent updates and version-aware processing |
| Debugging becomes difficult in production | Provide detailed evaluation logs and rule traceability |

## 11. Migration Approach

The migration should be phased to reduce delivery and operational risk.

### Phase 1: Build ingestion capability

* consume GBPS events from Solace,
* validate payloads,
* persist rules into `rpe_charge_db`.

### Phase 2: Implement the rule evaluator

* flatten request context,
* support core operators,
* validate matching behavior against sample rules.

### Phase 3: Introduce indexed evaluation

* build selective indexes,
* reduce candidate set size,
* benchmark performance under realistic volume.

### Phase 4: Add charge calculation strategies

* implement fixed, percentage, and slab-based calculators,
* validate calculation correctness.

### Phase 5: Replace eDMI-driven derivation

* perform side-by-side validation,
* compare production-equivalent outcomes,
* cut over once confidence thresholds are met.

### Architecture note

For SCB, the migration plan would be more convincing if it also included a formal parallel-run stage, rollback criteria, and production-readiness gates.

## 12. Key Open Questions

The following items should be clarified before final architecture approval:

* Will wildcard list support such as `[*]` be required in phase 1 or only later?
* How will rule priority be handled when multiple rules match?
* What is the tie-break rule if more than one candidate remains valid?
* Are there limits on index size, rule volume, or payload size?
* What is the acceptable synchronization SLA between GBPS and the Charge Service?
* Is there a requirement for rule versioning and effective-dating?
* How should the service behave if no rule matches?
* Is there a fallback expectation during GBPS sync delays or failures?

## 13. Important Missing Parts to Add Before Formal Sign-off

The current design is strong conceptually, but the following areas are still missing or underdefined for an architecture document intended for review by SCB architects.

## 13.1 Rule governance model

The document should define:

* who owns charge rule creation,
* who approves rule changes,
* how rule versions are tracked,
* how invalid or conflicting rules are prevented from reaching production.

## 13.2 Rule conflict and priority handling

The document should state clearly:

* whether first-match, best-match, or priority-based selection will be used,
* how overlapping rules are detected,
* what happens when multiple rules are equally valid.

## 13.3 Versioning and effective dating

This is especially important for banking systems. The design should clarify:

* whether rules have effective-from and effective-to timestamps,
* how future-dated rules are handled,
* how historical rule evaluation can be audited.

## 13.4 Failure and fallback behavior

The document should explain:

* what happens when the rule database is unavailable,
* what happens when the in-memory index is stale or partially rebuilt,
* whether the service can continue using the last known good rule set,
* what response is returned to calling systems during failure scenarios.

## 13.5 Deployment and runtime topology

The document should include:

* deployment model,
* HA expectations,
* scaling strategy,
* whether indexes are in-memory per node,
* startup and warm-up behavior,
* expected recovery behavior after restart.

## 13.6 Testing strategy

A formal implementation note should also cover:

* unit testing of evaluator conditions,
* contract testing of GBPS payload ingestion,
* performance testing under realistic rule volume,
* regression testing during migration from eDMI,
* replay testing using production-like payload samples.

## 13.7 Operational support model

The document should state how support teams will:

* trace why a rule matched,
* identify the active rule version,
* reprocess failed sync events,
* monitor sync lag and evaluation health.

## 13.8 Security and audit expectations

For a bank-grade deployment, the document should capture:

* authentication and authorization model,
* audit trail retention requirements,
* masking rules for sensitive logged fields,
* operational access boundaries.

## 14. Final Recommendation

The proposed architecture is a strong direction for replacing eDMI-based charge derivation.

Its main strengths are:

* strong alignment with GBPS as the source of truth,
* better runtime performance through indexed evaluation,
* improved transparency and auditability,
* better separation between rule synchronization, rule evaluation, and charge calculation,
* a clearer path for long-term enhancement.

For SCB, this should be presented as a solid target architecture, with the recommendation that formal approval be tied to closure of the missing design areas listed above, especially:

* rule conflict handling,
* failure behavior,
* versioning and effective dating,
* target NFR metrics,
* operational support design.

## 15. Conclusion

This design establishes a credible foundation for a **robust, scalable, and enterprise-ready GBPS Charge Service**.

It improves on the current eDMI-based approach by making rule execution more transparent, more maintainable, and more scalable. Just as importantly, it creates a cleaner integration boundary between GBPS, RPE, and charge computation.

With the missing areas refined in the next iteration, this document can evolve from a good technical concept note into a strong architecture submission suitable for SCB review and sign-off.
