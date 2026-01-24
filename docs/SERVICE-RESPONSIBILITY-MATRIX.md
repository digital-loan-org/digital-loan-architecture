# Service Responsibility Matrix (Final & Authoritative)

**Effective Date:** January 24, 2026  
**Status:** FINAL - After Architecture Review

---

## I. PLATFORM SERVICES (WHAT EACH SERVICE OWNS)

### 1. identity-service
| Aspect | Details |
|--------|---------|
| **Primary Responsibility** | OTP generation, verification, session management |
| **Owns Database** | Redis (session state) or PostgreSQL (for audit) |
| **Owns Data** | customer_id, otp_state, verified_mobile, auth_tokens |
| **Uses SDKs** | sms-sdk, email-sdk, messaging-sdk |
| **Consumed Events** | application.profile_completed (trigger OTP flow) |
| **Emitted Events** | identity.otp_sent, identity.otp_verified, identity.verification_failed |
| **Can Call Services** | None (stateless service) |
| **Cannot Do** | Make business decisions, store customer profile, run KYC |
| **Max Latency** | <500ms |
| **SLA** | 99.9% availability |

**Rules:**
- ✅ OTP validity: 10 minutes
- ✅ Max 3 OTP attempts per request
- ✅ Max 5 OTPs per customer per day
- ✅ Supports SMS & Email channels
- ❌ Does NOT store PII beyond mobile/email
- ❌ Does NOT verify KYC status

---

### 2. customer-service
| Aspect | Details |
|--------|---------|
| **Primary Responsibility** | Customer profile master data, de-duplication |
| **Owns Database** | PostgreSQL |
| **Owns Data** | customer_id, pan_hash, name, email, mobile, dob |
| **Uses SDKs** | messaging-sdk |
| **Consumed Events** | (upstream, no consumption) |
| **Emitted Events** | customer.created, customer.updated, customer.deactivated |
| **Can Call Services** | None (exposes API for reads only) |
| **Cannot Do** | Run KYC, manage OTP, make eligibility decisions |
| **Max Latency** | <200ms |
| **SLA** | 99.95% availability |

**Rules:**
- ✅ PAN-based de-duplication (one customer per PAN)
- ✅ All PII fields encrypted at rest
- ✅ Immutable customer_id
- ✅ Supports profile updates (email, mobile)
- ❌ Does NOT store raw PAN or Aadhaar
- ❌ Does NOT run identity verification

---

### 3. kyc-service (NEW)
| Aspect | Details |
|--------|---------|
| **Primary Responsibility** | KYC workflow orchestration, identity verification |
| **Owns Database** | PostgreSQL |
| **Owns Data** | application_id, kyc_status, kyc_attempts, failure_reason |
| **Uses SDKs** | kyc-sdk, messaging-sdk |
| **Consumed Events** | application.in_progress (start KYC) |
| **Emitted Events** | kyc.initiated, kyc.in_progress, kyc.verified, kyc.failed |
| **Can Call Services** | None (async via events) |
| **Cannot Do** | Store Aadhaar, store face images, make approval decisions |
| **Max Latency** | 5-30 seconds (async, vendor-dependent) |
| **SLA** | 99.5% availability (vendor-dependent) |

**Rules:**
- ✅ Async vendor integration with webhook callbacks
- ✅ Supports PAN, Aadhaar, face match workflows
- ✅ Retry logic via workflow-service
- ✅ Masks vendor responses in logs
- ✅ Immutable KYC status once verified
- ❌ Never stores Aadhaar data (in-memory only)
- ❌ Never stores face images
- ❌ Does NOT approve/reject (workflow-service decides)

---

### 4. application-service
| Aspect | Details |
|--------|---------|
| **Primary Responsibility** | Loan application state machine, lifecycle ownership |
| **Owns Database** | PostgreSQL |
| **Owns Data** | application_id, current_state, state_history, customer_id |
| **Uses SDKs** | messaging-sdk |
| **Consumed Events** | All domain events (for audit trail) |
| **Emitted Events** | application.state_changed (for each transition) |
| **Can Call Services** | None (state owner, not orchestrator) |
| **Cannot Do** | Make business decisions, call vendors, run eligibility logic |
| **Max Latency** | <100ms |
| **SLA** | 99.99% availability |

**Rules:**
- ✅ ONLY service allowed to change application state
- ✅ State transitions are immutable once committed
- ✅ Supports state versioning (FLOW_V1, FLOW_V2)
- ✅ Handles re-application as new application_id
- ✅ Terminal states are permanent
- ❌ Does NOT compute eligibility
- ❌ Does NOT verify vendors
- ❌ Does NOT read from other service databases

---

### 5. income-service
| Aspect | Details |
|--------|---------|
| **Primary Responsibility** | Income & employment verification |
| **Owns Database** | PostgreSQL |
| **Owns Data** | application_id, monthly_income, employment_type, verification_status |
| **Uses SDKs** | bank-sdk, epfo-sdk, messaging-sdk |
| **Consumed Events** | application.in_progress (trigger income check) |
| **Emitted Events** | income.verified, income.verification_failed |
| **Can Call Services** | None (async via events) |
| **Cannot Do** | Store bank statements, make eligibility decisions |
| **Max Latency** | 2-10 seconds (vendor-dependent) |
| **SLA** | 99.5% availability |

**Rules:**
- ✅ Fetches bank statements via bank-sdk
- ✅ Verifies EPFO records via epfo-sdk
- ✅ Classifies employment type (salaried, self-employed, etc.)
- ✅ Stores snapshot, not raw statements
- ✅ Retries on vendor timeout
- ❌ Does NOT store raw bank statements
- ❌ Does NOT compute FOIR/IIR (eligibility-service does)
- ❌ Does NOT approve/reject

---

### 6. credit-service
| Aspect | Details |
|--------|---------|
| **Primary Responsibility** | Bureau data fetch & credit interpretation |
| **Owns Database** | PostgreSQL |
| **Owns Data** | application_id, credit_score, dpd_status, bureau_vintage, adverse_flags |
| **Uses SDKs** | bureau-sdk, messaging-sdk |
| **Consumed Events** | income.verified (trigger bureau check) |
| **Emitted Events** | credit.verified, credit.verification_failed |
| **Can Call Services** | None (async via events) |
| **Cannot Do** | Store raw bureau report, approve/reject |
| **Max Latency** | 3-5 seconds |
| **SLA** | 99.5% availability |

**Rules:**
- ✅ Fetches bureau score via bureau-sdk
- ✅ Interprets credit risk (DPD, vintage, flags)
- ✅ Stores parsed data only (not raw report)
- ✅ Masks raw bureau response in logs
- ✅ Supports rate-limit backoff & retry
- ❌ Does NOT approve/reject (workflow-service decides)
- ❌ Does NOT compute FOIR/IIR
- ❌ Does NOT store raw bureau report

---

### 7. eligibility-service
| Aspect | Details |
|--------|---------|
| **Primary Responsibility** | FOIR/IIR calculation, eligibility determination |
| **Owns Database** | PostgreSQL (decision snapshot) |
| **Owns Data** | application_id, foir, iir, eligible_amount, max_tenure, decision |
| **Uses SDKs** | messaging-sdk |
| **Consumed Events** | income.verified, credit.verified |
| **Emitted Events** | eligibility.approved, eligibility.rejected |
| **Can Call Services** | None (pure computation) |
| **Cannot Do** | Call vendors, approve/reject directly, store business rules |
| **Max Latency** | <100ms |
| **SLA** | 99.99% availability |

**Rules:**
- ✅ Computes FOIR based on salary & existing liabilities
- ✅ Computes IIR based on monthly income
- ✅ Applies credit policy thresholds from config-service
- ✅ Stores decision snapshot for audit
- ✅ Deterministic computation (no randomness)
- ❌ Does NOT call vendors
- ❌ Does NOT make policy decisions (uses config-service rules)
- ❌ Does NOT approve/reject directly (workflow-service decides)

---

### 8. offer-service
| Aspect | Details |
|--------|---------|
| **Primary Responsibility** | Loan offer generation based on eligibility |
| **Owns Database** | PostgreSQL |
| **Owns Data** | application_id, eligible_amount, tenure_options, interest_rate, emi |
| **Uses SDKs** | messaging-sdk |
| **Consumed Events** | eligibility.approved |
| **Emitted Events** | offer.generated, offer.expired |
| **Can Call Services** | None (reads config-service via SDK) |
| **Cannot Do** | Store business rules, approve/reject |
| **Max Latency** | <200ms |
| **SLA** | 99.95% availability |

**Rules:**
- ✅ Generates offers based on approved eligibility
- ✅ Applies product-level caps (min/max tenure, max amount)
- ✅ Calculates EMI based on interest rate
- ✅ Supports multiple tenure options
- ✅ Offer expires after configured period
- ✅ Deterministic based on eligibility
- ❌ Does NOT make approval decisions
- ❌ Does NOT store business rules

---

### 9. sanction-service
| Aspect | Details |
|--------|---------|
| **Primary Responsibility** | Sanction letter generation, eSign orchestration |
| **Owns Database** | PostgreSQL + S3 for documents |
| **Owns Data** | application_id, sanctioned_amount, tenure, interest_rate, esign_status, document_ref |
| **Uses SDKs** | esign-sdk, messaging-sdk |
| **Consumed Events** | offer.accepted (customer accepts offer) |
| **Emitted Events** | sanction.initiated, sanction.esign_sent, sanction.completed, sanction.failed |
| **Can Call Services** | None (async via events) |
| **Cannot Do** | Store business rules, approve/reject |
| **Max Latency** | 5-10 seconds (eSign vendor-dependent) |
| **SLA** | 99.5% availability |

**Rules:**
- ✅ Generates sanction letter with loan terms
- ✅ Sends to eSign vendor for digital signature
- ✅ Stores signed document reference (not raw document)
- ✅ Handles eSign webhooks & callbacks
- ✅ Retries on eSign failures
- ✅ Masks vendor responses in logs
- ❌ Does NOT approve/reject (already approved)
- ❌ Does NOT make legal decisions

---

### 10. payment-verification-service (NEW)
| Aspect | Details |
|--------|---------|
| **Primary Responsibility** | Bank account validation & verification |
| **Owns Database** | PostgreSQL |
| **Owns Data** | application_id, ifsc, account_number_masked, verification_status, verification_method |
| **Uses SDKs** | bank-sdk, messaging-sdk |
| **Consumed Events** | sanction.completed (prepare for disbursement) |
| **Emitted Events** | payment.account_verified, payment.account_invalid |
| **Can Call Services** | None (async via events) |
| **Cannot Do** | Setup NACH, execute transfers, store raw account numbers |
| **Max Latency** | 2-3 seconds |
| **SLA** | 99.5% availability |

**Rules:**
- ✅ Validates IFSC code (format + master list)
- ✅ Validates account number format
- ✅ Supports NEFT test transaction verification
- ✅ Stores masked account number only
- ✅ Supports account verification retries
- ✅ Fails fast on invalid formats
- ❌ Does NOT setup NACH (disbursement-service does)
- ❌ Does NOT execute fund transfer

---

### 11. disbursement-service (SIMPLIFIED)
| Aspect | Details |
|--------|---------|
| **Primary Responsibility** | NACH mandate setup & fund transfer execution |
| **Owns Database** | PostgreSQL |
| **Owns Data** | application_id, loan_account_id, nach_status, disbursement_status, disbursal_timestamp |
| **Uses SDKs** | nach-sdk, payment-sdk, messaging-sdk |
| **Consumed Events** | sanction.completed, payment.account_verified |
| **Emitted Events** | disbursement.nach_initiated, disbursement.nach_confirmed, disbursement.fund_transferred, disbursement.completed, disbursement.failed |
| **Can Call Services** | None (async via events) |
| **Cannot Do** | Verify accounts (payment-verification-service does), approve/reject |
| **Max Latency** | 1-3 days (NACH is async) |
| **SLA** | 99.0% availability |

**Rules:**
- ✅ Creates NACH mandate via nach-sdk
- ✅ Waits for account verification before NACH setup
- ✅ Schedules fund transfer after NACH confirmation
- ✅ Executes transfer via payment-sdk
- ✅ Handles NACH rejection and retry scenarios
- ✅ Stores disbursement status for audit
- ❌ Does NOT verify bank accounts
- ❌ Does NOT approve/reject
- ❌ Does NOT call eligibility logic

---

### 12. collection-service (DOCUMENTED)
| Aspect | Details |
|--------|---------|
| **Primary Responsibility** | Post-disbursement loan account lifecycle, repayment tracking |
| **Owns Database** | PostgreSQL (separate from application DB) |
| **Owns Data** | loan_account_id, emi_schedule, repayment_history, delinquency_days, collection_status |
| **Uses SDKs** | payment-sdk, messaging-sdk |
| **Consumed Events** | disbursement.completed (create loan account) |
| **Emitted Events** | collection.account_created, collection.emi_due, collection.repayment_received, collection.delinquent, collection.escalated, collection.closed |
| **Can Call Services** | None (async via events) |
| **Cannot Do** | Approve/reject loans, change loan terms, make recovery decisions |
| **Max Latency** | <500ms |
| **SLA** | 99.9% availability |

**Rules:**
- ✅ Creates loan account after disbursement
- ✅ Generates EMI schedule based on sanction amount & tenure
- ✅ Tracks daily repayment status
- ✅ Calculates DPD (Days Past Due) daily
- ✅ Emits escalation events based on DPD thresholds
- ✅ Supports manual/online repayment processing
- ✅ Immutable EMI schedule (no mid-journey adjustments)
- ❌ Does NOT approve/reject loans
- ❌ Does NOT change loan terms
- ❌ Does NOT make write-off decisions (policy-driven)

---

### 13. notification-service
| Aspect | Details |
|--------|---------|
| **Primary Responsibility** | Customer communication (SMS, Email, Push) |
| **Owns Database** | Optional (notification log) |
| **Owns Data** | notification_id, customer_id, notification_type, delivery_status |
| **Uses SDKs** | sms-sdk, email-sdk, messaging-sdk |
| **Consumed Events** | Select events (application milestones, payment due, etc.) |
| **Emitted Events** | notification.sent, notification.delivered, notification.failed |
| **Can Call Services** | None (subscriber to events) |
| **Cannot Do** | Make business decisions, approve/reject, call vendor APIs directly |
| **Max Latency** | <5 seconds |
| **SLA** | 99.0% availability |

**Rules:**
- ✅ Consumes application milestones (offer, sanction, disbursal)
- ✅ Sends notifications via sms-sdk & email-sdk
- ✅ Supports templating with masking
- ✅ Handles delivery failures gracefully
- ✅ Respects customer opt-in preferences
- ✅ Logs all notifications for audit
- ❌ Does NOT store customer data
- ❌ Does NOT make business decisions

---

### 14. workflow-service (Camunda)
| Aspect | Details |
|--------|---------|
| **Primary Responsibility** | Loan journey orchestration & automated decision enforcement |
| **Owns Database** | Camunda DB |
| **Owns Data** | Workflow instances, execution state, decision history |
| **Uses SDKs** | messaging-sdk |
| **Consumed Events** | All domain events (to trigger workflow steps) |
| **Emitted Events** | workflow.approved, workflow.rejected, workflow.escalated |
| **Can Call Services** | Via REST adapters (as configured in workflow) |
| **Cannot Do** | Store domain data, make final decisions, call vendors directly |
| **Max Latency** | <1 second |
| **SLA** | 99.95% availability |

**Rules:**
- ✅ Defines loan journey orchestration
- ✅ Manages retries, waits, and compensation
- ✅ Enforces business rules via policy (not code)
- ✅ Triggers service actions via REST/messaging
- ✅ Handles saga compensation on failure
- ✅ Supports workflow versioning
- ✅ Policy changes without redeployment
- ❌ Does NOT store domain data
- ❌ Does NOT make approval decisions (just orchestrates)
- ❌ Does NOT call vendors

---

### 15. config-service
| Aspect | Details |
|--------|---------|
| **Primary Responsibility** | Platform & operational configuration management |
| **Owns Database** | PostgreSQL + Redis (for caching) |
| **Owns Data** | Configuration keys, values, versions, metadata |
| **Uses SDKs** | messaging-sdk |
| **Consumed Events** | (upstream) |
| **Emitted Events** | config.changed (for live reload) |
| **Can Call Services** | None (read-only API) |
| **Cannot Do** | Store business rules, approve/reject, make decisions |
| **Max Latency** | <50ms |
| **SLA** | 99.99% availability |

**Rules:**
- ✅ Stores vendor URLs, timeouts, retry counts
- ✅ Stores feature flags & toggles
- ✅ Supports environment-specific overrides (dev/staging/prod)
- ✅ Versioning for config changes
- ✅ Audit trail on all changes
- ✅ Four-eyes approval for production changes
- ✅ Live reload without restarts
- ❌ Does NOT store business rules (goes to Camunda)
- ❌ Does NOT store credit policy (goes to Camunda)
- ❌ Does NOT make decisions

---

### 16. audit-service
| Aspect | Details |
|--------|---------|
| **Primary Responsibility** | Immutable audit trail & decision logging |
| **Owns Database** | Append-only DB (Cassandra or similar) |
| **Owns Data** | Audit log entries (immutable), masked vendor responses |
| **Uses SDKs** | messaging-sdk (receives from Kafka only) |
| **Consumed Events** | ALL domain events (from kafka.audit topic) |
| **Emitted Events** | None |
| **Can Call Services** | None (passive listener only) |
| **Cannot Do** | Delete records, modify logs, make decisions |
| **Max Latency** | <1 second |
| **SLA** | 99.99% availability |

**Rules:**
- ✅ Consumes from Kafka (not direct SDK calls)
- ✅ Immutable, append-only storage
- ✅ Tamper-evident (checksums or signatures)
- ✅ Masking of sensitive data (PII, Aadhaar, etc.)
- ✅ Retention: indefinite
- ✅ Supports regulatory audits
- ✅ Query API for compliance/forensics
- ❌ Does NOT delete records
- ❌ Does NOT modify logs
- ❌ Does NOT call vendor APIs

---

---

## II. SDK RESPONSIBILITY MATRIX

### Common SDKs

| SDK | Purpose | Owns | Does NOT Own |
|---|---|---|---|
| **messaging-sdk** | Transactional Outbox + Kafka publishing | Event emission, Outbox writes, retry logic | Business logic, domain decisions |
| **vendor-observability-sdk** | Request tracing & PII masking | Spans, logs, masking rules | Domain logic, business decisions |
| **idempotency-sdk** | Request deduplication | Idempotency keys, tracking | Business logic, decision-making |

### Vendor SDKs

| SDK | Abstracts | Uses | Does NOT Do |
|---|---|---|---|
| **kyc-sdk** | PAN/Aadhaar/face match vendors (ULI, Karza, etc.) | Interface, mock adapter, real adapter | Store Aadhaar, approve/reject, business logic |
| **bank-sdk** | Bank aggregators (ULI, Ignosis, etc.) | Interface, statement fetch, salary calc | Store statements, approve/reject |
| **epfo-sdk** | EPFO employment verification | Interface, employment lookup | Store data, approve/reject |
| **bureau-sdk** | Bureau vendors (CRIF Highmark, etc.) | Interface, score fetch, report parse | Approve/reject, make decisions |
| **esign-sdk** | eSign vendors (ULI, SignDesk, etc.) | Interface, document signing, status | Store documents, approve/reject |
| **nach-sdk** | NACH mandate vendors (CAMS, Razorpay, etc.) | Interface, mandate creation, status | Approve/reject, fund transfer |
| **payment-sdk** | Payment gateways (mocked in architecture) | Interface, mock disbursement, payment tracking | Real money movement (mocked) |
| **sms-sdk** | SMS vendors | Interface, SMS sending, delivery tracking | Business decisions, approval logic |
| **email-sdk** | Email vendors | Interface, email sending, delivery tracking | Business decisions, approval logic |

---

## III. SERVICE INTERACTION RULES (STRICT)

### 1. Synchronous API Calls (REST)
**Allowed:**
- ✅ API Gateway → Any Service (read commands)
- ✅ Workflow-service → Service REST adapters (via Camunda)
- ✅ Services → Config-service (read config)

**Forbidden:**
- ❌ Service → Service direct DB access
- ❌ Service → Vendor direct calls (must use SDK)
- ❌ Service → Audit-service direct calls (use Kafka)

### 2. Asynchronous Communication (Kafka)
**Allowed:**
- ✅ All domain services → Kafka events (via messaging-sdk)
- ✅ Audit-service ← Kafka events (reads only)
- ✅ Notification-service ← Kafka events (reads only)

**Forbidden:**
- ❌ Kafka topics used for command/request (only facts/events)
- ❌ Consumer-to-producer coupling

### 3. Data Access Rules
**Allowed:**
- ✅ Each service reads its own database
- ✅ Services read other service data via REST API
- ✅ Read-only API calls to fetch data

**Forbidden:**
- ❌ Direct database access across services
- ❌ Service A writes to Service B's database
- ❌ Shared database schemas

---

## IV. Non-Negotiable Design Constraints

| Constraint | Enforced By | Violation Impact |
|---|---|---|
| SDKs have NO business logic | Code review | Vendor coupling, harder to switch |
| Audit-service is append-only | DB schema | Regulatory non-compliance |
| Messaging is NOT audit | Architecture | Audit gaps, missing decisions |
| Config-service has NO policy logic | Code review | Risk team blocked by deployments |
| Workflow-service does NOT decide | Process design | Hidden business logic |
| Identity-service does NOT profile | API contracts | Profile/auth confusion |
| Application-service is ONLY state owner | Versioned contracts | State inconsistency |
| No vendor calls outside SDKs | Code review | Vendor lock-in, hard to switch |
| All state is database-backed | Architecture | Rebuild/recovery issues |
| Events are immutable facts | Kafka schema | Audit corruption |

---

## V. Responsibility Summary Table

| Service | Single Responsibility | Critical Rule | SLA |
|---|---|---|---|
| identity-service | OTP/session management | No profile storage | 99.9% |
| customer-service | Profile master data | PAN de-duplication | 99.95% |
| kyc-service | KYC verification workflow | No Aadhaar storage | 99.5% |
| application-service | Loan state ownership | Only state owner | 99.99% |
| income-service | Income verification | No statement storage | 99.5% |
| credit-service | Bureau & credit scoring | No approval authority | 99.5% |
| eligibility-service | FOIR/IIR computation | Pure deterministic logic | 99.99% |
| offer-service | Offer generation | No rule storage | 99.95% |
| sanction-service | Sanction & eSign | No legal decisions | 99.5% |
| payment-verification-service | Bank account validation | No NACH setup | 99.5% |
| disbursement-service | NACH + fund transfer | No account verification | 99.0% |
| collection-service | EMI schedule & repayment | No approval authority | 99.9% |
| notification-service | Customer communication | No business logic | 99.0% |
| workflow-service | Orchestration | No domain data storage | 99.95% |
| config-service | Platform configuration | No policy logic | 99.99% |
| audit-service | Immutable audit trail | Append-only only | 99.99% |

---

**Document Status:** FINAL - Architecture Review Complete  
**Effective Date:** January 24, 2026
