# Service Landscape & SDK Responsibilities

This document defines **all platform services and SDKs**, their responsibilities,
data ownership, and interaction boundaries.

The goal is to:
- make ownership explicit
- avoid accidental coupling
- enforce vendor abstraction
- support independent scaling

---

## 1. High-Level Service & SDK Matrix (FINAL - Post Review)

### Core Platform Services

| Service | Primary Responsibility | Database | SDKs Used | Status |
|------|-----------------------|----------|-----------|--------|
| identity-service | OTP/auth/session management | Redis/PostgreSQL | sms-sdk, email-sdk, messaging-sdk | CLARIFIED |
| customer-service | Customer profile only | PostgreSQL | messaging-sdk | SPLIT |
| kyc-service | KYC verification workflow | PostgreSQL | kyc-sdk, messaging-sdk | **NEW** |
| application-service | Loan application state | PostgreSQL | messaging-sdk | KEPT |
| income-service | Income & employment verification | PostgreSQL | bank-sdk, epfo-sdk, messaging-sdk | KEPT |
| credit-service | Bureau & credit scoring | PostgreSQL | bureau-sdk, messaging-sdk | CLARIFIED |
| eligibility-service | FOIR/IIR computation | PostgreSQL | messaging-sdk | KEPT |
| offer-service | Offer generation | PostgreSQL | messaging-sdk | **ADDED** |
| sanction-service | Sanction & eSign | PostgreSQL + S3 | esign-sdk, messaging-sdk | KEPT |
| payment-verification-service | Bank account validation | PostgreSQL | bank-sdk, messaging-sdk | **NEW** |
| disbursement-service | NACH & fund transfer | PostgreSQL | nach-sdk, payment-sdk, messaging-sdk | SIMPLIFIED |
| collection-service | Post-disbursal loan lifecycle | PostgreSQL | payment-sdk, messaging-sdk | DOCUMENTED |
| notification-service | Customer communication | PostgreSQL (optional) | sms-sdk, email-sdk, messaging-sdk | KEPT |
| workflow-service | Loan orchestration (Camunda) | Camunda DB | messaging-sdk | KEPT |
| config-service | Platform configuration | PostgreSQL/Redis | messaging-sdk | KEPT |
| audit-service | Immutable audit trail | Append-only DB | messaging-sdk | KEPT |

---

### Common SDKs (Must be used by all domain services)

| SDK | Purpose | Key Feature |
|----|---------|-------------|
| messaging-sdk | Transactional Outbox + Kafka publishing | Event emission with consistency guarantee |
| vendor-observability-sdk | Tracing & PII masking | Request-level observability |
| idempotency-sdk | Request deduplication | Prevents duplicate processing |

---

### Vendor SDKs (Vendor-Agnostic Interfaces)

| SDK | Abstracts | Typical Vendors | Used By |
|----|----------|---------------|---------|
| kyc-sdk | PAN, Aadhaar, face match | ULI, Karza, etc. | kyc-service |
| bank-sdk | Bank statements & salary verification | ULI, Ignosis, etc. | income-service, payment-verification-service |
| epfo-sdk | EPFO employment verification | EPFO / ULI | income-service |
| bureau-sdk | Credit bureau data & scoring | CRIF Highmark, etc. | credit-service |
| esign-sdk | Digital document signing | ULI, SignDesk, etc. | sanction-service |
| nach-sdk | e-NACH mandate creation | CAMS, Razorpay, etc. | disbursement-service |
| payment-sdk | Payment & disbursement (mocked) | Bank APIs (mocked) | disbursement-service, collection-service |
| sms-sdk | SMS delivery | Vendor SMS gateway | notification-service, identity-service |
| email-sdk | Email delivery | Vendor email service | notification-service, identity-service |

---

## 2. Service-by-Service Details

---

###  customer-service
**Responsibility**
- Customer profile management
- Identity association
- De-duplication logic

**Owns**
- Customer ID
- PAN / Aadhaar references (masked)
- KYC linkage

**Uses SDK**
- `kyc-sdk`

**Does NOT Own**
- Loan data
- Eligibility decisions

---

### 📝 application-service
**Responsibility**
- Loan application lifecycle
- State machine ownership
- Customer ↔ Loan mapping

**Owns**
- Loan application ID
- Current loan state

**Uses SDK**
- None

**Key Rule**
> This service owns *state*, not *flow logic*.

---

### 🧾 kyc-service
**Responsibility**
- PAN verification
- Aadhaar verification
- Face match & liveliness

**Uses SDK**
- `kyc-sdk`

**Failure Handling**
- Retry with backoff
- DLQ after threshold

---

### 💰 income-service
**Responsibility**
- Salary verification
- Employment classification
- Bank statement analysis

**Uses SDKs**
- `bank-sdk`
- `epfo-sdk`

---

### 📊 bureau-service
**Responsibility**
- Fetch bureau report
- Parse score, DPD, vintage

**Uses SDK**
- `bureau-sdk`

**Important**
- This service NEVER approves or rejects

---

### 🧮 eligibility-service
**Responsibility**
- FOIR / IIR calculation
- Rule-based eligibility evaluation
- Cooling period checks

**Uses SDK**
- None

**Data Stored**
- Eligibility decisions
- Rejection reasons
- Historical outcomes

---

### 🎁 offer-service
**Responsibility**
- Generate eligible loan offers
- Apply product caps & tenure options

**Uses SDK**
- None

---

### ✍️ sanction-service
**Responsibility**
- Sanction letter generation
- eSign orchestration

**Uses SDK**
- `esign-sdk`

---

### 💳 disbursement-service
**Responsibility**
- Bank verification
- NACH setup
- Disbursement trigger (mocked)

**Uses SDKs**
- `bank-sdk`
- `nach-sdk`
- `payment-sdk`

---

### ⚙️ config-service
**Responsibility**
- Dynamic rule management
- Feature flags
- Threshold configuration

**Critical Rule**
> No policy change should require a redeployment.

---

### 🧾 audit-service
**Responsibility**
- Immutable audit logs
- Decision snapshots
- Masked vendor responses

**Storage**
- Append-only, tamper-resistant

---

### 🔄 workflow-service (Camunda)
**Responsibility**
- Orchestrate loan journey
- Handle retries & compensation
- Enforce automated decision logic

**Uses**
- None (orchestration engine only)

**Does NOT Own**
- Business rules
- Domain data

---

## 3. SDK Design Rules (Non-Negotiable)

- Services NEVER call vendors directly
- All vendors go through SDKs
- SDKs expose:
    - interface
    - mock adapter (default)
    - real adapter (config-driven)
- SDKs emit Kafka events
- SDKs handle retries & timeouts

---

## 4. Why This Design Works

- Clear ownership boundaries
- Safe vendor switching
- Replayable workflows
- Auditable decisions
- Interview-proof architecture

This service landscape reflects **real-world digital lending platforms**, not demo systems.
