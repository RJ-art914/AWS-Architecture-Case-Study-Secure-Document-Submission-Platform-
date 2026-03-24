# 02 — Requirements

## Overview

This document defines the functional and non-functional requirements for the Secure AWS-Based Document Submission and Processing Platform. These requirements form the foundation of the architectural design and service selection decisions described in this case study.

---

## Stakeholders

Stakeholders, assumptions, and scope boundaries are defined in [01-business-scenario.md](docs/01-business-scenario.md). This document focuses on functional and non-functional requirements. 

---

## User Roles

Three distinct roles govern access and functionality within the platform:

- **Supplier** — Can authenticate, upload documents, and view the status of their own submissions.
- **Internal Reviewer** — Can authenticate, view all submitted documents and metadata, and update processing status.
- **Administrator** — Can manage user accounts, configure access controls, and monitor platform health.

---

## Functional Requirements

The following functional requirements define what the system must do.

| ID | Requirement |
|---|---|
| FR-01 | External suppliers must be able to authenticate securely before accessing any platform functionality. |
| FR-02 | Authenticated suppliers must be able to upload documents through a web-based interface. |
| FR-03 | The platform must store all uploaded documents securely in durable, managed cloud storage. |
| FR-04 | The system must capture and persist metadata for each submission, including: supplier name, document type, upload timestamp, and processing status. |
| FR-05 | Internal reviewers must be able to log in and view a list of all submitted documents and their associated metadata. |
| FR-06 | Internal reviewers must be able to update the review or processing status of a submission (e.g. *Received*, *Under Review*, *Approved*, *Rejected*). |
| FR-07 | The system must send automated notifications to relevant internal users when a new document is submitted. |
| FR-08 | Suppliers must receive confirmation that their document submission was completed successfully. |
| FR-09 | The platform must enforce role-based access control to ensure suppliers can only access their own submissions, while internal users can access all records. |

---

## Non-Functional Requirements

The following non-functional requirements define the quality attributes and constraints the system must satisfy.

| ID | Requirement | Category |
|---|---|---|
| NFR-01 | The platform must provide secure, managed authentication and authorization for all user groups. | Security |
| NFR-02 | All data must be protected in transit using HTTPS and at rest using managed encryption mechanisms. | Security |
| NFR-03 | The solution must rely on managed AWS services to minimize infrastructure maintenance and operational overhead. | Operability |
| NFR-04 | The architecture must scale to accommodate fluctuating document submission volumes without manual intervention. | Scalability |
| NFR-05 | The system must provide centralized logging, metrics, and alerting for operational visibility. | Observability |
| NFR-06 | The design should support high availability through the use of resilient managed services wherever appropriate. | Availability |
| NFR-07 | The architecture must remain cost-conscious and financially appropriate for a mid-sized business. | Cost Efficiency |
| NFR-08 | The solution must consider data protection expectations relevant to the German and EU regulatory context (GDPR). | Compliance |
| NFR-09 | The solution must be documented clearly enough for both technical and non-technical stakeholders to understand. | Maintainability |

---

## Assumptions

The following assumptions are made in the context of this case study:

- The initial solution is deployed in a single AWS region: **Europe (Frankfurt) — `eu-central-1`**, to align with latency expectations and data residency preferences for the German market.
- The company prefers managed AWS services over self-managed infrastructure to reduce operational burden.
- The first release focuses on the core document submission, storage, review, and notification workflow.
- Integration with external ERP or document management systems is out of scope for the initial version.
- The solution is designed for a mid-sized business — not a large global enterprise with complex multi-region requirements.
- The application handles sensitive business documents; therefore, security and access control are essential requirements, not optional enhancements.

---

## Scope

### In Scope

The first version of the platform includes:

- Secure user authentication for suppliers and internal users
- Web-based document upload workflow
- Secure and durable document storage
- Metadata capture and processing status tracking
- Internal reviewer access to submitted documents
- Automated notifications upon new document submission
- Centralized logging and monitoring
- Role-based access control
- Encryption in transit and at rest
- Basic web application security controls

### Out of Scope

The following items are explicitly excluded from the initial version:

- OCR or AI-based document analysis
- Deep workflow automation or multi-step approval chains
- ERP or third-party system integration
- Multi-region deployment or disaster recovery
- Advanced analytics dashboards or reporting
- Full legal compliance certification processes

---

## Summary

These requirements establish a clear, traceable foundation for the architecture design decisions described in subsequent documents. Each AWS service selected in the solution can be mapped back to one or more of the requirements listed above, ensuring that the design is driven by real business and operational needs rather than technology preference alone.
