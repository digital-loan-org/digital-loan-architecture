# Application State Transition Rules

## Objective

This document defines **all valid application states and transitions**
for the Digital Loan Platform.

The system is **100% self-serve**:
- No ops
- No risk team
- No manual overrides
- No human intervention

---

## Core Principles

- Only `application-service` can change application state
- SDKs and third parties can NEVER change state
- Every state change is validated, recorded, and evented
- Rejections are explicit and reason-specific

---

## Application States

### Active States
- CREATED
- LEAD_CREATED
- IN_PROGRESS
- ELIGIBLE
- SANCTIONED
- DOCUMENT_VERIFICATION
- DISBURSEMENT_IN_PROGRESS
- DISBURSED


### Terminal States
- KYC_FAILED
- CREDIT_FAILED
- ELIGIBILITY_FAILED
- SELF_REJECTED
- CLOSED


---

## State Transition Rules

| From State | To State |
|----------|---------|
| CREATED | LEAD_CREATED |
| LEAD_CREATED | IN_PROGRESS |
| IN_PROGRESS | KYC_FAILED |
| IN_PROGRESS | CREDIT_FAILED |
| IN_PROGRESS | ELIGIBILITY_FAILED |
| IN_PROGRESS | SELF_REJECTED |
| IN_PROGRESS | ELIGIBLE |
| ELIGIBLE | SANCTIONED |
| SANCTIONED | DOCUMENT_VERIFICATION |
| DOCUMENT_VERIFICATION | DISBURSEMENT_IN_PROGRESS |
| DISBURSEMENT_IN_PROGRESS | DISBURSED |
| DISBURSED | CLOSED |

---

## Forbidden Transitions

- DISBURSED → IN_PROGRESS ❌
- *_FAILED → ELIGIBLE ❌
- CLOSED → any other state ❌
- Any SDK-initiated state change ❌

---

## Rejection Rules

- Every rejection maps to exactly ONE terminal state
- Generic `REJECTED` state is NOT allowed
- Once rejected, application cannot auto-retry

---

## Collection Handover

- Collection is initiated only after `DISBURSED`
- EMI schedule is generated during `ELIGIBLE`
- Collection lifecycle is independent of application lifecycle

---

## Auditing Requirements

Every state change must record:
- application_id
- previous_state
- new_state
- timestamp
- system_actor (application-service)
- triggering_event
