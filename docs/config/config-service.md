# Config Service – Policy & Platform Configuration

This document defines the design and responsibility of the `config-service`
used in the Digital Loan Origination Platform.

The config-service exists to ensure:
- business and risk teams can change behavior without redeployments
- services remain environment-agnostic
- operational changes are safe, auditable, and reversible

The platform follows a **policy-over-code** philosophy with a clear separation
between **platform configuration** and **product policy rules**.

---

## 1. Responsibility Split (Very Important)

### What Config-Service Owns
The config-service owns **platform and operational configuration**, including:

- Vendor base URLs
- Vendor enable / disable toggles
- Retry counts
- Retry intervals
- Timeout values
- Feature toggles at service level

These configurations affect **how services operate**, not **how loans are approved**.

---

### What Config-Service Does NOT Own
The config-service does **not** own business or credit policy rules.

The following are intentionally excluded:
- FOIR percentage
- IIR percentage
- Minimum salary
- Age limits
- Loan amount caps
- Tenure rules
- DPD thresholds
- EMI calculation logic

These rules live in **Camunda workflow configuration**, where product policies
are defined and versioned.

---

## 2. Why This Split Exists

| Concern | Owner |
|------|------|
| Platform stability | config-service |
| Vendor control | config-service |
| Credit & risk policy | Camunda |
| Product-specific rules | Camunda |

This separation ensures:
- engineers are not blocked by policy changes
- risk teams do not depend on code deployments
- platform behavior remains consistent across products

---

## 3. Configuration Granularity

### Config-Service
- Global per service
- Same config applies regardless of product
- Environment-specific overrides supported (dev / staging / prod)

### Camunda Configuration
- Product-level
- Policy-level
- Rule-level
- Supports multiple products (e.g., salaried, non-salaried in future)

---

## 4. Configuration Versioning Strategy

- All configurations are versioned
- Each loan application is pinned to a specific config version
- New configuration versions apply only to **new loan applications**

### Example
- Loan A → config v1
- Loan B → config v2


In-flight loans are never affected by mid-journey config changes.

---

## 5. Change Authority & Approval Flow

Configuration changes follow a **controlled approval process**.

### Who Defines Changes
- Risk team
- Compliance team

### Approval Model
- Mandatory approval before activation
- Four-eyes principle enforced
- No direct production edits

### Execution
- Approved changes are deployed automatically
- System enforces scope and versioning

---

## 6. Validation Rules

The config-service performs **basic validation only**, including:
- required fields present
- data type correctness
- simple value sanity checks

Complex business validation is intentionally avoided to keep the service
lightweight and predictable.

---

## 7. Audit & Traceability

Every configuration change is fully audited.

Audit records include:
- previous value
- new value
- config key
- config version
- changed by
- approved by
- change reason
- timestamp

Audit logs are:
- immutable
- append-only
- retained indefinitely

---

## 8. Runtime Access Pattern

Services consume configuration using an **event-driven refresh model**.

### Flow
1. Config change is approved and activated
2. `CONFIG_UPDATED` event is published
3. Services refresh in-memory cache
4. New config applies to new requests only

This avoids:
- per-request config calls
- high latency
- tight coupling

---

## 9. Failure Handling

- If config-service is unavailable:
    - services fall back to last known good config
- No service blocks on config-service at runtime
- Invalid or missing config triggers fail-safe defaults

---

## 10. Why This Design Works

- Clear separation of concerns
- Safe and auditable changes
- No redeploys for operational updates
- Supports multiple products in future
- Scales with business complexity

The config-service is intentionally simple, controlled, and predictable —
making it safe to operate in regulated lending environments.
