# Digital Loan Origination & Lifecycle Management Platform
**(Architecture & System Design Repository)**

---

## 1. Overview

This repository defines the **end-to-end system architecture and engineering design** for a production-grade, fully digital **Personal Loan Origination & Lifecycle Management Platform** for **salaried customers**.

The platform is intentionally designed to mirror **real-world fintech lending systems**, addressing:
- complex multi-stage loan journeys
- unreliable third-party vendor integrations
- long-running workflows with retries
- strict audit and compliance requirements
- frequent business and policy changes **without redeployments**

⚠️ **Important**  
This repository focuses on **architecture, boundaries, and decisions**.  
It deliberately does **not** contain application code.

This is the **source of truth** for *how the system is supposed to work* before any code is written.

---

## 2. Product Scope

### 2.1 In Scope

- Digital loan origination for **salaried customers only**
- Fully automated, end-to-end loan journey:
    - Customer onboarding & OTP
    - KYC & identity verification
    - Income & employment verification
    - Credit bureau checks
    - Eligibility computation (FOIR / IIR)
    - Offer generation
    - Loan sanction & eSign
    - Bank verification & NACH setup
    - Loan disbursement (**mocked**)
- Customer-facing APIs
- Fully automated system-driven decisions
- **Event-driven orchestration** using **Camunda**
- **Immutable audit trail** for every decision and action

---

### 2.2 Explicit Out of Scope (Non-Goals)

The following are **intentionally excluded** to protect scope and clarity:

- ML-based credit scoring
- Physical document handling
- Manual underwriting UIs
- Collections & recovery
- Real money movement via bank rails  
  (all payments & disbursements are **mocked via vendor adapters**)

These exclusions ensure the project stays focused on **core backend system design**, not peripheral complexity.

---

## 3. Target Audience

This repository is written for:

- Backend engineers
- Senior / Staff system design interview panels
- Recruiters evaluating real-world backend capability
- Engineers looking to understand **how fintech lending systems are actually built**

The content prioritizes:
- correctness
- explicit trade-offs
- operational realism
- long-term maintainability

---

## 4. Core Engineering Principles (Non-Negotiable)

### 4.1 Vendor Abstraction via SDKs
- Business logic never talks directly to vendors
- All vendors are accessed through internal SDKs
- SDKs support **mock + real adapters**
- Vendor switching does not impact domain services

---

### 4.2 Event-Driven Loan Journey
- Loan progression is driven by **domain events**
- Services are loosely coupled
- Partial failures are first-class citizens
- No hidden synchronous chains

---

### 4.3 Saga-Based Orchestration
- Long-running workflows are orchestrated using **Camunda**
- Explicit retries, waits, and compensations
- Automated decision enforcement
- No custom orchestration code

---

### 4.4 Database as Source of Truth
- Databases are authoritative
- Events are **derived facts**, not state
- Transactional Outbox ensures consistency
- Safe replay without corruption

---

### 4.5 Full Audit & Explainability
- Every state transition is auditable
- Every vendor response is captured (masked)
- Every system action is recorded
- Rejection reasons are explainable and immutable

---

### 4.6 Policy Over Code
- Platform configuration lives in `config-service`
- Product & eligibility rules live in **Camunda**
- No redeploys for business rule changes
- New configs apply only to **new loan applications**

---

## 5. High-Level Loan Journey

``` mermaid
graph TD
    %% Node Definitions
    Start((Start))
    Apply[Apply Loan]
    Profile["Profile & OTP Verification"]
    KYC["KYC (Identity)"]
    Income["Income & Employment Verification"]
    Bureau["Bureau Check (Credit Score)"]
    Eval{Eligibility Evaluation}
    Offer[Offer Generation]
    Accept["Offer Acceptance (User)"]
    Sanction["Sanction & eSign"]
    NACH["Bank Verification & NACH"]
    Disburse([Disbursement - Mocked])
    Reject([Loan Rejected])

    %% Flow Connections
    Start --> Apply
    Apply --> Profile
    Profile --> KYC
    KYC --> Income
    Income --> Bureau
    Bureau --> Eval

    Eval -->|Eligible| Offer
    Eval -->|Ineligible| Reject

    Offer --> Accept
    Accept --> Sanction
    Sanction --> NACH
    NACH --> Disburse

    %% Styling
    style Disburse fill:#00c853,stroke:#333,stroke-width:2px,color:#fff
    style Reject fill:#ff5252,stroke:#333,stroke-width:2px,color:#fff
    style Eval fill:#fff9c4,stroke:#fbc02d
```


### Journey Characteristics
- Sequential, fail-fast execution
- Each stage can fail independently
- Completed stages are **never rolled back**
- Rejection is **terminal per loan application**
- Re-application always creates a **new loan ID**

---

## 6. Architecture Overview

The platform follows a **microservices + event-driven** architecture with:

- Service-owned databases
- Kafka as asynchronous backbone
- Camunda for saga orchestration
- Strict separation of:
  - state ownership
  - orchestration
  - vendor execution

High-level diagrams include:
- C4 Context & Container diagrams
- Business flow diagrams
- Retry & failure flows
- Critical sequence diagrams

All diagrams are maintained as **Mermaid source** under `docs/diagrams/`.

---

## 7. Failure & Retry Philosophy

- All failures are **retryable first**
- Short retries handled at service level
- Long retries handled by workflow (Camunda)
- No rollback of successful stages
- Loan rejected only after retries exhaust or business rules fail
- System-driven decisions, fully automated

Failures are expected and explicitly modeled.

---

## 8. Security & Compliance Model

- Aadhaar data is **never stored**
- All PII is encrypted at rest and in transit
- Field-level encryption for sensitive data
- Secure authentication for API access
- Engineers have **no access** to production PII
- Immutable, append-only audit logs
- Long-term archival for compliance

Security is treated as a **system property**, not a feature.

---

## 9. Architecture Decision Records (ADRs)

All major architectural decisions are captured under:
docs/architecture-decisions/


Key ADRs include:
- ADR-001: Architecture style & system boundaries
- ADR-002: Eventing model & state consistency

ADRs document **why decisions were made**, not just what was built.

---

## 10. Repository Contents

This repository contains:

- Architecture overview documents
- Service landscape & boundaries
- Eventing model
- Failure strategy
- Security & compliance model
- Config-service design
- Architecture Decision Records
- Mermaid diagrams (C4, flow, sequence)

This repository is the **single source of truth** for system design.

---

## 11. What This Repository Does NOT Contain

- Spring Boot implementations
- API contracts
- Infrastructure code
- CI/CD pipelines
- Vendor credentials or secrets

Those will live in **separate repositories**.

---

## 12. Next Phase (Hands-On Implementation)

This repository is now **complete and frozen**.

🔜 **Next step (starting tomorrow)**:
- Fork into implementation repos
- Define API contracts
- Design event schemas
- Start coding services incrementally

Architecture first.  
Code second.  
No shortcuts.

---

### Final Note

> *“Good systems fail safely.  
> Great systems explain why.”*

You’ve designed a system that does both.
