# Domain Ownership & Responsibilities

## Objective

This document defines **clear domain ownership boundaries** for the Digital Loan Platform.

The goal is to:
- Prevent business logic leakage across services
- Avoid duplicate data ownership
- Ensure auditability and explainability
- Enable safe ops / risk overrides
- Support long-term scalability

---

## Core Principle

> **Data is not scattered. Decisions are distributed.**

### Golden Rule

> If a service makes a business decision, it must own a database.  
> If a service does not own a database, it must not make business decisions.

---

## Service Inventory – High Level View

| Service | Primary Role | Owns Business Decisions | Database | SDKs Used |
|------|-------------|-------------------------|----------|-----------|
| customer-service | Borrower identity | ✅ Yes | PostgreSQL | kyc-sdk |
| application-service | Loan lifecycle | ✅ Yes | PostgreSQL | None |
| income-service | Income & employment truth | ✅ Yes | PostgreSQL | bank-sdk, epfo-sdk |
| credit-service | Credit interpretation | ✅ Yes | PostgreSQL | bureau-sdk |
| eligibility-service | FOIR / IIR computation | ✅ Yes | Optional (Snapshot) | None |
| sanction-service | Legal sanction | ✅ Yes | PostgreSQL + Object Store | esign-sdk |
| disbursement-service | Ops & payout | ✅ Yes | PostgreSQL | bank-sdk, nach-sdk, payment-sdk |
| collection-service | Repayment lifecycle | ✅ Yes | PostgreSQL | payment-sdk |
| workflow-service | Orchestration | ❌ No | Engine DB only | None |
| config-service | Rule storage | ❌ No | PostgreSQL / Redis | None |
| audit-service | Audit trail | ❌ No | Append-only DB | None |
| notification-service | Communication | ❌ No | Optional Outbox | None |

---

## Core Domain Services (Business Owners)

These services **own business meaning** and persist domain data.

---

### 1. customer-service

**Owns**
- Borrower identity
- De-duplication logic

**Uses SDK**
- kyc-sdk

**Database**
- PostgreSQL

**Stores**
- customer_id
- pan_hash
- aadhaar_hash
- mobile
- dob

---

### 2. application-service (Single Source of Truth)

**Owns**
- Loan application lifecycle
- Application state machine
- State transition history

**Database**
- PostgreSQL

**Stores**
- application_id
- customer_id
- current_state
- state_history
- timestamps

**Rules**
- ONLY service allowed to change application state
- Other services may only READ state
- Ops / Risk actions go through secured APIs

---

### 3. income-service

**Owns**
- Income and employment truth

**Uses SDKs**
- bank-sdk
- epfo-sdk

**Database**
- PostgreSQL (Snapshot-based)

**Stores**
- application_id
- monthly_income
- employment_type
- verification_status

**Rules**
- Does not decide eligibility
- Does not update application state

---

### 4. credit-service

**Owns**
- Creditworthiness interpretation

**Uses SDK**
- bureau-sdk

**Database**
- PostgreSQL (Snapshot-based)

**Stores**
- application_id
- credit_score
- dpd_flags
- vintage_months
- ntc_flag

---

### 5. eligibility-service

**Owns**
- Eligibility computation logic

**Responsibilities**
- FOIR calculation
- IIR calculation
- Maximum EMI computation
- Eligibility decision

**Database**
- Optional (Decision snapshot)

**Stores**
- application_id
- foir
- iir
- eligible_amount
- decision

**Rules**
- Must NOT call vendors
- Must NOT update application state

---

### 6. sanction-service

**Owns**
- Legal loan sanction commitment

**Uses SDK**
- esign-sdk

**Database**
- PostgreSQL + Object Storage

**Stores**
- application_id
- sanctioned_amount
- tenure
- interest_rate
- esign_status
- document_reference

---

### 7. disbursement-service (Ops)

**Owns**
- Pre- and post-disbursement operations

**Uses SDKs**
- bank-sdk
- nach-sdk
- payment-sdk

**Database**
- PostgreSQL

**Stores**
- application_id
- bank_verification_status
- nach_status
- disbursement_status

---
### 8. collection-service

**Owns**
- Post-disbursement loan lifecycle

**Uses SDK**
- payment-sdk

**Database**
- PostgreSQL

**Stores**
- loan_account_id
- emi_schedule
- repayment_history
- outstanding_amount

---

## Supporting Services (No Business Ownership)

### workflow-service
- Orchestrates long-running flows using the Camunda rule engine
- Manages retries, compensation, and rule-driven decision enforcement
- Enforces automated Ops/Risk decisioning; normal flows do not require human Ops/Risk intervention
- Owns no business data
- Orchestrates long-running flows
- Manages retries and compensation
- Does not enforce rules

### audit-service
- Stores immutable, masked audit logs
- Records all decisions and overrides

### notification-service
- Sends SMS / Email / Push
- No business logic

---

## SDK Rules (Strict)

SDKs are **stateless libraries**, not services.

| Rule | Enforced |
|----|---------|
| SDKs have no database | ✅ |
| SDKs cannot approve / reject | ✅ |
| SDKs return raw responses only | ✅ |

---

## Non-Negotiable Rules

1. Only application-service can change application state
2. SDKs can never directly approve or reject applications; SDKs return raw responses and services or orchestration rules make decisions
3. No service may access another service’s database
4. Each service stores only what it decides
5. Snapshot data must be recomputable