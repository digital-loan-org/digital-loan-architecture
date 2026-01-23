# ADR-001: Architecture Style & System Boundaries

## Status
Accepted

---

## Context

The Digital Loan Origination Platform is a regulated, vendor-heavy system that
implements a long-running personal loan journey involving KYC, income checks,
bureau evaluation, eligibility rules, sanction, and disbursement.

Key constraints influencing the architecture:
- heavy dependence on external vendors
- long-running and retryable workflows
- strict audit and traceability requirements
- frequent business and policy changes
- need for failure isolation and safe retries

An early architectural decision was required to choose:
- overall system style
- communication model
- orchestration approach
- source of truth strategy

This ADR records the reasoning behind those foundational choices.

---

## Decision

We decided to build the platform using:

- **Microservices architecture**
- **Event-driven communication (Kafka)**
- **Hybrid synchronous + asynchronous execution**
- **Camunda-based saga orchestration**
- **Database as the single source of truth (not event sourcing)**

These decisions are intentional and mutually reinforcing.

---

## Rationale

### 1. Why Microservices (and not a Monolith)

Microservices were chosen primarily for:

- **Vendor isolation**  
  Each external dependency (KYC, income, bureau, NACH) is isolated behind a
  dedicated service boundary to prevent vendor-specific failures from cascading.

- **Independent deployment**  
  Services can be updated, fixed, or scaled independently without redeploying
  the entire platform.

A secondary motivation is **learning and skill growth**, allowing deep exposure
to real-world distributed system patterns used in modern fintech platforms.

---

### 2. Why Event-Driven Communication (Kafka)

Kafka is used to decouple services and manage asynchronous progression of the
loan journey.

Primary drivers:
- **Decoupling of services**  
  Producers do not need to know or depend on consumers.

- **Long-running workflows**  
  Loan journeys span minutes to hours with retries, waits, and compensations.

Eventing also enables future analytics and observability without impacting core
business flows.

---

### 3. Why Camunda for Orchestration (and not custom Kafka listeners)

Camunda was selected as the workflow orchestration engine to implement saga-style
coordination.

Key reasons:
- **Saga compensation support**  
  Explicit modeling of retries, waits, and compensating actions is required in
  lending workflows.

- **Business-driven change without redeployment**  
  Workflow behavior and sequencing can change without code changes, aligning
  with policy-over-code principles.

Camunda avoids the need to build and maintain a custom orchestration framework,
which would add risk and complexity.

---

### 4. Why Database as Source of Truth (Not Event Sourcing)

We explicitly chose **not** to use event sourcing.

Reason:
> After emitting an event, if database insertion fails, the system enters an
> inconsistent and unauditable state.

By persisting state **first** and emitting events **after commit**, the platform:
- guarantees audit correctness
- ensures analytics and services rely on authoritative data
- avoids complex event replay semantics

Events are treated as **derived facts**, not primary state.

---

### 5. Why Hybrid Sync + Async (Not 100% Async)

A pure asynchronous model was intentionally avoided.

Hybrid execution is used because:
- **Rule evaluation is deterministic and fast**, making synchronous execution
  simpler and safer.
- **Saga compensation logic benefits from async orchestration**, especially when
  interacting with unreliable vendors.

This balance reduces latency where possible while preserving robustness where
needed.

---

## Alternatives Considered

### Modular Monolith
Rejected due to limited failure isolation and vendor blast-radius concerns.

### Event Sourcing + CQRS
Rejected due to:
- higher operational complexity
- steeper learning curve
- increased audit and replay risk in a regulated domain

### Custom Workflow Engine
Rejected due to:
- high implementation cost
- risk of subtle bugs
- lack of operational visibility

---

## Consequences

### Positive
- Strong failure isolation
- Clear ownership boundaries
- Safe handling of long-running workflows
- High auditability and traceability
- Real-world, production-grade architecture

### Negative (Accepted Trade-offs)
- **More infrastructure components** (Kafka, Camunda, multiple services)
- **Higher conceptual complexity**, especially around saga compensation

These downsides are accepted to meet domain and regulatory requirements.

---

## Final Notes

This ADR represents a **foundational decision**.

Any change to this ADR would require re-evaluating:
- service boundaries
- eventing model
- workflow design
- failure handling strategy

As such, this decision is intended to be stable and long-lived.
