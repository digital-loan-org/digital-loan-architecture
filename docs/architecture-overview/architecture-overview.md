# Architecture Overview

## Architecture Style

The Digital Loan Origination Platform follows a **microservices-based, event-driven architecture** designed to handle long-running financial workflows, third-party dependencies, and regulatory audit requirements.

The system prioritizes:
- independent service ownership
- asynchronous execution where latency and failure are expected
- synchronous execution where deterministic computation is required

---

## High-Level Architecture Characteristics

### Microservices
Each business capability is owned by a dedicated service with:
- clear data ownership
- isolated deployment lifecycle
- well-defined API and event contracts

Services communicate using:
- synchronous REST APIs (commands & reads)
- asynchronous Kafka events (state transitions & side effects)

---

### Event-Driven Design
The loan journey progresses through **events**, not chained service calls.

Examples:
- `kyc.completed`
- `income.verified`
- `bureau.checked`
- `eligibility.calculated`
- `loan.sanctioned`

This enables:
- failure isolation
- retry without cascading impact
- auditability
- workflow replay

---

## Orchestration Strategy

### Hybrid Orchestration Model (Chosen Deliberately)

| Responsibility | Owner |
|--------------|-------|
| Loan application state | `loan-application-service` |
| Journey orchestration | `workflow-service (Camunda)` |
| Business rules | `eligibility-service` |
| Vendor execution | Domain services |

**Why Hybrid?**
- State is always queryable from application service
- Explicit compensation paths
- Safe replay of long-running journeys
- Automated decision enforcement

Camunda drives *when* things happen.  
Services decide *what* happens.

---

## Sync vs Async Execution Model

### Asynchronous (Event-Driven)
Used when:
- vendor latency is unpredictable
- retries are required
- cost control matters

| Stage | Reason |
|----|----|
| KYC | External identity vendors |
| Income verification | Bank aggregators |
| Bureau checks | Rate-limited & paid APIs |
| NACH setup | External mandate providers |

---

### Synchronous (API-Based)
Used when:
- computation is deterministic
- response is required immediately

| Stage | Reason |
|----|----|
| Eligibility calculation | Pure rule evaluation |
| Offer generation | Customer UX |

---

## Data Ownership & Access

- Each service owns its **write database**
- Direct cross-service DB writes are forbidden
- Read access is exposed via APIs
- Frontend may call multiple services via API Gateway

This avoids:
- tight coupling
- hidden dependencies
- cascading failures

---

## High-Level System Diagram

``` mermaid
graph TD
Client[Client] --> Gateway[API Gateway]
Gateway --> LoanSvc[Loan Application Service]
LoanSvc --> Kafka{Kafka Events}
Kafka --> Workflow[Workflow Service - Camunda]
Workflow --> Domain[Domain Services]
Domain --> Vendors[Vendor SDKs - Mock/Real]
```

---

## Loan Journey Flow (Simplified)

Application Created
-  KYC (async)
- Income (async)
- Bureau (async)
- Eligibility (sync)
- Offer
- Sanction
- NACH
- Disbursement


Each stage:
- emits an event
- persists a decision snapshot
- supports retries or compensation

---

## Critical Design Guarantees
- No service blocks waiting for vendors
- No hidden synchronous chains
- Every decision is traceable
- Failures are first-class citizens
