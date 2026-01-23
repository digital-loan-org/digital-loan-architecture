# Event Catalog (Application Lifecycle)

## Objective

This document defines **all business events** emitted in the Digital Loan Platform.

The platform is:
- Fully self-serve
- Event-driven
- No human intervention
- Application-service centric

Events represent **facts that already happened**, not commands.

---

## Event Design Principles

1. Events are immutable facts
2. Events are emitted **after state change**
3. Events do NOT trigger direct DB updates in other services
4. Payloads are minimal and business-focused
5. Each event has a single producer

---

## Mandatory Event Fields (All Events)

Every event MUST include:

- event_id
- event_type
- application_id
- previous_state
- current_state
- occurred_at (ISO timestamp)

---

## Core Application Events

| Event Name | Producer | When Emitted | Description |
|----------|----------|-------------|-------------|
| application.created | application-service | Application initialized | Loan application created |
| lead.created | application-service | Lead generated | Lead captured from user |
| application.in_progress | application-service | Processing started | Application moved to processing |
| application.eligible | application-service | Eligibility success | Applicant is eligible |
| application.kyc_failed | application-service | KYC failed | Identity verification failed |
| application.credit_failed | application-service | Credit failed | Bureau rejection |
| application.eligibility_failed | application-service | Eligibility failed | FOIR/IIR failure |
| application.self_rejected | application-service | User dropped | User exited flow |
| application.sanctioned | application-service | Sanction done | Loan sanctioned |
| application.document_verification | application-service | Docs phase | Document verification started |
| application.disbursement_in_progress | application-service | Disbursement started | Funds being processed |
| application.disbursed | application-service | Disbursement success | Funds credited |
| application.closed | application-service | Lifecycle complete | Application closed |

---

## Downstream Consumption (Examples)

| Consumer | Events Consumed | Purpose |
|--------|----------------|--------|
| income-service | application.in_progress | Start income verification |
| credit-service | application.in_progress | Trigger bureau check |
| eligibility-service | income.verified, credit.verified | Compute eligibility |
| sanction-service | application.eligible | Generate sanction |
| disbursement-service | application.sanctioned | Start payout |
| collection-service | application.disbursed | Create loan account |
| audit-service | ALL events | Audit trail |
| notification-service | Select events | User notifications |

---

## Failure & Retry Events

| Event Name | Producer | Description |
|----------|----------|-------------|
| income.verification_failed | income-service | Salary verification failed |
| credit.verification_failed | credit-service | Bureau check failed |
| disbursement.failed | disbursement-service | Disbursement failure |

These events do NOT change application state directly.

---

## Forbidden Event Patterns

❌ Command-style events (e.g. `do_eligibility`)  
❌ SDK-emitted business events  
❌ State change without event emission  
❌ Multiple producers for same event

---

## Summary

- Events describe **what happened**
- application-service is the authority on lifecycle events
- Other services react, compute, and report results
- No event causes direct state mutation outside application-service

---

## Status

**Status:** Approved & Frozen  
**Last Updated:** Day 3 – Event Model Freeze
