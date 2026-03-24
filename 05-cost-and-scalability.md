# 05 — Cost and Scalability

## Overview

One of the core motivations for choosing a serverless architecture was the alignment between its pricing model and the usage profile of this platform. Document submission is not a continuous, high-throughput workload — it is event-driven, bursty, and low-volume during off-peak periods. Serverless AWS services charge based on actual consumption, making this architecture inherently cost-efficient for the target scenario.

---

## Cost Model: Pay-Per-Use by Design

Every major service in this architecture is billed on a consumption basis, with no minimum provisioned capacity required.

### Amazon API Gateway
- Billed per API call (REST API: ~USD 3.50 per million requests in eu-central-1)
- No idle costs when no requests are made
- Low-volume document submission workflows result in minimal monthly charges

### AWS Lambda
- Billed per invocation and per GB-second of execution time
- Free tier includes 1 million invocations/month and 400,000 GB-seconds/month
- For this use case (document metadata writes, review triggers, notification dispatch), Lambda costs remain negligible at low-to-medium scale

### Amazon S3
- Storage: ~USD 0.023 per GB/month (Standard tier, Frankfurt / eu-central-1)
- PUT/GET requests billed per 1,000 operations
- Pre-signed URL uploads go directly from client to S3, reducing Lambda invocation cost
- Lifecycle policies can transition older documents to S3-IA or Glacier to reduce long-term storage costs

### Amazon DynamoDB
- On-Demand capacity mode used — billed per read/write request unit
- No provisioned throughput required; cost scales linearly with actual usage
- Well-suited for irregular document submission patterns with unpredictable traffic spikes

### Amazon Cognito
- Free tier: 50,000 Monthly Active Users (MAUs)
- Sufficient for the supplier + reviewer + admin user base of a mid-sized manufacturer
- No additional cost for token issuance, validation, or session management within the free tier

### Amazon SNS
- First 1 million publishes per month are free
- Email notifications via SNS are extremely low cost at the expected volume
- Future SMS notifications, if added, are billed per message sent

### Amazon CloudFront
- Free tier: 1 TB of data transfer out/month and 10 million HTTP requests
- Suitable for serving the static React frontend at near-zero cost during early phases
- Reduces S3 origin request costs by caching static assets at edge locations

### AWS WAF
- Approximately USD 5.00/month per Web ACL + USD 1.00 per million requests inspected
- A predictable, flat cost that provides significant security value relative to the investment

### AWS KMS
- USD 1.00/month per Customer Managed Key (CMK)
- USD 0.03 per 10,000 API calls (encrypt/decrypt operations)
- Minimal overhead cost given the strong encryption guarantees it provides

### Amazon CloudWatch + CloudTrail
- CloudTrail: First management event trail per region is free
- CloudWatch Logs: ~USD 0.57 per GB ingested (Frankfurt); log retention policies reduce storage cost
- CloudWatch Alarms: USD 0.10 per alarm/month for standard resolution alarms

---

## Estimated Monthly Cost (Low-Volume Baseline)

The following estimate reflects a small manufacturing company with ~50 active users, ~500 document uploads/month, and standard review workflows.

| Service         | Estimated Monthly Cost (USD) |
|-----------------|------------------------------|
| API Gateway     | < USD 1.00                   |
| Lambda          | Free tier (< USD 1.00)       |
| S3 Storage      | < USD 1.00                   |
| DynamoDB        | < USD 1.00                   |
| Cognito         | Free tier                    |
| SNS             | Free tier                    |
| CloudFront      | Free tier                    |
| WAF             | ~USD 5.00                    |
| KMS             | ~USD 1.00                    |
| CloudWatch      | ~USD 1.00–2.00               |
| CloudTrail      | Free (first trail)           |
| **Total**       | **~USD 10–15/month**         |

> **Note:** These are indicative estimates based on AWS public pricing for eu-central-1 (Frankfurt) as of early 2025. Actual costs depend on usage patterns, data volume, and retention settings. Use the [AWS Pricing Calculator](https://calculator.aws) for precise estimates.

---

## Scalability: How the Architecture Grows with Demand

The serverless design means most scaling is handled automatically by AWS without manual intervention or infrastructure changes.

### Automatic Horizontal Scaling
- **Lambda** scales concurrently per invocation — each function instance handles one request. AWS automatically provisions additional instances as traffic grows, up to the regional concurrency limit (default: 1,000 concurrent executions, adjustable on request).
- **API Gateway** handles tens of thousands of requests per second without configuration changes.
- **DynamoDB On-Demand** automatically adjusts read/write throughput to meet demand, with no capacity planning required.
- **S3** provides virtually unlimited storage and scales to handle any number of concurrent uploads or downloads.

### CloudFront as a Scalability Multiplier
- Static assets (frontend HTML, JS, CSS) are cached globally at CloudFront edge locations, reducing origin load and latency regardless of user location within Europe.
- This decouples frontend scalability from backend API capacity entirely.

### Cognito at Scale
- Amazon Cognito supports hundreds of thousands of MAUs without infrastructure changes.
- Authentication token verification is offloaded to API Gateway authorizers, keeping Lambda functions free from auth overhead.

---

## Scalability Limits and Considerations

While the architecture scales well automatically, some boundaries should be planned for as usage grows.

| Constraint | Default Limit | Mitigation |
|---|---|---|
| Lambda concurrent executions | 1,000 per region | Request limit increase via AWS Support |
| API Gateway throttling | 10,000 req/s (burst: 5,000) | Configure usage plans; request increase if needed |
| DynamoDB item size | 400 KB per item | Store document content in S3; store only metadata in DynamoDB |
| S3 PUT request rate | 3,500 PUT/s per prefix | Use randomised key prefixes if high-concurrency uploads are anticipated |
| SNS delivery retries | Limited retry window | Use SQS as a buffer for guaranteed delivery in high-volume scenarios |

---

## Cost Optimisation Strategies

Several design decisions already reduce cost, and further optimisations are available as the platform matures.

**Already implemented:**
- Pre-signed URLs for direct client-to-S3 uploads avoid routing large file payloads through Lambda, saving both invocation time and data transfer costs.
- DynamoDB On-Demand mode eliminates over-provisioning costs during quiet periods.
- CloudFront caching reduces repeated S3 GET requests and data transfer fees.

**Recommended for future implementation:**
- **S3 Lifecycle Policies:** Automatically transition documents older than 90 days to S3-IA (Infrequent Access) and archive after 1 year to S3 Glacier, reducing long-term storage costs by up to 60–70%.
- **Lambda Power Tuning:** Use the AWS Lambda Power Tuning tool to find the optimal memory configuration that balances speed and cost per function.
- **CloudWatch Log Retention:** Set log group retention to 30–90 days to prevent unbounded log storage costs.
- **Reserved Concurrency:** For predictable workloads, Lambda Provisioned Concurrency can reduce cold-start latency without significantly increasing cost.
- **AWS Cost Anomaly Detection:** Enable Cost Anomaly Detection in AWS Cost Explorer to receive alerts when spending deviates unexpectedly from baseline.

---

## Scalability Roadmap

As the platform grows beyond its initial scope, the following enhancements would extend its capacity and capabilities without requiring a re-architecture.

1. **Multi-region active-passive failover** — Replicate S3 buckets and DynamoDB tables to a second AWS region (e.g., eu-west-1 / Ireland) for disaster recovery and improved resilience.
2. **SQS-backed async processing** — Introduce Amazon SQS between the upload trigger and downstream Lambda functions to decouple ingestion from processing and handle traffic bursts gracefully.
3. **Document volume growth** — S3 and DynamoDB are both designed for petabyte-scale workloads. No architectural changes are needed to support 10x or 100x the current document volume.
4. **Expanded user base** — Cognito supports enterprise-scale user pools. Integration with a corporate identity provider (e.g., Azure AD via SAML/OIDC) can be added without changing the core architecture.
5. **Analytics layer** — Amazon Athena can query S3-based document metadata or CloudTrail logs directly for operational reporting and compliance auditing at scale.

---

## Summary

The serverless architecture is fundamentally aligned with both cost efficiency and elastic scalability. At low volume, the platform runs for approximately **USD 10–15/month**, with the majority of that cost attributable to WAF and KMS — fixed security investments rather than usage-driven expenses. As submission volume grows, costs scale linearly and predictably, with no sudden capacity cliffs or manual scaling operations required. The combination of pre-signed URL uploads, DynamoDB On-Demand, and CloudFront caching ensures the platform remains lean and performant at every stage of growth.
