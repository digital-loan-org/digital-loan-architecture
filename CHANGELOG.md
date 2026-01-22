# Changelog

All notable changes to the **digital-loan-architecture** repository are documented in this file.

This repository follows an **architecture-first approach**.  
Changes represent **design decisions, documentation, and diagrams**, not code.

---

## [1.0.0] - Architecture Baseline Finalized
**Date:** 2026-01-23
**Release Type:** Major

### 🎯 Summary
Established a complete, production-grade architectural foundation for a
Digital Loan Origination & Lifecycle Management Platform (Salaried).

This release marks the **end of the architecture and design phase** and
serves as the **frozen baseline** for all future implementation work.

---

### ✅ Added – Core Architecture Documents
- `architecture-overview.md`  
  High-level architecture style, execution model, and system guarantees.
- `service-landscape.md`  
  Clear service boundaries, responsibilities, and SDK usage.
- `loan-journey-states.md`  
  Versioned loan state machine with terminal rejection semantics.
- `eventing-model.md`  
  Generic event schema, transactional outbox, DLQ strategy, and replay rules.
- `failure-strategy.md`  
  Retry philosophy, failure classification, and ops intervention model.
- `security-and-compliance.md`  
  PII handling, encryption strategy, audit model, and regulatory alignment.
- `config-service.md`  
  Policy-over-code strategy and separation between platform config and product rules.

---

### 🧾 Added – Architecture Decision Records (ADRs)
- `ADR-001-architecture-style.md`  
  Decision on microservices, event-driven design, Camunda orchestration, and DB as source of truth.
- `ADR-002-eventing-model.md`  
  Decision on generic event envelope, transactional outbox, immutability, and replay control.

These ADRs capture **why** key architectural decisions were made and are intended
to remain stable over time.

---

### 📐 Added – Architecture Diagrams (Mermaid)
**C4 Diagrams**
- System Context diagram
- Container diagram

**Business Flow Diagrams**
- Loan happy path
- Loan rejection path
- Retry and escalation flow

**Sequence Diagrams**
- Loan creation
- Income verification
- Bureau check
- Disbursement
- Ops override (approve / reject)

All diagrams are:
- text-based (Mermaid)
- version-controlled
- GitHub-renderable
- aligned with documented architecture

---

### 🔐 Established – Core Design Principles
- Vendor abstraction via SDKs
- Event-driven, saga-based orchestration
- Fail-fast, cost-aware execution
- No rollback of successful stages
- Terminal rejection per loan application
- New loan ID required for re-application
- Immutable audit trail
- Policy changes without redeployments

---

### 🧭 Repository Status
- Architecture phase: **COMPLETE**
- Design baseline: **FROZEN**
- Implementation: **NOT STARTED**

Any future changes must:
- reference existing ADRs, or
- introduce new ADRs if decisions change.

---

### 🚀 Next Phase
This repository will be **forked** to begin hands-on implementation, including:
- API contracts (OpenAPI)
- Event contracts
- Service skeletons
- Incremental backend development

Architecture decisions documented here are the **single source of truth** for all downstream codebases.

---

### Versioning Notes
- **Major (x.0.0):** Backward-incompatible API or data model changes.
- **Minor (x.y.0):** Backward-compatible new features or structural improvements.
- **Patch (x.y.z):** Backward-compatible bug fixes, small optimizations, or hotfixes.
---
[1.0.0]: https://github.com/digital-loan-org/digital-loan-architecture/releases/tag/1.0.0
