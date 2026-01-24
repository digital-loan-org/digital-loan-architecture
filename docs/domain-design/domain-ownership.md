# Domain Ownership & Responsibilities

## Objective

This document defines **clear domain ownership boundaries** for the Digital Loan Platform.

The goal is to:
- Prevent business logic leakage across services
- Avoid duplicate data ownership
- Ensure auditability and explainability
- Enable clear service boundaries
- Support long-term scalability

---

## Core Principle

> **Data is not scattered. Decisions are distributed.**

### Golden Rule

> If a service makes a business decision, it must own a database.  
> If a service does not own a database, it must not make business decisions.

---

## Service Inventory – High Level View (FINAL - Post Review)

| Service | Primary Role | Owns Business Decisions | Database | SDKs Used | Status |
|------|-------------|-------------------------|----------|-----------|--------|
| identity-service | OTP/auth/session | ❌ No | PostgreSQL/Redis | sms-sdk, email-sdk, messaging-sdk | CLARIFIED |
| customer-service | Borrower profile | ✅ Yes | PostgreSQL | messaging-sdk | SPLIT |
| kyc-service | KYC verification workflow | ✅ Yes | PostgreSQL | kyc-sdk, messaging-sdk | **NEW** |
| application-service | Loan lifecycle | ✅ Yes | PostgreSQL | messaging-sdk | KEPT |
| income-service | Income & employment | ✅ Yes | PostgreSQL | bank-sdk, epfo-sdk, messaging-sdk | KEPT |
| credit-service | Credit interpretation | ✅ Yes | PostgreSQL | bureau-sdk, messaging-sdk | CLARIFIED |
| eligibility-service | FOIR / IIR computation | ✅ Yes | PostgreSQL | messaging-sdk | KEPT |
| offer-service | Offer generation | ✅ Yes | PostgreSQL | messaging-sdk | **ADDED** |
| sanction-service | Legal sanction | ✅ Yes | PostgreSQL + S3 | esign-sdk, messaging-sdk | KEPT |
| payment-verification-service | Bank account validation | ✅ Yes | PostgreSQL | bank-sdk, messaging-sdk | **NEW** |
| disbursement-service | Payment & payout | ✅ Yes | PostgreSQL | nach-sdk, payment-sdk, messaging-sdk | SIMPLIFIED |
| collection-service | Repayment lifecycle | ✅ Yes | PostgreSQL | payment-sdk, messaging-sdk | DOCUMENTED |
| workflow-service | Orchestration | ❌ No | Camunda DB | messaging-sdk | KEPT |
| config-service | Rule storage | ❌ No | PostgreSQL / Redis | messaging-sdk | KEPT |
| audit-service | Audit trail | ❌ No | Append-only DB | messaging-sdk | KEPT |
| notification-service | Communication | ❌ No | Optional | sms-sdk, email-sdk, messaging-sdk | KEPT |

---

## Core Domain Services (Business Owners)

These services **own business meaning** and persist domain data.

---

### 1. identity-service (CLARIFIED)

**Owns**
- OTP state and verification
- Temporary authentication tokens
- Mobile/Email verification status

**Uses SDKs**
- sms-sdk
- email-sdk
- messaging-sdk

**Database**
- PostgreSQL or Redis

**Stores**
- customer_id
- otp_state (PENDING, VERIFIED, EXPIRED)
- verified_mobile
- verified_email
- auth_token_state

**Rules**
- Does NOT store customer profile
- Does NOT run KYC workflow
- Does NOT make business decisions
- OTP valid for 10 minutes max

---

### 2. customer-service (UPDATED - PROFILE ONLY)

**Owns**
- Borrower profile data
- De-duplication by PAN

**Uses SDK**
- messaging-sdk

**Database**
- PostgreSQL

**Stores**
- customer_id
- pan_hash (encrypted)
- name
- email
- mobile
- dob

**Rules**
- Does NOT handle KYC verification
- Does NOT manage OTP/auth state
- PAN-based de-duplication only
- All PII encrypted at rest

---

### 3. kyc-service (NEW)

**Owns**
- KYC verification workflow
- Identity verification orchestration
- KYC status and history

**Uses SDKs**
- kyc-sdk
- messaging-sdk

**Database**
- PostgreSQL

**Stores**
- application_id
- kyc_status (PENDING, IN_PROGRESS, VERIFIED, FAILED)
- kyc_attempt_count
- kyc_failure_reason
- verification_timestamp

**Rules**
- Does NOT store Aadhaar data (in-memory only, immediately discarded)
- Does NOT store face images
- Does NOT make approval/rejection decisions
- Supports PAN, Aadhaar, and face match workflows
- Handles async vendor callbacks via webhooks

---

### 4. application-service (Single Source of Truth)

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

**Uses SDK**
- messaging-sdk

**Rules**
- ONLY service allowed to change application state
- Other services may only READ state
- All state changes are validated and audited

---

### 5. income-service

**Owns**
- Income and employment truth

**Uses SDKs**
- bank-sdk
- epfo-sdk
- messaging-sdk

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
- Does not store raw bank statements

---

### 6. credit-service (CLARIFIED)

**Owns**
- Credit score interpretation
- Bureau data fetch and parsing
- Credit risk assessment

**Uses SDK**
- bureau-sdk
- messaging-sdk

**Database**
- PostgreSQL (Snapshot-based)

**Stores**
- application_id
- credit_score
- dpd_flags
- vintage_months
- adverse_flags

**Rules**
- Does NOT approve/reject (workflow-service decides)
- Does NOT store raw bureau report
- Masks vendor responses in logs

---

### 7. eligibility-service

**Owns**
- Eligibility computation logic

**Responsibilities**
- FOIR calculation
- IIR calculation
- Maximum EMI computation
- Eligibility decision

**Database**
- PostgreSQL (Decision snapshot)

**Uses SDK**
- messaging-sdk

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

### 8. offer-service (ADDED)

**Owns**
- Loan offer generation

**Responsibilities**
- Generate eligible loan offers
- Apply product caps and tenure options
- Calculate EMI

**Database**
- PostgreSQL

**Uses SDK**
- messaging-sdk

**Stores**
- application_id
- eligible_amount
- tenure_options
- interest_rate
- emi

**Rules**
- Does NOT make approval decisions
- Does NOT store business rules

---

### 9. sanction-service

**Owns**
- Legal loan sanction commitment

**Uses SDK**
- esign-sdk
- messaging-sdk

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

### 10. payment-verification-service (NEW)

**Owns**
- Bank account validation
- Account verification status

**Uses SDKs**
- bank-sdk
- messaging-sdk

**Database**
- PostgreSQL

**Stores**
- application_id
- ifsc
- account_number_masked
- verification_status
- verification_method

**Rules**
- Does NOT setup NACH (disbursement-service does)
- Does NOT execute fund transfer
- Returns payment readiness status only

---

### 11. disbursement-service (SIMPLIFIED)

**Owns**
- NACH mandate creation
- Fund transfer execution
- Disbursement tracking

**Uses SDKs**
- nach-sdk
- payment-sdk
- messaging-sdk

**Database**
- PostgreSQL

**Stores**
- application_id
- loan_account_id
- nach_status
- disbursement_status

**Rules**
- Does NOT verify bank accounts (payment-verification-service does)
- Waits for account verification before NACH setup

---

### 12. collection-service (DOCUMENTED)

**Owns**
- Post-disbursement loan account lifecycle
- EMI schedule management
- Repayment collection tracking

**Uses SDK**
- payment-sdk
- messaging-sdk

**Database**
- PostgreSQL

**Stores**
- loan_account_id
- emi_schedule
- repayment_history
- outstanding_amount
- delinquency_days

**Rules**
- Does NOT make approval/rejection decisions
- Does NOT change loan terms mid-journey

---

## Supporting Services (No Business Ownership)

### workflow-service
- Orchestrates long-running flows using the Camunda rule engine
- Manages retries, compensation, and rule-driven decision enforcement
- Enforces automated decisioning; all flows are system-driven
- Owns no business data
- Uses messaging-sdk for event publishing

### config-service
- Stores platform configuration (timeouts, retries, vendor URLs)
- Stores feature flags and toggles
- Does NOT store business or credit policy rules
- Uses messaging-sdk for config change notifications

### audit-service
- Stores immutable, masked audit logs
- Records all decisions and overrides
- Consumes from Kafka only (not direct service calls)
- Append-only database only

### notification-service
- Sends SMS / Email / Push notifications
- No business logic
- Consumes domain events from Kafka
- Uses messaging-sdk

---

## SDK Rules (Strict)

SDKs are **stateless libraries**, not services.

| Rule | Enforced |
|----|---------|
| SDKs have no database | ✅ |
| SDKs cannot approve / reject | ✅ |
| SDKs return raw responses only | ✅ |
| All services use messaging-sdk | ✅ |

---

## Non-Negotiable Rules

1. Only application-service can change application state
2. SDKs can never directly approve or reject applications; SDKs return raw responses and services or orchestration rules make decisions
3. No service may access another service's database
4. Each service stores only what it decides
5. Snapshot data must be recomputable
6. kyc-service does NOT store Aadhaar data
7. payment-verification-service does NOT setup NACH
8. All events must flow through messaging-sdk Transactional Outbox
9. Audit-service consumes from Kafka only
10. audit-sdk is REMOVED (use Kafka instead)