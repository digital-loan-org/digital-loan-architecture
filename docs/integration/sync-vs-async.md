# Sync vs Async Execution Model

## Objective

This document defines **which operations are synchronous vs asynchronous**
in the Digital Loan Platform.

The system is:
- 100% self-serve
- Fully automated
- No human intervention
- Optimized for fast user response and resilience

---

## Core Principles

1. **User-blocking actions must be synchronous**
2. **Vendor calls must be asynchronous**
3. **Business decisions must never wait on slow external systems**
4. **Application state changes happen only after async completion**
5. **Failures in async flows are handled automatically**

---

## Decision Rule

> If the user is waiting on the screen → **SYNC**  
> If the system is calling a vendor or doing long work → **ASYNC**

---

## Sync Operations (User Waiting)

These operations respond immediately to the user.

| Step | Type | Reason |
|----|----|----|
| Application creation | Sync | Lightweight |
| Lead creation | Sync | UI continuity |
| User input validation | Sync | UX |
| Eligibility computation (FOIR/IIR) | Sync | In-house logic |
| Offer presentation | Sync | Deterministic |
| Sanction confirmation trigger | Sync | User action |

---

## Async Operations (Background)

These operations are long-running or vendor dependent.

| Step | Type | Reason |
|----|----|----|
| KYC verification | Async | External vendor |
| Income verification | Async | Bank aggregator |
| Credit bureau check | Async | Cost + latency |
| Document verification | Async | External dependency |
| eSign processing | Async | External system |
| NACH setup | Async | External system |
| Disbursement | Async | Banking rails |
| EMI collection | Async | Scheduled processing |

---

## Async Execution Model

1. application-service triggers async task
2. SDK or downstream service executes work
3. Result event is emitted
4. application-service validates result
5. application state is updated

**No async task can directly change state.**

---

## State Change Rules

- Only `application-service` updates state
- Async results are treated as **inputs**, not decisions
- SDK failures never change state directly

---

## Failure Handling (Async)

| Failure Type | Handling |
|------------|---------|
| Vendor timeout | Automatic retry |
| Vendor error | Mark failure state |
| Invalid response | Reject application |
| Retry exhausted | Terminal failure |

No manual retry or human escalation exists.

---

## Forbidden Patterns

❌ Vendor calls inside synchronous APIs  
❌ SDK-triggered state changes  
❌ Blocking UI on async vendor response  
❌ Human-driven retry or override

---

## Summary Table

| Category | Sync | Async |
|------|------|------|
| User interaction | ✅ | ❌ |
| Vendor calls | ❌ | ✅ |
| Business rules | ✅ | ❌ |
| State updates | ✅ | ❌ |
| Long-running tasks | ❌ | ✅ |

---

## Status

**Status:** Approved & Frozen  
**Last Updated:** Day 3 – Execution Model Freeze
