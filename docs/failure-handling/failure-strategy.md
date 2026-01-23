# Failure Strategy & Recovery Model

This document defines how failures are handled across the Digital Loan
Origination Platform.

The strategy is designed to:
- minimize unnecessary rejections
- control vendor costs
- preserve completed work
- maintain safe execution
- maintain audit and regulatory safety

Failures are treated as **expected system behavior**, not exceptions.

---

## 1. Core Failure Principles

- All stages are **retryable first**
- Business failures and technical failures are treated differently
- Successful stages are never rolled back
- Rejection is a **business decision**, not a system error
- System recovery is automated and self-healing

---

## 2. Failure Classification

### 2.1 Technical Failures
Failures caused by infrastructure or third-party systems.

Examples:
- Vendor timeouts
- Network errors
- Temporary unavailability
- Rate limiting

**Handling**
- Retry based on stage-specific configuration
- Exponential backoff
- Escalate to workflow-level retry on exhaustion

---

### 2.2 Business Failures
Failures caused by policy or data conditions.

Examples:
- Salary below threshold
- Bureau score below minimum
- DPD violations
- Age or eligibility breaches

**Handling**
- No technical retries
- Loan marked for rejection decision
- Clear business reason recorded

---

## 3. Retry Strategy

### 3.1 Retry Philosophy
The platform follows a **stage-dependent, conservative retry strategy**.

- Retry only when recovery is likely
- Avoid aggressive retries that increase vendor cost
- Fail fast on deterministic business failures

---

### 3.2 Retry Ownership (Hybrid Model)

| Layer | Responsibility |
|----|---------------|
| Service | Short retries (timeouts, transient errors) |
| Workflow (Camunda) | Long retries, waiting, escalation |

Services never block indefinitely.

---

### 3.3 Configurable Retry Controls

Each stage supports configuration via `config-service`:
- max retry attempts
- retry interval
- backoff strategy
- terminal failure threshold

No retry logic is hardcoded.

---

## 4. Partial Completion Rules (No Rollback)

Completed stages are **never undone**.

### Example
- INCOME_VERIFIED ✅
- BUREAU_FAILED ❌

Behavior:
- Income remains verified
- Bureau stage is retried or escalated
- Loan journey pauses, not restarts

This avoids:
- repeated vendor cost
- inconsistent customer experience
- state corruption

---

## 5. Rejection Handling

- Rejection occurs only after:
    - retries are exhausted, or
    - a non-recoverable business failure occurs
- Rejection is terminal for the loan application
- Re-application requires a new loanApplicationId

Each rejection includes:
- standardized reason code
- human-readable explanation
- timestamp and decision source

---

## 6. Customer-Facing Error Messaging

Customers see **business-friendly messages only**.

### Characteristics
- No technical terms
- No vendor names
- No internal error codes

### Example
> “We are unable to proceed with your loan at this time due to eligibility criteria. You may reapply later.”

Technical details are retained internally for audit and system diagnostics.

---

## 7. Why This Strategy Works

- Minimizes false rejections
- Controls vendor spend
- Preserves successful work
- Supports regulatory audits

This failure strategy reflects **real-world digital lending systems** operating
under regulatory and operational constraints.
