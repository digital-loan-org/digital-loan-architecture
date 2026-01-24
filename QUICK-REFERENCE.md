# 🎯 Quick Reference: 16-Service Architecture

**Date:** January 24, 2026 | **Status:** ✅ FINAL

---

## SERVICE QUICK LOOKUP

### Tier 1: Core Domain Services (Own Business Decisions)

| # | Service | Responsibility | DB | Key SDKs |
|---|---------|-----------------|----|----|
| 1 | **identity-service** | OTP/auth/sessions | Redis | sms-sdk, email-sdk |
| 2 | **customer-service** | Profile only | PG | - |
| 3 | **kyc-service** | Verification workflow | PG | kyc-sdk |
| 4 | **application-service** | State owner | PG | - |
| 5 | **income-service** | Salary verification | PG | bank-sdk, epfo-sdk |
| 6 | **credit-service** | Bureau + scoring | PG | bureau-sdk |
| 7 | **eligibility-service** | FOIR/IIR | PG | - |
| 8 | **offer-service** | Offer generation | PG | - |
| 9 | **sanction-service** | Sanction + eSign | PG+S3 | esign-sdk |
| 10 | **payment-verification-service** | Bank validation | PG | bank-sdk |
| 11 | **disbursement-service** | NACH + transfer | PG | nach-sdk, payment-sdk |
| 12 | **collection-service** | EMI + repayment | PG | payment-sdk |

### Tier 2: Support Services (No Business Decisions)

| # | Service | Responsibility | DB |
|---|---------|-----------------|-----|
| 13 | **workflow-service** | Orchestration only | Camunda |
| 14 | **config-service** | Platform config | Redis |
| 15 | **audit-service** | Immutable logs | Append-only |
| 16 | **notification-service** | SMS/Email/Push | - |

---

## WHO CAN CALL WHO

### ✅ ALLOWED

```
API Gateway → Any Service (REST read)
Workflow-Service → Services (REST, via Camunda)
Any Service → config-service (read config)
All Services → Kafka (emit events via messaging-sdk)
Audit-Service ← Kafka (read events only)
Notification-Service ← Kafka (read events only)
```

### ❌ FORBIDDEN

```
Service → Service direct DB access
Service → Vendor direct (must use SDK)
Service → Audit-Service direct (use Kafka)
SDK → Any business logic
audit-sdk → audit-service (removed - use Kafka)
Multiple producers → Same event type
```

---

## CRITICAL RULES (DO NOT VIOLATE)

```
1. ONLY application-service changes state
2. NO vendor calls outside SDKs
3. Aadhaar is NEVER stored (in-memory only)
4. Audit is Kafka-only (Transactional Outbox)
5. SDKs have NO database
6. Each service owns its database
7. Events are IMMUTABLE facts
8. No service accesses another's database
9. config-service has NO business logic
10. workflow-service stores NO domain data
```

---

## SERVICE INTERACTION MAP

```
┌─────────────────────────────────────────────────────────────────┐
│                          API Gateway                             │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                ┌──────────┴──────────┐
                ▼                     ▼
        ┌─────────────────┐    ┌──────────────────┐
        │  APPLICATION    │◄──►│  WORKFLOW        │
        │  SERVICE        │    │  SERVICE         │
        │ (STATE OWNER)   │    │ (Camunda)        │
        └────────┬────────┘    └──────────────────┘
                 │
        ┌────────┴───────────────────────────────┐
        │                                        │
        ▼                                        ▼
  ┌────────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ IDENTITY   │  │CUSTOMER  │  │   KYC    │  │ INCOME   │
  │ SERVICE    │  │ SERVICE  │  │ SERVICE  │  │ SERVICE  │
  │ (OTP/Auth) │  │(Profile) │  │(Verify)  │  │(Salary)  │
  └────────────┘  └──────────┘  └──────────┘  └──────────┘
                                                    │
        ┌──────────────────────────────────────────┘
        │
        ▼
   ┌─────────────┐  ┌──────────┐  ┌────────────┐  ┌────────┐
   │  CREDIT     │  │ELIGIBILITY│ │  OFFER     │  │SANCTION│
   │ SERVICE     │  │ SERVICE   │ │  SERVICE   │  │SERVICE │
   │(Bureau Chk) │  │(FOIR/IIR) │ │(Generate)  │  │(eSign) │
   └─────────────┘  └──────────┘  └────────────┘  └────────┘
                                                         │
        ┌────────────────────────────────────────────────┘
        │
        ▼
   ┌────────────────────┐  ┌──────────────────────┐
   │ PAYMENT VERIF      │  │ DISBURSEMENT SERVICE │
   │ SERVICE            │  │ (NACH + Transfer)    │
   │(Bank Validation)   │  │                      │
   └────────────────────┘  └──────────────────────┘
                                     │
        ┌────────────────────────────┘
        │
        ▼
   ┌─────────────────┐
   │ COLLECTION      │
   │ SERVICE         │
   │(EMI + Repay)    │
   └─────────────────┘

         ALL SERVICES
              │
    ┌─────────┼─────────┐
    ▼         ▼         ▼
 ┌─────┐  ┌──────┐  ┌───────┐
 │KAFKA│  │CONFIG│  │AUDIT  │
 │EVENT│  │SERV  │  │SERV   │
 └─────┘  └──────┘  └───────┘
```

---

## ERROR CODE QUICK REFERENCE

| Service | Errors | Format | Example |
|---------|--------|--------|---------|
| customer | Validation, Duplicate | `CUST_*` | `CUST_DUP_001` |
| identity | OTP, Rate limit | `IDENTITY_*` | `IDENTITY_OTP_001` |
| kyc | PAN, Aadhaar, Face | `KYC_*` | `KYC_PAN_001` |
| income | Salary, Employment | `INCOME_*` | `INCOME_SALARY_001` |
| credit | Score, DPD, Vendor | `CREDIT_*` | `CREDIT_SCORE_001` |
| eligibility | FOIR, IIR, Age | `ELIG_*` | `ELIG_FOIR_001` |
| payment-verif | Bank, Account | `PVER_*` | `PVER_BANK_001` |
| disbursement | NACH, Transfer | `DISB_*` | `DISB_NACH_001` |
| sanction | eSign errors | `SANC_*` | `SANC_ESIGN_001` |
| collection | Payment, EMI | `COLL_*` | `COLL_PAYMENT_001` |

**All errors also include:**
- Retry policy (HARD_FAIL, SOFT_FAIL, WARN)
- Customer message (non-technical)
- HTTP status code
- Internal logging details

---

## DATABASE OWNERSHIP

```
Each Service = Own PostgreSQL Database (or specialized)

identity-service          → Redis (sessions) / PG (audit)
customer-service          → PostgreSQL
kyc-service              → PostgreSQL
application-service      → PostgreSQL (AUTHORITATIVE)
income-service           → PostgreSQL
credit-service           → PostgreSQL
eligibility-service      → PostgreSQL
offer-service            → PostgreSQL
sanction-service         → PostgreSQL + S3 (documents)
payment-verification-svc → PostgreSQL
disbursement-service     → PostgreSQL
collection-service       → PostgreSQL (separate)
workflow-service         → Camunda DB (BPMN engine)
config-service           → PostgreSQL + Redis (cache)
audit-service            → Append-only DB (Cassandra)
notification-service     → Optional (logging only)
```

**Rule:** NO cross-service database access. ALL reads via API.

---

## SDK DEPENDENCIES

### ALL SERVICES USE
```
messaging-sdk          (Kafka + Transactional Outbox)
```

### VENDOR-SPECIFIC
```
kyc-service            → kyc-sdk
income-service         → bank-sdk, epfo-sdk
credit-service         → bureau-sdk
payment-verif-service  → bank-sdk
disbursement-service   → nach-sdk, payment-sdk
collection-service     → payment-sdk
sanction-service       → esign-sdk
notification-service   → sms-sdk, email-sdk
identity-service       → sms-sdk, email-sdk
```

### COMMON
```
vendor-observability-sdk (all: tracing + masking)
idempotency-sdk          (select services)
```

**REMOVED:**
```
❌ audit-sdk (audit via Kafka only)
```

---

## STATE MACHINE (Simplified)

```
CREATED
  ↓
PROFILE_COMPLETED (identity verified)
  ↓
KYC_VERIFIED
  ↓
INCOME_VERIFIED
  ↓
BUREAU_VERIFIED
  ↓
ELIGIBILITY_EVALUATED
  ├─ IF ELIGIBLE → OFFER_GENERATED
  │                ↓
  │           OFFER_ACCEPTED
  │                ↓
  │           SANCTIONED
  │                ↓
  │           NACH_SETUP
  │                ↓
  │           DISBURSED
  │                ↓
  │           CLOSED ✅
  │
  └─ IF NOT ELIGIBLE → REJECTED ❌

Terminal States: REJECTED, EXPIRED, CANCELLED
```

---

## KAFKA TOPICS (Domain-Oriented)

```
loan.application.events      ← All application state changes
identity.events               ← OTP, auth events
customer.events               ← Profile changes
kyc.events                   ← KYC verification events
income.events                ← Salary verification events
credit.events                ← Bureau check events
eligibility.events           ← Eligibility decisions
offer.events                 ← Offer events
sanction.events              ← Sanction & eSign events
payment.events               ← Bank verification & payment events
disbursement.events          ← NACH & fund transfer events
collection.events            ← EMI & repayment events
notification.events          ← Notification sent events
audit.events                 ← Immutable audit trail (all services publish)
```

---

## RETRY STRATEGIES

### SDK/Service Level (3 retries)
```
Backoff: 100ms → 200ms → 400ms
Max: 10 seconds
Used for: Transient errors, vendor timeouts
```

### Workflow Level (5 retries)
```
Backoff: 1m → 1.5m → 2.25m → 3.375m → 5m
Max: 1 hour
Used for: Long-running async operations
```

### Vendor SDK (Rate-limit Backoff)
```
Backoff: 10s → 30s → 1m → 5m
Used for: Rate-limit errors, quota exhausted
```

---

## MUST-KNOW CONSTRAINTS

### Data Handling
```
✅ All PII encrypted at rest
✅ Aadhaar NEVER stored (in-memory only, immediately discarded)
✅ Face images NOT persisted
✅ Bank statements NOT stored (snapshot only)
✅ Bureau report NOT stored (parsed fields only)
❌ NO customer data in logs
❌ NO raw PAN values stored
❌ NO Aadhaar in audit trail
```

### State Management
```
✅ Database is authoritative (not Kafka)
✅ application-service ONLY can change state
✅ State transitions immutable once committed
✅ Snapshots recomputable anytime
❌ NO event sourcing (database-first)
❌ NO hidden state in services
❌ NO rollbacks (stage completion is final)
```

### Orchestration
```
✅ Camunda defines all workflows
✅ Explicit retries & compensation
✅ Policy changes without redeploy
✅ All decisions automated (no manual)
❌ workflow-service stores NO data
❌ workflow-service makes NO approval decisions
❌ NO custom orchestration code
```

---

## COMPLIANCE CHECKLIST

```
✅ RBI-aligned data handling
✅ Immutable audit trail (append-only)
✅ Aadhaar non-storage guarantee
✅ Encryption at rest & transit
✅ Access control per role
✅ PII masking in logs
✅ Decision snapshots for audit
✅ State transition immutability
✅ Vendor isolation (SDKs)
✅ Error traceability
```

---

## COMMON MISTAKES TO AVOID

| ❌ WRONG | ✅ RIGHT | Why |
|---------|---------|-----|
| Service calls vendor directly | Service uses SDK | Vendor abstraction |
| Direct DB access | REST API calls | Data ownership |
| Multiple state owners | application-service only | Consistency |
| Events before commit | Transactional Outbox | Audit correctness |
| Audit SDK calls | Kafka events | Dual-write safety |
| SDKs with logic | Stateless SDKs only | Easy switching |
| Stored Aadhaar | In-memory only | RBI compliance |
| Business logic in config-service | In Camunda | Clear separation |
| Workflow stores data | No domain storage | Orchestration only |
| Shared DLQ | Per-service DLQ | Clear ownership |

---

## GETTING HELP

| Question | Document |
|----------|----------|
| "Why this architecture?" | [ADR-001](docs/architecture-decisions/ADR-001-architecture-style.md) |
| "What does this service do?" | [SERVICE-RESPONSIBILITY-MATRIX](docs/SERVICE-RESPONSIBILITY-MATRIX.md) |
| "How do I handle this error?" | [ERROR-CATALOG](docs/api-contracts/ERROR-CATALOG.md) |
| "What events exist?" | [kafka-event-catalog](docs/integration/kafka-event-catalog.md) |
| "Data security rules?" | [security-compliance](docs/security-compliance/security-and-compliance.md) |
| "State machine?" | [loan-journey-states](docs/lifecycle/loan-journey-states.md) |
| "Retry logic?" | [failure-strategy](docs/failure-handling/failure-strategy.md) |
| "Full overview?" | [service-landscape](docs/architecture-overview/service-landscape.md) |

---

**Status:** ✅ FINAL  
**Last Updated:** January 24, 2026  
**Architecture Ready:** YES  
**For Production:** YES
