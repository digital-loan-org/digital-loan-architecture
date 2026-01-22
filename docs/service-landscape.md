# Service Landscape & SDK Responsibilities

This document defines **all platform services and SDKs**, their responsibilities,
data ownership, and interaction boundaries.

The goal is to:
- make ownership explicit
- avoid accidental coupling
- enforce vendor abstraction
- support independent scaling

---

## 1. High-Level Service & SDK Matrix

### Core Services

| Service | Primary Responsibility | Database | SDKs Used |
|------|-----------------------|----------|-----------|
| auth-service | Authentication & authorization | Keycloak DB | — |
| customer-service | Customer profile & identity | PostgreSQL | identity-sdk |
| loan-application-service | Loan lifecycle & state | MongoDB | workflow-sdk |
| kyc-service | PAN, Aadhaar, face match | PostgreSQL | identity-sdk |
| income-service | Salary & employment checks | PostgreSQL | bank-agg-sdk, epfo-sdk |
| bureau-service | Credit bureau fetch & parsing | PostgreSQL | bureau-sdk |
| eligibility-service | FOIR/IIR & policy rules | PostgreSQL | config-sdk |
| offer-service | Loan offer generation | PostgreSQL | config-sdk |
| sanction-service | Sanction & eSign | PostgreSQL | esign-sdk |
| payment-service | Penny drop, NACH, disbursement (mock) | PostgreSQL | nach-sdk, payment-sdk |
| config-service | Dynamic rules & thresholds | PostgreSQL | — |
| audit-service | Immutable audit trail | Append-only DB | — |
| workflow-service | Loan journey orchestration | Camunda DB | workflow-sdk |

---

### Platform SDKs (Internal – Non-Vendor)

| SDK | Purpose | Used By |
|----|--------|--------|
| platform-common | Exceptions, headers, idempotency | All services |
| workflow-sdk | Camunda client abstraction | loan-application-service |
| config-sdk | Rule & threshold access | eligibility, offer |
| audit-sdk | Audit event publishing | All services |

---

### Vendor SDKs (Vendor-Agnostic Interfaces)

| SDK | Abstracts | Typical Vendors |
|----|----------|---------------|
| identity-sdk | PAN, Aadhaar, face match | ULI, Karza |
| bank-agg-sdk | Bank statements & salary | ULI, Ignosis |
| epfo-sdk | EPFO employment verification | EPFO / ULI |
| bureau-sdk | Credit bureau data | CRIF Highmark |
| esign-sdk | Digital document signing | ULI, SignDesk |
| nach-sdk | e-NACH mandate | CAMS, Razorpay |
| payment-sdk | Disbursement rails (mocked) | Bank APIs |

---

## 2. Service-by-Service Details

---

### 🔐 auth-service
**Responsibility**
- User authentication
- Token issuance
- Role-based access control

**Owns**
- Users
- Roles
- Access policies

**Notes**
- Backed by Keycloak
- No business data allowed

---

### 👤 customer-service
**Responsibility**
- Customer profile management
- Identity association
- De-duplication logic

**Owns**
- Customer ID
- PAN / Aadhaar references (masked)
- KYC linkage

**Uses SDK**
- `identity-sdk`

**Does NOT Own**
- Loan data
- Eligibility decisions

---

### 📝 loan-application-service
**Responsibility**
- Loan application lifecycle
- State machine ownership
- Customer ↔ Loan mapping

**Owns**
- Loan application ID
- Current loan state

**Uses SDK**
- `workflow-sdk`

**Key Rule**
> This service owns *state*, not *flow logic*.

---

### 🧾 kyc-service
**Responsibility**
- PAN verification
- Aadhaar verification
- Face match & liveliness

**Uses SDK**
- `identity-sdk`

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
- `bank-agg-sdk`
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
- `config-sdk`

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
- `config-sdk`

---

### ✍️ sanction-service
**Responsibility**
- Sanction letter generation
- eSign orchestration

**Uses SDK**
- `esign-sdk`

---

### 💳 payment-service
**Responsibility**
- Penny drop verification
- NACH setup
- Disbursement trigger (mocked)

**Uses SDKs**
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
- Provide Ops visibility

**Uses**
- `workflow-sdk`

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
