# Business Scenario

## Company Profile

A mid-sized manufacturing company based in Germany works with a network of external suppliers to source components, materials, and services critical to its operations. As part of its procurement and quality management processes, the company regularly requests and receives a variety of business-critical documents from these suppliers — including technical specifications, quality certificates, compliance declarations, and audit reports.

---

## Current Process and Its Problems

At present, document collection relies primarily on **email and manual file handling**. Suppliers send documents as email attachments, which are then downloaded, sorted, and stored by internal staff — often inconsistently and without a centralized system.

This approach creates a number of recurring problems:

| Problem | Impact |
|---|---|
| No centralized document repository | Files are scattered across inboxes and local folders |
| No submission tracking | Internal teams cannot easily see what has been received, reviewed, or is missing |
| Weak auditability | No structured record of who submitted what and when |
| Poor access control | Documents shared over email can be forwarded or accessed unintentionally |
| Manual effort | Internal staff spend time chasing submissions, renaming files, and updating spreadsheets |
| Scalability limits | As supplier numbers or submission volumes grow, the manual process breaks down |
| Compliance exposure | Informal document handling is difficult to defend in a regulated business environment |

---

## Business Problem Statement

> The company needs a structured, secure, and scalable digital platform on AWS that allows external suppliers to submit required documents through a controlled web-based workflow, while enabling internal teams to review submissions, track processing status, and receive automated notifications — all with minimal operational overhead and strong access controls.

---

## Business Objectives

The modernization effort is driven by the following goals:

1. **Replace manual email-based document collection** with a structured digital submission process
2. **Improve security and access control** for sensitive supplier documents
3. **Increase visibility** into submission status and document handling across the organization
4. **Reduce administrative burden** through automation and managed cloud services
5. **Strengthen auditability** to support compliance and internal governance requirements
6. **Build a scalable foundation** that can grow with supplier volumes and business needs
7. **Align with EU and German data protection expectations** by ensuring data residency and controlled access

---

## Stakeholders

The following groups are considered in the design of this solution:

| Stakeholder | Role in the System |
|---|---|
| **External Suppliers** | Submit required documents through the platform |
| **Internal Reviewers** | View submissions, validate documents, and update processing status |
| **Platform Administrators** | Manage user access, system configuration, and platform oversight |
| **Business Management** | Benefit from improved visibility, traceability, and process efficiency |
| **IT / Security Teams** | Responsible for operational stability, access governance, and compliance |

---

## Assumptions

This case study is based on the following assumptions:

- The initial solution is deployed in a **single AWS region**: Europe (Frankfurt) — `eu-central-1`
- The company prefers **managed AWS services** to minimize direct infrastructure management
- The first release focuses on document submission, storage, review workflows, and notifications
- Integration with external ERP or document management systems is **out of scope** for the initial version
- The solution handles business documents and metadata, making **secure storage and access control essential**
- The design is tailored to a **mid-sized organization**, not a large global enterprise with multi-region complexity

---

## Scope

### In Scope

The initial version of the platform includes:

- Secure user authentication for external suppliers and internal users
- Web-based document upload through a controlled submission workflow
- Centralized and durable document storage
- Metadata capture and processing status tracking per submission
- Internal review access with status update capability
- Automated notifications when new documents are submitted
- Role-based access control separating supplier and internal reviewer permissions
- Logging, monitoring, and basic auditability

### Out of Scope

The following items are explicitly excluded from the initial version:

- OCR or AI-based document analysis
- Deep workflow automation or complex multi-step approval chains
- ERP or third-party system integration
- Multi-region disaster recovery architecture
- Advanced analytics or reporting dashboards
- Full legal certification or formal compliance auditing processes

---

## Why This Scenario Is Realistic

Manual document handling via email remains a common challenge for mid-sized industrial companies, particularly in supply-chain-intensive sectors like manufacturing. A cloud-based submission and review platform represents a realistic and high-value modernization initiative — one that touches authentication, secure file storage, metadata management, event-driven notifications, and operational monitoring. This makes it an ideal scenario for demonstrating practical AWS architecture design skills.
