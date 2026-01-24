# Architecture Review & Correction Document

**Date:** January 24, 2026  
**Reviewer:** Staff / Principal Backend Architect  
**Status:** Final Review Complete

---

## Executive Summary

The Digital Loan Architecture is **substantially correct** in its core design principles:
- Event-driven microservices ✅
- Database as source of truth ✅
- Transactional Outbox pattern ✅
- Saga-based orchestration via Camunda ✅
- Vendor abstraction via SDKs ✅

However, **10 critical issues** were identified requiring service boundary corrections and documentation improvements.

**Key Changes:**
1. ✅ CREATE `payment-verification-service` (split from disbursement)
2. ✅ CREATE/CLARIFY `kyc-service` (separate from customer-service)
3. ✅ CLARIFY `identity-service` (OTP/auth only, not profile)
4. ✅ SPLIT `customer-service` (profile only)
5. ✅ REMOVE `audit-sdk` (audit via Kafka only)
6. ✅ ADD missing `offer-service` to service list
7. ✅ DOCUMENT `collection-service` properly
8. ✅ CLARIFY `credit-service` vs `bureau-service`
9. ✅ ADD `messaging-sdk` to all service dependencies
10. ✅ CREATE error catalog & API contracts

---

## Issue-by-Issue Breakdown

### **Issue #1: Missing `payment-verification-service`** [CRITICAL]

**Current State:**
- `disbursement-service` owns both:
  - Bank account verification
  - NACH mandate setup/verification
  - Fund transfer execution

**Problem:**
This violates Single Responsibility Principle and couples services with different SLAs:
- Bank verification: synchronous, <2 sec
- NACH mandate: asynchronous, 1-3 days
- Fund transfer: scheduled, not immediate

**Real-World Fintech Pattern:**
- **Bank Verification** → Fast, synchronous, high-frequency retry
- **NACH Setup** → Async, low-frequency, compliance-sensitive
- **Disbursement** → Scheduled execution, separate timing

**Solution:**

Create `payment-verification-service`:

```markdown
## payment-verification-service

**Responsibility**
- Bank account validation (IFSC, account number format)
- Account verification via NEFT test transaction (if required)
- Return payment method readiness status

**Owns**
- application_id
- verified_bank_account
- verification_timestamp
- verification_method

**Uses SDK**
- bank-sdk (for account lookup / validation)
- messaging-sdk

**Consumed Events**
- disbursement.ready_for_payment → Verify bank account

**Emitted Events**
- payment.account_verified
- payment.account_invalid

**Key Rules**
- Does NOT set up NACH mandates (that's disbursement-service)
- Does NOT initiate fund transfer
- Returns payment readiness status only
- Supports retries for transient bank API failures
```

**Updated disbursement-service:**

```markdown
## disbursement-service (UPDATED)

**Responsibility**
- NACH mandate creation & setup
- Fund transfer execution
- Disbursement tracking

**Owns**
- application_id / loan_account_id
- nach_mandate_status
- disbursement_status
- disbursement_timestamp

**Uses SDK**
- nach-sdk
- payment-sdk
- messaging-sdk

**Consumed Events**
- sanction.completed → Prepare for disbursement
- payment.account_verified → Setup NACH
- workflow.disbursement_approved → Execute fund transfer

**Emitted Events**
- disbursement.nach_initiated
- disbursement.nach_completed
- disbursement.fund_transfer_initiated
- disbursement.completed
- disbursement.failed

**Key Rules**
- DOES NOT verify bank accounts (payment-verification-service does that)
- Waits for both sanction + payment verification before NACH setup
- Fund transfer only after NACH mandate is active
```

---

### **Issue #2: Missing/Unclear `kyc-service`** [CRITICAL]

**Current State:**
Documentation mentions KYC workflow but ownership is fuzzy:
- Is it in `customer-service`?
- Is there a separate `kyc-service`?

**Problem:**
KYC is a **multi-step, async workflow** with:
- PAN verification (vendor call)
- Aadhaar verification (vendor call)
- Face match (vendor call)
- Webhook callbacks
- Complex retry logic

This is fundamentally different from customer profile management.

**Solution:**

Create explicit `kyc-service`:

```markdown
## kyc-service (NEW)

**Responsibility**
- PAN verification via KYC vendor
- Aadhaar-based identity verification
- Face match / liveness detection
- KYC workflow orchestration

**Owns**
- application_id
- kyc_status (PENDING, IN_PROGRESS, VERIFIED, FAILED)
- kyc_attempt_count
- kyc_failure_reason
- verification_timestamp

**Uses SDK**
- kyc-sdk (vendor: ULI, Karza, etc.)
- messaging-sdk

**Consumed Events**
- application.profile_completed → Initiate KYC

**Emitted Events**
- kyc.initiated
- kyc.in_progress
- kyc.verified
- kyc.failed

**Key Rules**
- Does NOT store Aadhaar data (in-memory only)
- Does NOT store face images
- Supports async callbacks from KYC vendor
- Retries via workflow-service orchestration
- Masks vendor responses in audit logs

**NOT Owned by kyc-service:**
- Customer profile (customer-service owns this)
- Customer identity state (identity-service owns OTP/auth)
```

**Updated customer-service:**

```markdown
## customer-service (UPDATED - PROFILE ONLY)

**Responsibility**
- Customer profile management
- De-duplication logic (PAN-based)

**Owns**
- customer_id
- pan_hash
- name
- email
- mobile
- dob

**Uses SDK**
- messaging-sdk

**Consumed Events**
- None (upstream of most flows)

**Emitted Events**
- customer.created
- customer.updated

**Key Rules**
- Does NOT verify identity (identity-service does OTP)
- Does NOT run KYC workflow (kyc-service does)
- Stores minimal PII (all encrypted)
- No PAN or Aadhaar raw values stored
```

---

### **Issue #3: Missing `identity-service` Definition** [HIGH]

**Current State:**
Listed in service list but never defined.

**Solution:**

```markdown
## identity-service

**Responsibility**
- OTP generation for all channels
- OTP verification
- Temporary authentication tokens
- Mobile/Email ownership verification

**Owns**
- customer_id
- verified_mobile
- verified_email
- otp_state (PENDING, VERIFIED, EXPIRED)
- auth_token_state

**Uses SDK**
- email-sdk
- sms-sdk
- messaging-sdk

**Does NOT Own:**
- Customer profile
- KYC status
- Business decisions
- Permanent authentication (that's auth-service if it exists)

**Key Rules**
- OTP validity: 10 minutes
- Max 3 attempts per OTP
- Max 5 OTPs per day per customer
- No permanent storage of OTP values
- Tokens are session-scoped only
```

---

### **Issue #4: Clarity on `credit-service` vs `bureau-service`** [MEDIUM]

**Current State:**
Documentation uses both names inconsistently.

**Solution:**

Single service owns both bureau integration AND credit interpretation:

```markdown
## credit-service

**Responsibility**
- Bureau data fetch & parsing
- Credit score interpretation
- Credit risk assessment
- DPD (Days Past Due) trend analysis

**Owns**
- application_id
- bureau_score
- dpd_status
- bureau_vintage_months
- adverse_flags
- credit_interpretation

**Uses SDK**
- bureau-sdk (vendor: CRIF Highmark)
- messaging-sdk

**Consumed Events**
- income.verified → Trigger bureau check

**Emitted Events**
- credit.verified
- credit.verification_failed
- credit.rejected (business decision)

**Key Rules**
- Does NOT approve/reject (workflow-service decides)
- Stores only parsed bureau data, not raw response
- Masks raw bureau response in audit logs
- Supports retries on vendor timeout
```

**Decision:** Keep single `credit-service`, discard `bureau-service` terminology.

---

### **Issue #5: Missing `offer-service` in Service List** [HIGH]

**Current State:**
Documented in domain-ownership.md but missing from final service list.

**Solution:**

Add to official service list and document:

```markdown
## offer-service

**Responsibility**
- Loan offer generation based on eligibility
- Apply product-level caps & tenure options
- Calculate interest rates

**Owns**
- application_id
- eligible_amount
- loan_tenure_options
- interest_rate
- emi
- offer_validity_period

**Uses SDK**
- messaging-sdk

**Consumed Events**
- eligibility.approved → Generate offer

**Emitted Events**
- offer.generated
- offer.presented

**Key Rules**
- Uses config-service for product rules
- Does NOT approve/reject (eligibility-service does)
- Offer is deterministic based on eligibility
- Supports multiple tenure/rate options for customer choice
```

---

### **Issue #6: Remove `audit-sdk` from SDK List** [CRITICAL]

**Current State:**
`audit-sdk` is listed but this violates audit correctness principles.

**Problem:**
If services call audit-service directly via SDK:
- Audit calls can fail while domain service commits
- Dual-write consistency issue
- Audit trail can have gaps
- Not crash-safe

**Solution:**

**REMOVE `audit-sdk` from SDKs.**

Audit must flow through Transactional Outbox:

```
Domain Service
  ↓
Persists State + Event to Outbox (single transaction)
  ↓
Message Publisher (background job)
  ↓
Kafka Topic
  ↓
Audit-Service consumes from Kafka
```

No direct SDK calls to audit-service.

---

### **Issue #7: Clarify `messaging-sdk` Usage** [HIGH]

**Current State:**
`messaging-sdk` exists but services don't explicitly list it as a dependency.

**Solution:**

Every domain service MUST use `messaging-sdk` for:
- Writing events to Outbox table
- Transactional consistency
- Automatic Kafka publishing

**Updated dependencies:**

```
Customer-Service
  ├─ kyc-sdk (for KYC initiation reference)
  └─ messaging-sdk (for event emission)

KYC-Service
  ├─ kyc-sdk (vendor integration)
  └─ messaging-sdk

Income-Service
  ├─ bank-sdk
  ├─ epfo-sdk
  └─ messaging-sdk

Credit-Service
  ├─ bureau-sdk
  └─ messaging-sdk

Application-Service
  └─ messaging-sdk

Eligibility-Service
  └─ messaging-sdk

Offer-Service
  └─ messaging-sdk

Sanction-Service
  ├─ esign-sdk
  └─ messaging-sdk

Payment-Verification-Service
  ├─ bank-sdk
  └─ messaging-sdk

Disbursement-Service
  ├─ nach-sdk
  ├─ payment-sdk
  └─ messaging-sdk

Collection-Service
  ├─ payment-sdk
  └─ messaging-sdk

Notification-Service
  ├─ email-sdk
  ├─ sms-sdk
  └─ messaging-sdk
```

---

### **Issue #8: Create Error Catalog** [MEDIUM]

**Current State:**
No standardized error codes or customer-facing error messages.

**Solution:**

Create `docs/api-contracts/error-catalog.md` with:

```markdown
# Error Catalog

## Error Code Format
`<SERVICE>_<CATEGORY>_<CODE>`

Example: `CUST_KYC_001`

## Standardized Errors

### Customer Service
- `CUST_PROFILE_001`: Invalid mobile format
- `CUST_PROFILE_002`: Invalid email format
- `CUST_DUP_001`: Customer already exists (duplicate PAN)

### KYC Service
- `KYC_PAN_001`: PAN verification failed (invalid PAN)
- `KYC_PAN_002`: PAN format invalid
- `KYC_AADHAAR_001`: Aadhaar verification failed
- `KYC_FACE_001`: Face match failed / Liveliness check failed
- `KYC_VENDOR_001`: Vendor unavailable (retry)
- `KYC_VENDOR_002`: Vendor rate limit (backoff & retry)

### Income Service
- `INCOME_BANK_001`: Bank account not found
- `INCOME_SALARY_001`: Salary below minimum threshold
- `INCOME_VERIFICATION_001`: Employment verification failed
- `INCOME_VENDOR_001`: Bank vendor unavailable

### Credit Service
- `CREDIT_BUREAU_001`: Bureau check failed (vendor error)
- `CREDIT_SCORE_001`: Credit score below minimum
- `CREDIT_DPD_001`: Applicant has open DPD flags
- `CREDIT_BUREAU_002`: Bureau data unavailable

### Eligibility Service
- `ELIG_FOIR_001`: FOIR exceeds maximum
- `ELIG_IIR_001`: IIR exceeds maximum
- `ELIG_AGE_001`: Applicant below minimum age
- `ELIG_AGE_002`: Applicant above maximum age
- `ELIG_COOL_001`: Cooling period not satisfied

### Disbursement Service
- `DISB_BANK_001`: Bank account invalid
- `DISB_BANK_002`: Bank not supported
- `DISB_NACH_001`: NACH mandate creation failed
- `DISB_NACH_002`: NACH mandate pending (retry later)
- `DISB_PAYMENT_001`: Fund transfer failed

## Error to Rejection Reason Mapping

| Error Code | Customer Message | Workflow Action |
|---|---|---|
| `KYC_PAN_001` | "Identity verification failed" | REJECT |
| `INCOME_SALARY_001` | "Eligibility criteria not met" | REJECT |
| `CREDIT_SCORE_001` | "Unable to process at this time" | REJECT |
| `DISB_BANK_001` | "Bank account details invalid" | ESCALATE |
| `KYC_VENDOR_001` | "Processing delay, please try later" | RETRY |

## Error Severity Levels

- **HARD_FAIL**: No retry, terminal rejection
- **SOFT_FAIL**: Retry with backoff
- **WARN**: Retry, escalate if repeated
```

---

### **Issue #9: Split `customer-service` Scope** [MEDIUM]

**Current State:**
Customer-service owns profile + identity + KYC linkage (too broad).

**Solution:**

| Data | Owner | Why |
|---|---|---|
| Name, DOB, Email, Mobile | **customer-service** | Master profile data |
| OTP, Auth tokens, Sessions | **identity-service** | Auth state |
| KYC status, KYC attempts | **kyc-service** | Compliance workflow |

This separation allows:
- Scaling auth independently
- KYC workflow isolation
- Clear compliance audit trail

**Ownership Matrix:**

```
customer-service:
- customer_id (generated)
- pan_hash (encrypted)
- name
- email
- mobile
- dob
- created_timestamp
- updated_timestamp

identity-service:
- customer_id
- otp_state
- verified_mobile
- verified_email
- auth_token
- token_expiry

kyc-service:
- application_id (not customer_id)
- kyc_status
- kyc_attempts
- kyc_completion_date
- failure_reason (if failed)
```

---

### **Issue #10: Document `collection-service`** [MEDIUM]

**Current State:**
Listed in service list but barely documented.

**Solution:**

```markdown
## collection-service

**Responsibility**
- Post-disbursement loan account lifecycle
- EMI schedule generation & management
- Repayment collection tracking
- Delinquency (DPD) calculation
- Recovery workflow triggering

**Owns**
- loan_account_id (NOT application_id)
- emi_schedule
- repayment_history
- outstanding_amount
- delinquency_days_past_due
- collection_status

**Uses SDK**
- payment-sdk (for manual/online payment processing)
- messaging-sdk

**Consumed Events**
- disbursement.completed → Create loan account & EMI schedule

**Emitted Events**
- collection.account_created
- collection.emi_due
- collection.repayment_received
- collection.repayment_failed
- collection.delinquent (when DPD > 0)
- collection.escalated (when DPD > 30)
- collection.loan_closed

**Key Rules**
- Separate database from application-service
- Does NOT approve/reject loans
- EMI schedule is generated from sanction amount & tenure
- DPD is calculated daily
- Escalation rules configured in config-service
- Supports partial payments (if product allows)
- Integration with external collection vendors (future scope)

**Out of Scope (Current):**
- Physical collection visits
- Manual underwriting
- Legal recovery action
- Write-off decisions (policy-based, driven by config-service)
```

---

## Final Verified Service Matrix

| Service | Responsibility | DB | SDKs | Status |
|---|---|---|---|---|
| **identity-service** | OTP/auth/sessions | Redis (optional) | sms-sdk, email-sdk, messaging-sdk | CLARIFY |
| **customer-service** | Profile only | PostgreSQL | messaging-sdk | SPLIT |
| **kyc-service** | KYC verification workflow | PostgreSQL | kyc-sdk, messaging-sdk | CREATE |
| **application-service** | Loan application state | PostgreSQL | messaging-sdk | KEEP |
| **income-service** | Income verification | PostgreSQL | bank-sdk, epfo-sdk, messaging-sdk | KEEP |
| **credit-service** | Bureau & credit scoring | PostgreSQL | bureau-sdk, messaging-sdk | CLARIFY |
| **eligibility-service** | FOIR/IIR rules | PostgreSQL | messaging-sdk | KEEP |
| **offer-service** | Offer generation | PostgreSQL | messaging-sdk | ADD |
| **sanction-service** | Sanction & eSign | PostgreSQL + S3 | esign-sdk, messaging-sdk | KEEP |
| **payment-verification-service** | Bank account validation | PostgreSQL | bank-sdk, messaging-sdk | CREATE |
| **disbursement-service** | NACH + fund transfer | PostgreSQL | nach-sdk, payment-sdk, messaging-sdk | SIMPLIFY |
| **collection-service** | Post-disbursal | PostgreSQL | payment-sdk, messaging-sdk | DOCUMENT |
| **notification-service** | SMS/Email/Push | Optional | email-sdk, sms-sdk, messaging-sdk | KEEP |
| **workflow-service** | Orchestration (Camunda) | Camunda DB | messaging-sdk | KEEP |
| **config-service** | Platform config | PostgreSQL/Redis | messaging-sdk | KEEP |
| **audit-service** | Immutable audit logs | Append-only DB | messaging-sdk | KEEP |

---

## SDK List (FINAL - CORRECTED)

```
digital-loan-sdks/

common/
  - vendor-observability-sdk (tracing & masking)
  - idempotency-sdk (request deduplication)
  - messaging-sdk (Outbox + Kafka publishing)

vendor/
  - email-sdk
  - sms-sdk
  - kyc-sdk
  - bank-sdk
  - epfo-sdk
  - bureau-sdk
  - esign-sdk
  - nach-sdk
  - payment-sdk

REMOVED:
  - audit-sdk (use Kafka instead)
```

---

## Implementation Checklist

- [ ] Create `kyc-service` service definition
- [ ] Create `payment-verification-service` service definition
- [ ] Split `customer-service` to profile-only
- [ ] Clarify `identity-service` (OTP/auth)
- [ ] Clarify `credit-service` (bureau + scoring)
- [ ] Add `offer-service` to official service list
- [ ] Document `collection-service` fully
- [ ] Remove `audit-sdk` from SDK list
- [ ] Update all services to list `messaging-sdk` as dependency
- [ ] Create `error-catalog.md`
- [ ] Create `api-versioning.md`
- [ ] Update service-landscape.md with all corrections
- [ ] Update domain-ownership.md with new services
- [ ] Create service responsibility matrix (tables)
- [ ] Create architecture diagrams (updated C4)

---

## Verdict

**Overall Architecture: SOUND** ✅

The core patterns (microservices, event-driven, saga orchestration, database as source of truth) are correct and production-grade.

**Issues: SERVICE BOUNDARIES** (not design patterns)

The 10 issues identified are all about **service decomposition**, not fundamental architecture flaws. These corrections align the design with real-world fintech standards.

**Post-Correction Status: PRODUCTION-READY** ✅

After implementing these corrections, the architecture is ready for:
- Senior engineering interviews
- Regulatory design reviews
- Real-world development teams
- Scale-up with confidence

---

**Review Completed By:** Staff Backend Architect  
**Date:** January 24, 2026
