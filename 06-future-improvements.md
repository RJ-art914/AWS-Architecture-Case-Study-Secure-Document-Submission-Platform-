# 06 — Future Improvements & Trade-Offs

## Overview

This document reflects on the architectural decisions made during the design of the Secure Serverless Document Intake & Review Platform, acknowledges the trade-offs accepted within the current scope, and outlines a prioritised roadmap of enhancements that would strengthen the platform in a real-world production setting.

---

## Trade-Offs in the Current Design

Every architecture involves deliberate trade-offs. The following decisions were made consciously to meet the project's goals of simplicity, security, and cost-efficiency within a defined scope.

### 1. Serverless vs. Container-Based Compute
**Decision:** AWS Lambda over ECS/Fargate.
**Trade-off:** Lambda imposes a 15-minute execution limit and cold-start latency on infrequently invoked functions. For document processing that may involve large files or complex transformations, this could become a constraint.
**Accepted because:** The current workload (metadata writes, notification triggers, review status updates) is well within Lambda's execution model. The operational simplicity and zero server management outweigh the limitations at this scale.

### 2. DynamoDB On-Demand vs. Provisioned Capacity
**Decision:** On-Demand pricing mode.
**Trade-off:** On-Demand is more expensive per request than Provisioned at consistently high throughput. It also offers slightly less predictable latency under extreme burst conditions.
**Accepted because:** The submission workflow is irregular and low-volume. On-Demand eliminates the risk of under-provisioning and removes capacity planning overhead entirely.

### 3. SNS Direct Email vs. SES or SQS-Backed Queue
**Decision:** SNS for email notifications.
**Trade-off:** SNS does not guarantee exactly-once delivery and has limited retry configurability. For high-volume or mission-critical notifications, message loss is a risk.
**Accepted because:** Notification volume is low and non-critical. SNS is fully managed, requires zero configuration, and is sufficient for the current reviewer alert use case.

### 4. Single-Region Deployment (Frankfurt Only)
**Decision:** eu-central-1 as the sole deployment region.
**Trade-off:** No geographic redundancy. A regional AWS outage would result in platform unavailability.
**Accepted because:** GDPR data residency requirements favour a single EU region. Multi-region active-active adds significant complexity and cost that is not justified for a mid-sized manufacturer's internal tooling at this stage.

### 5. No CI/CD Pipeline in Current Scope
**Decision:** Infrastructure and deployment automation are out of scope.
**Trade-off:** Manual deployments introduce human error risk and slow down iteration cycles.
**Accepted because:** This project focuses on architecture design and documentation. A CI/CD pipeline is a natural next step for any production implementation.

### 6. REST API Gateway vs. HTTP API or GraphQL
**Decision:** REST API Gateway (v1).
**Trade-off:** HTTP API (v2) is cheaper and lower-latency for simple proxy use cases. REST API was chosen for its richer feature set (usage plans, request validation, API keys), but carries a slightly higher per-request cost.
**Accepted because:** The additional features of REST API Gateway — particularly request validation and WAF integration — are directly relevant to the security requirements of this platform.

---

## Prioritised Improvement Roadmap

The following enhancements are grouped by priority and effort level.

### Priority 1 — High Value, Low Effort

**S3 Lifecycle Policies**
Implement automated transitions for stored documents: Standard → S3-IA after 90 days → S3 Glacier after 365 days. This reduces long-term storage costs by 60–70% with minimal configuration effort and no impact on the application layer.

**CloudWatch Log Retention Policies**
Set explicit retention periods (30–90 days) on all Lambda and API Gateway log groups to prevent unbounded log storage costs accumulating silently over time.

**Lambda Reserved Concurrency**
Set reserved concurrency limits on each Lambda function to prevent one function from consuming the entire regional concurrency pool during an unexpected spike, protecting overall platform stability.

**Input Validation on API Gateway**
Add request body validation schemas directly in API Gateway to reject malformed requests before they reach Lambda, reducing unnecessary invocations and improving the security posture at the entry point.

---

### Priority 2 — High Value, Medium Effort

**CI/CD Pipeline with AWS CodePipeline or GitHub Actions**
Automate deployments using Infrastructure as Code (AWS SAM or Terraform) with a pipeline that runs tests, validates templates, and deploys to a staging environment before production. This is the single most impactful operational improvement available.

**SQS Buffer Between Upload Trigger and Processing Lambda**
Introduce Amazon SQS between the S3 event notification and the downstream processing Lambda function. This decouples ingestion from processing, provides built-in retry logic with a Dead Letter Queue (DLQ) for failed messages, and handles traffic bursts without dropping events.

**Document Versioning in S3**
Enable S3 Object Versioning on the document storage bucket to maintain a full history of uploaded files. This supports compliance use cases where audit trails of document changes are required.

**Structured Logging with Correlation IDs**
Introduce structured JSON logging in all Lambda functions with a shared correlation ID per request. This dramatically improves observability by allowing end-to-end request tracing across API Gateway, Lambda, and DynamoDB in CloudWatch Logs Insights.

---

### Priority 3 — Strategic Value, Higher Effort

**Infrastructure as Code (IaC) with AWS SAM or Terraform**
Define the entire architecture in code, enabling repeatable deployments, environment parity (dev/staging/prod), and version-controlled infrastructure changes. AWS SAM is the natural choice given the serverless-first design; Terraform offers broader multi-cloud portability for organisations with mixed environments.

**Multi-Region Active-Passive Disaster Recovery**
Replicate S3 buckets (Cross-Region Replication) and DynamoDB tables (Global Tables) to a second EU region (eu-west-1 / Ireland) for disaster recovery. Route 53 health checks and failover routing policies would redirect traffic automatically during a regional outage.

**Corporate Identity Provider Integration (Azure AD / SAML)**
Extend Amazon Cognito with a federated identity provider via SAML 2.0 or OIDC to allow internal reviewers and administrators to authenticate using their existing corporate credentials. This eliminates the need to manage a separate user pool for internal staff.

**Amazon EventBridge for Workflow Orchestration**
Replace direct Lambda-to-Lambda invocations or S3 event triggers with Amazon EventBridge rules for a more loosely coupled, extensible event-driven architecture. New consumers (e.g., an audit service, a reporting service) can subscribe to events without modifying existing functions.

---

### Priority 4 — Future Vision

**AI-Assisted Document Classification (Amazon Textract / Comprehend)**
Integrate Amazon Textract to automatically extract text and key data fields from uploaded documents (e.g., invoices, certificates, compliance forms). Amazon Comprehend could classify document type and flag incomplete submissions before they reach a human reviewer — significantly reducing manual review effort at scale.

**Self-Service Supplier Portal**
Build a richer frontend experience with submission history, document status tracking, and re-upload capabilities, reducing the administrative burden on internal staff for routine supplier queries.

**Analytics and Reporting Layer (Amazon Athena + QuickSight)**
Export DynamoDB metadata to S3 via DynamoDB Streams and Lambda, then query it with Amazon Athena and visualise trends in Amazon QuickSight — covering submission volumes, review turnaround times, and supplier compliance rates.

**Automated Compliance Reporting**
Generate periodic GDPR compliance reports from CloudTrail and CloudWatch data, summarising access events, data processing activities, and security incidents for the Data Protection Officer (DPO).

---

## Lessons Learned

**Security is cheapest when designed in from the start.**
Retrofitting WAF rules, KMS encryption, or IAM least-privilege policies onto an existing architecture is significantly more complex and error-prone than building them in at the design phase. Every security control in this architecture was intentional, not remedial.

**Serverless simplicity has a learning curve.**
The absence of servers does not mean the absence of complexity — it shifts complexity from infrastructure management to IAM policies, event source mappings, cold start behaviour, and distributed tracing. Understanding these nuances is key to reliable serverless operations.

**Documentation is a first-class deliverable.**
Architecture decisions made without written rationale are difficult to defend, audit, or hand over. Capturing trade-offs and design intent in structured documents is as important as the architecture itself — especially in regulated industries like manufacturing.

**Cost awareness should be continuous.**
Serverless cost models are transparent but can surprise teams unfamiliar with per-request pricing at scale. Enabling AWS Cost Anomaly Detection and setting billing alerts from day one ensures unexpected cost spikes are caught early.

**Scope discipline protects quality.**
Clearly defining what is out of scope (multi-region, AI features, CI/CD) allowed the core architecture to be designed thoroughly rather than superficially. A well-documented MVP with clear extension points is more valuable than an over-engineered design that lacks depth.

---

## Summary

The current architecture is deliberately scoped to deliver a secure, functional, and cost-efficient foundation. The trade-offs made — single-region deployment, SNS for notifications, no CI/CD pipeline — are appropriate for the project's phase and constraints. The roadmap above provides a clear, prioritised path from this foundation to a production-hardened, enterprise-grade platform, with each improvement building naturally on what already exists. Together, these six documentation files represent a complete picture of the platform: its purpose, its design, its security posture, its cost profile, and its future potential.
