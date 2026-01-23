# ADR-002: Eventing Model & State Consistency Strategy

## Status
Accepted

---

## Context

The Digital Loan Origination Platform is a distributed system with:
- multiple independently deployed services
- long-running workflows
- external vendor integrations
- strict audit and compliance requirements

Services need to communicate asynchronously while ensuring:
- state consistency
- audit correctness
- safe retries
- replay capability without data corruption

A decision was required on:
- event schema design
- event emission timing
- source of truth
- failure handling
- replay control

---

## Decision

We decided to adopt the following eventing strategy:

- **Generic event envelope** for all domain events
- **Database as the single source of truth**
- **Transactional Outbox Pattern** for event emission
- **Immutable, append-only events**
- **Domain-oriented Kafka topics**
- **Per-service DLQs**
- **Engineering-controlled replay only**

This model intentionally avoids full event sourcing.

---

## Rationale

### 1. Generic Event Envelope

All events use a common structure with event-specific payloads.

Reasons:
- simplifies tooling and consumers
- avoids schema explosion
- supports easy replay and audit
- reduces coupling between services

Strong typing is enforced at service boundaries, not at the broker level.

---

### 2. Database as Source of Truth

The database is authoritative for:
- loan state
- decisions
- eligibility outcomes

Events are treated as **derived facts**, not primary state.

Reasoning:
- emitting events before DB commit can lead to inconsistency
- audits and analytics must rely on persisted data
- DB-first ensures correctness under failure

---

### 3. Transactional Outbox Pattern

Events are emitted using a transactional outbox.

Why:
- guarantees no event without committed state
- guarantees eventual event delivery
- survives crashes and restarts
- avoids dual-write problems

This approach is simpler and safer than distributed transactions.

---

### 4. Immutable Events

Events are never modified or deleted.

Corrections are expressed using new events (e.g. revocation or versioned events).

This preserves:
- full decision history
- regulatory traceability
- temporal correctness

---

### 5. Topic Strategy

Kafka topics are **domain-oriented**, not service-oriented.

This allows:
- multiple consumers per domain
- easier onboarding of analytics and monitoring
- separation of concerns

---

### 6. Dead Letter Queues (DLQ)

Each service owns its own DLQ.

Reasons:
- clear operational ownership
- isolated blast radius
- service-specific retry policies

Shared DLQs were explicitly rejected.

---

### 7. Replay Policy

Event replay is:
- engineering-controlled only
- executed via controlled consumer groups
- scoped and audited

Manual or UI-triggered replay is explicitly forbidden to avoid silent
data corruption.

---

## Alternatives Considered

### Full Event Sourcing
Rejected due to:
- higher operational complexity
- steeper learning curve
- increased audit and replay risk
- unnecessary complexity for the domain

### Event-Specific Schemas (Avro + Registry)
Rejected due to:
- schema versioning overhead
- slower iteration
- limited additional value at this stage

### Emitting Events Before DB Commit
Rejected due to:
- risk of inconsistent state
- audit failures
- replay ambiguity

---

## Consequences

### Positive
- strong consistency guarantees
- clear debugging and audit trail
- safe retries and recovery
- simpler mental model

### Negative (Accepted Trade-offs)
- additional outbox infrastructure
- eventual consistency between DB and Kafka
- slightly delayed event visibility

These trade-offs are acceptable for correctness and audit safety.

---

## Final Notes

This ADR complements **ADR-001** and defines how asynchronous communication
works across the platform.

Any change to this decision would require revisiting:
- failure strategy
- workflow orchestration
- audit design
- analytics assumptions

This decision is intended to be stable and long-lived.
