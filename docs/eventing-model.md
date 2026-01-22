# Eventing Model & Messaging Strategy

This document defines the asynchronous communication model used across the
Digital Loan Origination Platform.

The eventing system is designed for:
- long-running loan workflows
- vendor isolation
- auditability and explainability
- safe recovery and replay

This platform is **not event-sourced**.
The database is the single source of truth.

---

## 1. Eventing Philosophy

- Database state is authoritative
- Events are derived facts, not state
- Events notify what happened, not what should happen
- Events are immutable and append-only
- No event is emitted without a committed database state
- Replay is engineering-controlled, never Ops-controlled

Events exist to **decouple services**, not to replace domain ownership.

---

## 2. Generic Event Schema

All events follow a single generic envelope.

```json
{
  "eventId": "uuid",
  "eventType": "INCOME_VERIFIED",
  "aggregateType": "LOAN_APPLICATION",
  "aggregateId": "loanApplicationId",
  "eventVersion": 1,
  "timestamp": "ISO-8601",
  "sourceService": "income-service",
  "correlationId": "trace-id",
  "payload": {}
}
```

### Field Semantics

- `eventId` – unique identifier for the event

- `eventType` – business fact that occurred

- `aggregateId` – loan application identifier

- `aggregateType` – domain aggregate (e.g. LOAN_APPLICATION)

- `eventVersion` – schema version for evolution

- `sourceService` – service that emitted the event

- `correlationId` – end-to-end trace identifier

- `payload` – event-specific data


Payload structure is not enforced at broker level to avoid tight coupling.

---

## 3. Event Immutability Rules

- Events are never updated

- Events are never deleted

- Corrections are expressed using new events


### Example

`INCOME_VERIFIED INCOME_VERIFICATION_REVOKED INCOME_VERIFIED_V2`

This preserves a complete historical timeline for audits and investigations.

---

## 4. Event Emission Guarantee (Transactional Outbox)

To avoid inconsistencies between database state and Kafka, the platform uses  
the **Transactional Outbox Pattern**.

### Emission Flow

1. Domain state is updated in the service database

2. Event is written to `outbox_events` table in the same transaction

3. Transaction commits successfully

4. Background publisher reads outbox records

5. Events are published to Kafka

6. Outbox records are marked as published


### Guarantees

- No event without committed state

- No committed state without eventual event

- Crash-safe and retry-safe delivery


---

## 5. Kafka Topic Strategy

Topics are **domain-oriented**, not service-oriented.

### Example Topics

```
loan.application.events 
kyc.events 
income.events 
bureau.events 
eligibility.events 
payment.events` 
```

Each topic may contain multiple event types related to that domain.

---

## 6. Failure Handling & Dead Letter Queues

### DLQ Strategy

- Each service owns its own DLQ

- DLQs are never shared

- Naming convention:


`<service-name>.dlq`

### Events are sent to DLQ when:

- Deserialization fails

- Business validation fails and is non-retryable

- Retry limits are exhausted


DLQs are diagnostic tools, not recovery mechanisms.

---

## 7. Event Consumption Rules

All event consumers must:

- Be idempotent

- Safely handle duplicate events

- Respect ordering per `aggregateId`

- Never assume event completeness

- Never write to another service’s database


Consumers react to events; they do not coordinate workflows.

---

## 8. Replay Policy

- Events are retained per Kafka retention configuration

- Replay is **engineering-controlled only**

- Ops-triggered replay is explicitly forbidden

- Replay requires:

    - dedicated consumer groups

    - scoped time windows

    - explicit audit logging


Replay is treated as a production change.

---

## 9. Operational Guarantees

This eventing model guarantees:
- strong consistency between state and events
- clear service ownership
- predictable failure handling
- audit-grade traceability


The design intentionally avoids over-engineering while remaining robust  
enough for real-world regulated lending platforms.