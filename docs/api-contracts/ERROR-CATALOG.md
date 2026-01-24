# API Contracts & Error Catalog

**Effective Date:** January 24, 2026  
**Status:** FINAL - Architecture Review

---

## 1. Standardized Error Code Format

All errors follow the pattern: `<SERVICE>_<CATEGORY>_<CODE>`

Examples:
- `CUST_PROFILE_001` – Customer service, profile category, error 001
- `KYC_PAN_001` – KYC service, PAN verification, error 001
- `DISB_BANK_001` – Disbursement service, bank validation, error 001

---

## 2. Error Severity Levels

| Level | Retry | Terminal | Action |
|-------|-------|----------|--------|
| **HARD_FAIL** | ❌ Never | ✅ Yes | Immediate rejection, no retry |
| **SOFT_FAIL** | ✅ Auto (exponential backoff) | ❌ No | Automatic retry, escalate after threshold |
| **WARN** | ✅ Manual/scheduled | ❌ No | Log, alert, human review if repeated |
| **INFO** | N/A | ❌ No | Informational only, no action |

---

## 3. Service-Specific Error Codes

### Customer Service (CUST_*)

| Code | Message | Category | Severity | Retry |
|------|---------|----------|----------|-------|
| `CUST_PROFILE_001` | Invalid mobile number format | Validation | HARD_FAIL | ❌ |
| `CUST_PROFILE_002` | Invalid email address format | Validation | HARD_FAIL | ❌ |
| `CUST_PROFILE_003` | Invalid date of birth (age out of range) | Validation | HARD_FAIL | ❌ |
| `CUST_DUP_001` | Customer already exists (duplicate PAN) | Business | HARD_FAIL | ❌ |
| `CUST_DB_001` | Database unavailable | Technical | SOFT_FAIL | ✅ |
| `CUST_DB_002` | Database timeout | Technical | SOFT_FAIL | ✅ |

**Customer-Facing Message:**
- All above → "Unable to process your request. Please try again later."

---

### Identity Service (IDENTITY_*)

| Code | Message | Category | Severity | Retry |
|------|---------|----------|----------|-------|
| `IDENTITY_OTP_001` | OTP send failed (SMS vendor error) | Technical | SOFT_FAIL | ✅ |
| `IDENTITY_OTP_002` | OTP send timeout | Technical | SOFT_FAIL | ✅ |
| `IDENTITY_OTP_003` | OTP invalid (expired or wrong) | Business | HARD_FAIL | ❌ |
| `IDENTITY_OTP_004` | Max OTP attempts exceeded | Business | HARD_FAIL | ❌ |
| `IDENTITY_RATE_001` | Rate limit exceeded (too many OTP requests) | Business | WARN | ✅ (after delay) |
| `IDENTITY_EMAIL_001` | Email send failed | Technical | SOFT_FAIL | ✅ |
| `IDENTITY_SESSION_001` | Session token invalid | Business | HARD_FAIL | ❌ |

**Customer-Facing Message:**
- `IDENTITY_OTP_001`, `IDENTITY_OTP_002` → "OTP delivery delayed. Please try again."
- `IDENTITY_OTP_003` → "OTP is incorrect or expired. Please request a new one."
- `IDENTITY_OTP_004` → "Too many attempts. Please try after some time."
- `IDENTITY_RATE_001` → "Please wait before requesting another OTP."

---

### KYC Service (KYC_*)

| Code | Message | Category | Severity | Retry |
|------|---------|----------|----------|-------|
| `KYC_PAN_001` | PAN verification failed (invalid PAN) | Business | HARD_FAIL | ❌ |
| `KYC_PAN_002` | PAN format invalid | Validation | HARD_FAIL | ❌ |
| `KYC_PAN_003` | PAN mismatch with name | Business | HARD_FAIL | ❌ |
| `KYC_AADHAAR_001` | Aadhaar verification failed | Business | HARD_FAIL | ❌ |
| `KYC_AADHAAR_002` | Aadhaar format invalid | Validation | HARD_FAIL | ❌ |
| `KYC_FACE_001` | Face match failed (low confidence) | Business | HARD_FAIL | ❌ |
| `KYC_FACE_002` | Liveness check failed (not a live person) | Business | HARD_FAIL | ❌ |
| `KYC_FACE_003` | Liveliness timeout | Technical | SOFT_FAIL | ✅ |
| `KYC_VENDOR_001` | Vendor unavailable | Technical | SOFT_FAIL | ✅ |
| `KYC_VENDOR_002` | Vendor rate limit exceeded | Technical | SOFT_FAIL | ✅ (with backoff) |
| `KYC_VENDOR_003` | Vendor timeout | Technical | SOFT_FAIL | ✅ |
| `KYC_MAX_ATTEMPTS` | Max KYC attempts exceeded | Business | HARD_FAIL | ❌ |

**Customer-Facing Message:**
- All KYC failures → "Identity verification could not be completed. Please try again later."
- `KYC_MAX_ATTEMPTS` → "You have exceeded maximum verification attempts. Please contact support."

---

### Income Service (INCOME_*)

| Code | Message | Category | Severity | Retry |
|------|---------|----------|----------|-------|
| `INCOME_BANK_001` | Bank account not found | Business | HARD_FAIL | ❌ |
| `INCOME_BANK_002` | Multiple bank accounts found (ambiguous) | Business | HARD_FAIL | ❌ |
| `INCOME_SALARY_001` | Salary below minimum threshold | Business | HARD_FAIL | ❌ |
| `INCOME_EMPLOYMENT_001` | Employment not found / inactive | Business | HARD_FAIL | ❌ |
| `INCOME_EMPLOYMENT_002` | Employment type not supported | Business | HARD_FAIL | ❌ |
| `INCOME_VENDOR_001` | Bank vendor unavailable | Technical | SOFT_FAIL | ✅ |
| `INCOME_VENDOR_002` | Bank vendor rate limit | Technical | SOFT_FAIL | ✅ (with backoff) |
| `INCOME_VENDOR_003` | Bank vendor timeout | Technical | SOFT_FAIL | ✅ |
| `INCOME_EPFO_001` | EPFO record not found | Business | HARD_FAIL | ❌ |
| `INCOME_EPFO_002` | EPFO vendor unavailable | Technical | SOFT_FAIL | ✅ |
| `INCOME_INVALID_DURATION` | Income verification period insufficient | Business | HARD_FAIL | ❌ |

**Customer-Facing Message:**
- `INCOME_SALARY_001`, `INCOME_EMPLOYMENT_001`, `INCOME_INVALID_DURATION` → "Your profile does not meet eligibility criteria."
- Vendor errors → "We are processing your income verification. Please check back later."

---

### Credit Service (CREDIT_*)

| Code | Message | Category | Severity | Retry |
|------|---------|----------|----------|-------|
| `CREDIT_BUREAU_001` | Bureau check failed (vendor error) | Technical | SOFT_FAIL | ✅ |
| `CREDIT_BUREAU_002` | Bureau data unavailable | Technical | SOFT_FAIL | ✅ |
| `CREDIT_BUREAU_003` | Bureau rate limit exceeded | Technical | SOFT_FAIL | ✅ (with backoff) |
| `CREDIT_BUREAU_004` | Bureau timeout | Technical | SOFT_FAIL | ✅ |
| `CREDIT_SCORE_001` | Credit score below minimum | Business | HARD_FAIL | ❌ |
| `CREDIT_DPD_001` | Applicant has open DPD flags (delinquent) | Business | HARD_FAIL | ❌ |
| `CREDIT_DPD_002` | Applicant has recent DPD history | Business | HARD_FAIL | ❌ |
| `CREDIT_ADVERSE_001` | Applicant has adverse flags (suit-filed, wilful default) | Business | HARD_FAIL | ❌ |
| `CREDIT_INQUIRY_001` | Excessive recent credit inquiries | Business | HARD_FAIL | ❌ |

**Customer-Facing Message:**
- Vendor errors → "We are checking your credit profile. Please try again later."
- Business failures → "We are unable to process your loan at this time."

---

### Eligibility Service (ELIG_*)

| Code | Message | Category | Severity | Retry |
|------|---------|----------|----------|-------|
| `ELIG_FOIR_001` | FOIR exceeds maximum allowed | Business | HARD_FAIL | ❌ |
| `ELIG_IIR_001` | IIR exceeds maximum allowed | Business | HARD_FAIL | ❌ |
| `ELIG_AGE_001` | Applicant below minimum age | Business | HARD_FAIL | ❌ |
| `ELIG_AGE_002` | Applicant above maximum age | Business | HARD_FAIL | ❌ |
| `ELIG_COOL_001` | Cooling period not satisfied (recent rejection) | Business | HARD_FAIL | ❌ |
| `ELIG_INCOME_INVALID` | Income insufficient for requested amount | Business | HARD_FAIL | ❌ |
| `ELIG_MISSING_DATA` | Required data missing for eligibility check | Business | HARD_FAIL | ❌ |

**Customer-Facing Message:**
- All eligibility failures → "You do not meet our eligibility criteria at this time. You may reapply after 30 days."

---

### Payment Verification Service (PVER_*)

| Code | Message | Category | Severity | Retry |
|------|---------|----------|----------|-------|
| `PVER_IFSC_001` | Invalid IFSC code format | Validation | HARD_FAIL | ❌ |
| `PVER_IFSC_002` | IFSC code not found in bank master list | Business | HARD_FAIL | ❌ |
| `PVER_ACCOUNT_001` | Invalid account number format | Validation | HARD_FAIL | ❌ |
| `PVER_ACCOUNT_002` | Account number validation failed | Business | HARD_FAIL | ❌ |
| `PVER_BANK_001` | Bank not supported for disbursement | Business | HARD_FAIL | ❌ |
| `PVER_BANK_002` | Bank vendor unavailable | Technical | SOFT_FAIL | ✅ |
| `PVER_BANK_003` | Bank vendor timeout | Technical | SOFT_FAIL | ✅ |
| `PVER_TEST_TXN_001` | Account test transaction failed | Business | HARD_FAIL | ❌ |

**Customer-Facing Message:**
- Validation errors → "Bank details are invalid. Please verify and try again."
- Business failures → "Your bank account could not be verified. Please provide a different account."

---

### Disbursement Service (DISB_*)

| Code | Message | Category | Severity | Retry |
|------|---------|----------|----------|-------|
| `DISB_NACH_001` | NACH mandate creation failed | Technical | SOFT_FAIL | ✅ |
| `DISB_NACH_002` | NACH mandate pending (in progress) | Technical | WARN | ✅ (scheduled) |
| `DISB_NACH_003` | NACH mandate rejected by bank | Business | HARD_FAIL | ❌ |
| `DISB_NACH_004` | NACH mandate expired | Business | HARD_FAIL | ❌ |
| `DISB_PAYMENT_001` | Fund transfer failed (vendor error) | Technical | SOFT_FAIL | ✅ |
| `DISB_PAYMENT_002` | Fund transfer timeout | Technical | SOFT_FAIL | ✅ |
| `DISB_PAYMENT_003` | Insufficient balance (mocked scenario) | Business | HARD_FAIL | ❌ |
| `DISB_ACCOUNT_CLOSED` | Bank account closed or frozen | Business | HARD_FAIL | ❌ |
| `DISB_VENDOR_001` | Payment vendor unavailable | Technical | SOFT_FAIL | ✅ |

**Customer-Facing Message:**
- Vendor errors → "Fund transfer is in progress. Your loan will be disbursed shortly."
- Business failures → "We were unable to disburse your loan. Please contact support."

---

### Sanction Service (SANC_*)

| Code | Message | Category | Severity | Retry |
|------|---------|----------|----------|-------|
| `SANC_ESIGN_001` | eSign initiation failed | Technical | SOFT_FAIL | ✅ |
| `SANC_ESIGN_002` | eSign timeout | Technical | SOFT_FAIL | ✅ |
| `SANC_ESIGN_003` | eSign document generation failed | Technical | SOFT_FAIL | ✅ |
| `SANC_ESIGN_REJECTED` | Applicant rejected eSign document | Business | HARD_FAIL | ❌ |
| `SANC_ESIGN_EXPIRED` | eSign document link expired | Business | HARD_FAIL | ❌ |
| `SANC_VENDOR_001` | eSign vendor unavailable | Technical | SOFT_FAIL | ✅ |

**Customer-Facing Message:**
- Vendor errors → "Sanction letter delivery delayed. Please check your email."
- Business failures → "Loan sanction could not be processed. Please contact support."

---

### Collection Service (COLL_*)

| Code | Message | Category | Severity | Retry |
|------|---------|----------|----------|-------|
| `COLL_PAYMENT_001` | EMI payment failed | Technical | SOFT_FAIL | ✅ |
| `COLL_PAYMENT_002` | Insufficient funds (collection failure) | Business | WARN | ✅ (scheduled) |
| `COLL_ACCOUNT_INVALID` | Bank account invalid for repayment | Business | HARD_FAIL | ❌ |
| `COLL_SCHEDULE_ERROR` | EMI schedule calculation error | Technical | SOFT_FAIL | ❌ |

**Customer-Facing Message:**
- Payment failures → "Your EMI payment could not be processed. Please ensure sufficient balance."
- Account issues → "Your repayment account is invalid. Please update your bank details."

---

## 4. HTTP Status Codes Mapping

| HTTP Code | Scenario | Retry | Notes |
|-----------|----------|-------|-------|
| `200 OK` | Success | N/A | Operation completed successfully |
| `400 Bad Request` | Validation error | ❌ Never | Malformed input, fix & resubmit |
| `401 Unauthorized` | Missing/invalid auth | ❌ Never | Session expired, re-authenticate |
| `403 Forbidden` | Insufficient permissions | ❌ Never | User role mismatch |
| `404 Not Found` | Resource not found | ❌ Never | Entity does not exist |
| `409 Conflict` | State conflict | ❌ Never | State machine violation |
| `422 Unprocessable Entity` | Business validation failed | ❌ Never | Business rule violation (eligibility, credit, etc.) |
| `429 Too Many Requests` | Rate limit exceeded | ✅ (after delay) | Backoff and retry |
| `500 Internal Server Error` | Server error | ✅ (exponential) | Temporary issue, retry with backoff |
| `502 Bad Gateway` | Vendor/downstream error | ✅ (exponential) | External service unavailable |
| `503 Service Unavailable` | Service temporarily down | ✅ (exponential) | Maintenance or overload |
| `504 Gateway Timeout` | Vendor timeout | ✅ (exponential) | External service timeout |

---

## 5. Error Response Format (Standard)

All error responses follow this format:

```json
{
  "success": false,
  "error": {
    "code": "KYC_PAN_001",
    "message": "Identity verification could not be completed. Please try again later.",
    "category": "KYC",
    "severity": "HARD_FAIL",
    "retryable": false,
    "timestamp": "2026-01-24T10:30:00Z",
    "requestId": "req-12345-xyz",
    "details": {
      "fieldName": "pan",
      "reason": "PAN verification failed with vendor"
    }
  }
}
```

### Fields:
- **code** – Machine-readable error code
- **message** – Customer-friendly, non-technical message
- **category** – Service/domain (e.g., KYC, Income, Credit)
- **severity** – HARD_FAIL, SOFT_FAIL, WARN
- **retryable** – Whether client should retry
- **timestamp** – When error occurred
- **requestId** – For tracing/support
- **details** – Optional debugging info (internal use only, not shown to customers)

---

## 6. Retry Strategy Specification

### Service-Level Retries (SDK/Client)

```
Max Retries: 3
Initial Backoff: 100ms
Backoff Multiplier: 2.0
Max Backoff: 10 seconds

Attempt 1: 100ms
Attempt 2: 200ms
Attempt 3: 400ms
Attempt 4: Fail / escalate to workflow
```

### Workflow-Level Retries (Camunda)

```
Max Retries: 5
Initial Backoff: 1 minute
Backoff Multiplier: 1.5
Max Backoff: 1 hour

Attempt 1: 1m
Attempt 2: 1.5m
Attempt 3: 2.25m
Attempt 4: 3.375m
Attempt 5: 5m
Attempt 6: Escalate / manual review
```

### Vendor SDK Retries

```
Rate-Limit Errors: Exponential backoff (10s to 5m)
Timeout Errors: Exponential backoff (1s to 30s)
Server Errors (5xx): Exponential backoff (100ms to 10s)
Client Errors (4xx): No retry
```

---

## 7. API Versioning Strategy

### URL Versioning

```
POST /api/v1/applications
POST /api/v2/applications (future)
```

### Required Headers

```
Content-Type: application/json
Authorization: Bearer <token>
X-Request-ID: <unique-request-id>
X-API-Version: 1.0
```

### Backward Compatibility Rules

1. **All new services launch at v1**
2. **Breaking changes require major version bump**
3. **Additive changes maintain current version**
4. **Deprecated endpoints marked with `Deprecation` header**
5. **Sunset period: 6 months minimum**

---

## 8. Customer-Facing Rejection Messages (Standard)

| Situation | Message | Internal Code |
|-----------|---------|----------------|
| **Identity Verification Failed** | "Identity verification could not be completed. Please try again." | KYC_* |
| **Income Below Threshold** | "Your income does not meet our requirements." | INCOME_SALARY_001 |
| **Credit Score Too Low** | "We are unable to process your loan at this time due to credit profile." | CREDIT_SCORE_001 |
| **Eligibility Criteria Not Met** | "You do not meet our eligibility criteria. You may reapply after 30 days." | ELIG_* |
| **Technical Issue** | "We are processing your request. Please try again later." | *_VENDOR_* |
| **System Unavailable** | "Our system is temporarily unavailable. Please try again soon." | *_001, *_002 (Technical) |
| **Excessive Attempts** | "Too many attempts. Please try again after some time." | KYC_MAX_ATTEMPTS, IDENTITY_OTP_004 |

---

**Document Status:** FINAL - Architecture Review Complete  
**Effective Date:** January 24, 2026
