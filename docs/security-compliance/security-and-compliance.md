# Security & Compliance Model

This document defines the security, data protection, and compliance strategy
for the Digital Loan Origination Platform.

The platform is designed for regulated lending environments where:
- sensitive financial data is processed
- auditability is mandatory
- access must be strictly controlled

Security is treated as a **foundational system property**, not an afterthought.

---

## 1. Data Classification

The platform classifies data into the following categories:

### 1.1 Highly Sensitive PII
- Aadhaar number
- Aadhaar XML
  - Bank account number
- Bureau report raw data
- Salary details
- Bank statements

### 1.2 Sensitive PII
- PAN
- Mobile number
- Email address

### 1.3 Non-PII
- IFSC
- Loan identifiers
- System metadata

---

## 2. Aadhaar Data Handling (RBI-Aligned)

- Aadhaar data is **never stored**, even in masked form
- Aadhaar is used only **in-memory at runtime** for verification
- Aadhaar numbers, XML, and documents are discarded immediately after use
- No Aadhaar data is written to logs, databases, or audit stores

This ensures compliance with RBI and Aadhaar data handling guidelines.

---

## 3. Data Storage & Protection Rules

| Data Type | Storage Policy |
|---------|----------------|
| Aadhaar | Not stored |
| PAN | Stored encrypted |
| Bank account number | Stored encrypted |
| Mobile / Email | Stored encrypted |
| Bureau report | Stored encrypted |
| Salary & bank statements | Stored encrypted |
| Face image / selfie | Not persisted |

All encrypted fields use strong, industry-standard encryption algorithms.

---

## 4. Encryption Strategy

### 4.1 Encryption at Rest
- Mandatory for all databases
- Mandatory for all backups
- Mandatory for archived data

### 4.2 Encryption in Transit
- All service-to-service communication uses TLS
- Vendor integrations use secure channels only

### 4.3 Field-Level Encryption
- Applied to all PII fields
- Encryption occurs before persistence
- Decryption occurs only at authorized service boundaries

### 4.4 Key Management
- Encryption keys are managed internally
- Keys are stored securely and rotated periodically
- Access to keys is restricted and audited

---

## 5. Access Control Model

### 5.1 Customer Access
Customers can view:
- loan application status
- business-approved rejection reason

Customers cannot view:
- internal rules
- vendor responses
- technical failure details

---

### 5.2 Application Services Access
Application services can access:
- loan application data required for processing
- encrypted PII during processing
- decryption keys only at authorized service boundaries

Services cannot:
- directly access raw PII without encryption
- bypass audit logging
- replay events

---

### 5.3 Engineering Access
Engineers:
- do not have access to production PII
- can access logs, metrics, and traces only
- cannot query production databases directly

All access is role-based and authenticated.

---

## 6. Audit & Traceability

The platform maintains a **complete, immutable audit trail**.

Audited actions include:
- every loan state transition
- every vendor request and response (masked)
- every eligibility and policy decision
- every system decision and action
- every configuration change

Audit logs are:
- append-only
- immutable
- time-ordered
- tamper-resistant

Audit data is never deleted.

---

## 7. Data Retention & Archival

### 7.1 Active Data
- Loan data remains active while the loan is in progress
- All associated documents and decisions are retained

### 7.2 Inactive Loans
Loans that are:
- rejected, or
- completed and closed

and have had **no activity for more than 1 year** are moved to archival storage.

### 7.3 Archived Data
Archived data includes:
- loan application data
- KYC records
- bank verification data
- audit logs

Archived data:
- remains encrypted
- is read-only
- is retained indefinitely for compliance

No data is permanently deleted.

---

## 8. Logging & Monitoring Controls

- Logs must never contain raw PII
- Sensitive fields are masked or excluded from logs
- Correlation IDs are used for tracing
- Security events are monitored and alerted

---

## 9. Compliance Alignment

The platform is designed to align with:
- IT Act (India)
- RBI-aligned data handling practices
- Internal compliance and audit requirements

Security and compliance requirements are treated as **non-negotiable constraints**.

---

## 10. Security Design Guarantees

This model guarantees:
- protection of sensitive customer data
- controlled and auditable access
- regulatory defensibility
- safe long-term data retention

The system is designed to be secure by default and auditable by design.
