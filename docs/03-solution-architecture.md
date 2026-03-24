# 03 — Solution Architecture

## Overview

This document describes the technical architecture of the secure supplier document submission platform. It covers the high-level design, AWS service selection with rationale, the end-to-end data flow, user roles, and the key design decisions and trade-offs that shaped the solution.

The architecture follows a **serverless, managed-services-first approach** deployed in the **AWS Europe (Frankfurt) region — eu-central-1**. This region was chosen to support low latency for German users and to align with EU data residency expectations.

---

## Architectural Approach

The platform is built entirely on managed AWS services. There are no virtual machines or self-managed servers. This decision was driven by the goal of minimizing operational overhead for a mid-sized organization that does not maintain a dedicated cloud infrastructure team.

The core components are:

- A static web frontend delivered through a CDN
- Managed authentication for suppliers and internal users
- A serverless API and business logic layer
- Separate storage for documents (binary files) and metadata (structured records)
- Event-driven notifications when key actions occur
- Monitoring, audit logging, and encryption throughout

---

## High-Level Architecture

```
External Supplier / Internal Reviewer / Administrator
            |
            v
        Route 53 (DNS)
            |
            v
        CloudFront (CDN + HTTPS + WAF)
            |
            v
        S3 — Frontend Bucket (Static Web App)
            |
            v
        Amazon Cognito (Authentication)
            |
            v
        API Gateway (HTTPS REST API)
            |
            v
          Lambda (Business Logic)
          /              \
         v                v
    DynamoDB          S3 — Document Bucket
    (Metadata)            |
                          v
                         SNS (Notifications)

        [CloudWatch — Logs, Metrics, Alarms]
        [CloudTrail — API Audit Logging]
        [KMS — Encryption Key Management]
        [IAM — Least-Privilege Access Control]
```

(![Solutions Architecture Diagram](diagrams/solutions-architecture-diagram.png))
---

## AWS Services — Selection and Rationale

| AWS Service | Role in the Solution | Why It Was Chosen |
|---|---|---|
| **Amazon Route 53** | DNS management for the application domain | Standard AWS DNS service; integrates naturally with CloudFront |
| **Amazon CloudFront** | Delivers the frontend securely and efficiently | Low latency, HTTPS enforcement, WAF integration, avoids direct S3 exposure |
| **AWS WAF** | Web application firewall protecting the CloudFront distribution | Provides basic protection against common web threats at the edge layer |
| **S3 — Frontend Bucket** | Hosts the static web application files | Low-cost, durable, and simple to operate for static frontend hosting |
| **Amazon Cognito** | Authentication and identity management for all user types | Avoids building a custom login system; supports secure managed identity flows |
| **Amazon API Gateway** | Exposes HTTPS API endpoints for the frontend | Fully managed API layer with Cognito authorizer support and Lambda integration |
| **AWS Lambda** | Runs backend business logic | No server management required; cost-efficient and event-driven |
| **S3 — Document Bucket** | Stores uploaded supplier documents | Durable, secure, scalable object storage; correct type for binary document files |
| **Amazon DynamoDB** | Stores document metadata and processing status | Fast, serverless, schema-flexible; well-suited for submission record and status tracking |
| **Amazon SNS** | Sends notifications on new submissions or status changes | Simple pub/sub mechanism that decouples events from notification delivery |
| **Amazon CloudWatch** | Collects logs, metrics, and alarms | Essential for operational monitoring and troubleshooting |
| **AWS CloudTrail** | Records AWS API activity for audit purposes | Supports governance and traceability requirements |
| **AWS KMS** | Manages encryption keys | Enables server-side encryption for S3 and DynamoDB at rest |
| **AWS IAM** | Controls access between users and services | Required for least-privilege security across all service interactions |

---

## User Roles

Three user types interact with the platform, each with a defined access scope:

| Role | Access Rights |
|---|---|
| **External Supplier** | Authenticate, upload documents, view own submission status |
| **Internal Reviewer** | Authenticate, view all submissions, inspect documents, update processing status |
| **Administrator** | Manage user access, oversee platform configuration and health |

Role separation is enforced through Amazon Cognito user groups and IAM policies. API Gateway endpoints validate the authenticated token and pass identity information to Lambda, which applies role-based logic before executing any action.

---

## End-to-End Data Flow

### Step 1 — User accesses the application

A supplier or internal employee navigates to the platform domain. Route 53 resolves the domain, and CloudFront delivers the static frontend from the S3 frontend bucket. All traffic is served over HTTPS. AWS WAF inspects requests at the edge.

### Step 2 — Authentication

The user signs in through Amazon Cognito. Cognito issues a JWT token upon successful authentication. The frontend stores this token and attaches it to all subsequent API requests. Suppliers and internal reviewers are placed into separate Cognito user groups.

### Step 3 — Frontend calls backend APIs

After authentication, the frontend sends HTTPS requests to API Gateway. The API Gateway Cognito authorizer validates the JWT token before allowing requests through. Typical API actions include:

- Requesting a pre-signed upload URL
- Listing submitted documents
- Viewing submission metadata
- Updating review status

### Step 4 — Lambda processes the request

Lambda functions handle the business logic behind each API request. Examples of Lambda responsibilities:

- Validate the authenticated user identity and group membership
- Check that the user has the correct permissions for the requested action
- Generate a pre-signed S3 URL for document upload
- Write or update submission metadata in DynamoDB
- Retrieve document lists or status information for internal reviewers

### Step 5 — Supplier uploads the document directly to S3

The frontend uses the pre-signed URL returned by Lambda to upload the document directly to the S3 document bucket. **The file does not pass through API Gateway or Lambda.** This is a deliberate architectural choice that keeps the backend lightweight and avoids hitting API Gateway payload size limits.

### Step 6 — Metadata is stored in DynamoDB

After the upload, Lambda writes a metadata record to DynamoDB. The record captures:

- Supplier ID and display name
- Document type
- S3 object key (the file path)
- Upload timestamp
- Initial processing status (e.g. `received`)

### Step 7 — Notification is triggered via SNS

An SNS notification is published when a new document is submitted. The notification can target an internal review email distribution list or an operations team queue. This decouples the upload event from the notification delivery mechanism, making it easier to extend in the future.

### Step 8 — Internal reviewer accesses the submission

Internal reviewers authenticate through the same frontend and retrieve submission records from the API. They can view the document (served via a time-limited pre-signed S3 read URL) and update the processing status. Supported status values are:

- `received`
- `under_review`
- `approved`
- `rejected`

### Step 9 — Monitoring and audit

CloudWatch captures Lambda execution logs, API Gateway metrics, and alarms for failures or anomalies. CloudTrail records all AWS API activity for governance and audit traceability.

---

## Design Decisions and Trade-offs

### Decision 1 — Serverless architecture

**Decision:** Use API Gateway, Lambda, S3, and DynamoDB rather than EC2-based components.

**Reasoning:** Managed services significantly reduce operational overhead. The goal was a solution that a mid-sized business could run without a dedicated DevOps team. Serverless services scale automatically and align well with variable, non-continuous workloads like document submission.

**Trade-off:** Distributed serverless architectures can be harder to debug locally than a monolithic application. Cold start latency on Lambda is acceptable for this use case but would need to be reviewed if stricter response-time SLAs were required.

---

### Decision 2 — Direct S3 uploads via pre-signed URLs

**Decision:** Suppliers upload files directly to S3 rather than routing through Lambda and API Gateway.

**Reasoning:** This prevents large files from passing through the backend layer, avoids API Gateway's payload size limits, and is a well-established cloud-native pattern for file uploads.

**Trade-off:** The upload flow requires an extra API call from the frontend to request the pre-signed URL before the upload begins. This adds a small amount of frontend complexity but is straightforward to implement.

---

### Decision 3 — DynamoDB over a relational database

**Decision:** Use DynamoDB for metadata storage instead of Amazon RDS.

**Reasoning:** The primary access pattern is simple key-value-style lookups by submission ID or supplier ID. DynamoDB is serverless, scales automatically, and requires no database instance management.

**Trade-off:** If the use case later requires complex relational queries, cross-table joins, or analytical reporting, a relational database would be more appropriate. For the current scope, DynamoDB is the better fit.

---

### Decision 4 — Single-region deployment

**Decision:** Deploy the first version in eu-central-1 (Frankfurt) only.

**Reasoning:** Single-region deployment keeps the architecture simple and directly addresses the data residency preferences of a Germany-based company. CloudFront provides global edge delivery for the frontend without requiring multi-region backend deployment.

**Trade-off:** The architecture does not provide full multi-region resilience. A regional outage would affect backend availability. For version 1, this is an accepted trade-off. Multi-region failover could be added in a future iteration.

---

### Decision 5 — Cognito for authentication

**Decision:** Use Amazon Cognito instead of a custom-built identity solution.

**Reasoning:** Authentication is a security-critical component. Cognito provides a managed, audited, and well-integrated identity service that avoids the risk of custom-built authentication vulnerabilities.

**Trade-off:** Cognito's customization options for advanced UX flows can be limited compared to fully custom solutions. For this use case, the default secure login flow is sufficient.

---

## Security Summary

Security controls are applied at every layer of the architecture:

- **Edge:** CloudFront enforces HTTPS; AWS WAF filters malicious web traffic
- **Identity:** Cognito manages authentication; IAM enforces least-privilege between services
- **API:** API Gateway validates JWT tokens via the Cognito authorizer before forwarding requests
- **Storage:** S3 buckets are private by default; documents are accessed only through pre-signed URLs; KMS provides encryption at rest
- **Database:** DynamoDB data is encrypted at rest via KMS
- **Audit:** CloudTrail logs all AWS API activity; CloudWatch captures application-level logs and metrics

---

## Out of Scope — Version 1

The following capabilities are not included in this initial design:

- Multi-region deployment or disaster recovery failover
- AI-assisted document analysis or OCR processing
- Integration with ERP or CRM systems
- SLA-based alerting or escalation workflows
- Infrastructure as Code (IaC) deployment scripts
