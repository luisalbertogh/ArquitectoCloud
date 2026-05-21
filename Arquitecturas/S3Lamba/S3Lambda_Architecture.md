# S3Lambda — Architecture Plan

## Executive Summary

This document describes the architecture for a **Serverless File Processing Application** on AWS. The solution consists of two private S3 buckets, a Python Lambda function with access to both buckets, and an event-driven trigger that automatically invokes the function whenever a new file is uploaded to the source bucket.

The architecture follows an **event-driven, serverless** pattern that requires zero server management, scales automatically to workload demand, and keeps operational costs proportional to actual usage.

---

## System Context

![System Context Diagram](diagrams/01_system_context.svg)

### Overview

The System Context diagram places the application within its broader ecosystem. It identifies the one external actor (Operator/Developer) and the two supporting AWS services that cross the system boundary (IAM and CloudWatch).

### Key Components

| Element | Role |
|---|---|
| **Developer / Operator** | Uploads files to the Source S3 Bucket via the AWS Console, CLI, or SDK |
| **Serverless File Processing Application** | The system under consideration — contains S3 buckets, Lambda, IAM, and CloudWatch |
| **AWS IAM** | External identity and access management service that provides the Lambda execution role and S3 access policies |
| **Amazon CloudWatch** | External observability platform that receives logs, metrics, and alarms from the application |

### Design Decisions

- The system boundary is deliberately drawn to **exclude** IAM and CloudWatch because they are shared AWS platform services, not owned by the application.
- Only one external human actor is modelled. Automated uploaders (ETL jobs, other systems) interact the same way and can be considered instances of the same actor role.

---

## Architecture Overview

The solution is built entirely on **AWS managed services** using an event-driven architecture:

```
File Upload → S3 Source Bucket → S3 Event Notification → Lambda Function → S3 Destination Bucket
                                                                  ↓
                                                          CloudWatch Logs
```

Key architectural patterns applied:

| Pattern | Implementation |
|---|---|
| **Event-Driven Architecture** | S3 `s3:ObjectCreated:*` notification triggers Lambda asynchronously |
| **Principle of Least Privilege** | IAM policy grants only `GetObject`, `PutObject`, `ListBucket` on the specific buckets |
| **Private by Default** | Both S3 buckets block all public access; SSE-S3 encryption at rest |
| **Serverless Compute** | Lambda eliminates server management and scales to zero when idle |
| **Decoupled Storage & Compute** | S3 and Lambda are loosely coupled through an event notification |

---

## Component Architecture

![Component Diagram](diagrams/02_component.svg)

### Overview

The Component diagram shows all logical AWS components, their grouping by layer, and their dependencies.

### Key Components

| Component | Responsibility |
|---|---|
| **Source Bucket** (`s3-source-*`) | Persistent storage for incoming raw files. Hosts the S3 Event Notification configuration |
| **S3 Event Notification** | Decouples the storage event from compute. Delivers a JSON event payload to Lambda on every `s3:ObjectCreated:*` event |
| **Lambda Function** (Python 3.12) | Reads the uploaded file from the source bucket, applies business logic, and writes the result to the destination bucket |
| **Destination Bucket** (`s3-destination-*`) | Persistent storage for processed output. No event notification — it is a terminal data sink |
| **IAM Execution Role** | Service principal assumed by Lambda (`lambda.amazonaws.com`) |
| **IAM Policy** | Least-privilege policy: `s3:GetObject` on source, `s3:PutObject` on destination, `s3:ListBucket` on both, and CloudWatch Logs permissions |
| **CloudWatch Logs** | Centralised log group `/aws/lambda/file-processor` for execution traces, errors, and duration metrics |

### Relationships

- The **Source Bucket** publishes an event to the **S3 Event Notification** on every `PutObject`.
- The **S3 Event Notification** asynchronously invokes the **Lambda Function**, passing the bucket name and object key.
- The **Lambda Function** calls `GetObject` on the source bucket to retrieve the file, processes it, then calls `PutObject` on the destination bucket.
- The **IAM Policy** is attached to the **IAM Execution Role**, which is assumed by the Lambda service.

### NFR Considerations

- **Scalability**: Lambda scales automatically per invocation. Each file upload triggers an independent Lambda execution.
- **Security**: IAM policy uses resource-level ARN constraints (`arn:aws:s3:::bucket-name/*`) to restrict access to specific buckets only.
- **Reliability**: S3 provides 11 nines of durability. Lambda retries on service errors automatically (2 async retries by default).
- **Maintainability**: Clear separation of concerns — storage, event routing, compute, and security are independent layers.

---

## Deployment Architecture

![Deployment Diagram](diagrams/03_deployment.svg)

### Overview

The Deployment diagram shows the actual AWS resource configuration within a single AWS Region, including runtime settings, bucket configurations, and IAM resource names.

### Key Components

| Resource | Configuration |
|---|---|
| **s3-source-`<account-id>`** | Versioning enabled, SSE-S3 encryption, Block Public Access, `ObjectCreated` event → Lambda |
| **s3-destination-`<account-id>`** | Versioning enabled, SSE-S3 encryption, Block Public Access |
| **Lambda: file-processor** | Runtime Python 3.12, Memory 512 MB, Timeout 5 min, Reserved concurrency 10 |
| **IAM Role: lambda-s3-execution-role** | Trust policy: `lambda.amazonaws.com`, inline policy attached |
| **IAM Policy: lambda-s3-access-policy** | `s3:GetObject` (source), `s3:PutObject` (destination), `s3:ListBucket` (both), `logs:*` |
| **CloudWatch Log Group** | `/aws/lambda/file-processor`, retention 30 days, metric filters + alarms |

### Deployment Strategy

- All resources are deployed in a **single AWS Region** to minimise latency and data transfer costs.
- Bucket names include the `<account-id>` suffix to guarantee global uniqueness and prevent bucket squatting.
- **Reserved concurrency** on Lambda (set to 10) prevents a burst of uploads from consuming the entire regional Lambda concurrency quota.

### NFR Considerations

- **Performance**: 512 MB Lambda memory provides a good balance of CPU and memory. For large files, this can be increased up to 10 GB.
- **Security**: Bucket names include the account ID to prevent predictable names. Both buckets use SSE-S3 (can be upgraded to SSE-KMS for stricter key management).
- **Reliability**: S3 versioning prevents accidental overwrite data loss. CloudWatch alarms on Lambda error rate enable rapid incident response.
- **Cost Efficiency**: Lambda pricing is pay-per-invocation and per-GB-second. For typical file processing workloads this is far cheaper than a persistent EC2 or ECS tier.

---

## Data Flow

![Data Flow Diagram](diagrams/04_data_flow.svg)

### Overview

The Data Flow diagram illustrates how data moves through the system from the moment a file is uploaded until the processed result is stored and the execution is logged.

### Flow Description

| Step | Data | Description |
|---|---|---|
| 1 | Binary object | Operator uploads any file type via `PutObject` |
| 2 | ObjectCreated event | S3 emits an event with bucket name, object key, file size, and eTag |
| 3 | JSON event payload | Lambda receives the event; extracts `bucket` and `key` from `event["Records"][0]["s3"]` |
| 4 | GetObject request | Lambda calls the S3 API to stream the raw file bytes |
| 5 | File bytes stream | S3 returns the file content |
| 6 | Transformed object | Lambda processes/transforms the file according to business logic |
| 7 | PutObject call | Lambda writes the result to the destination bucket |
| 8 | Execution metadata | Lambda emits start time, duration, requestId, and any errors to CloudWatch |

### Data Handling Principles

- **No data persistence in Lambda**: Lambda is stateless; it reads from S3, transforms in memory, and writes back to S3.
- **Data at rest**: All objects are encrypted with SSE-S3 in both buckets.
- **Data in transit**: All S3 API calls use HTTPS (TLS 1.2+) enforced by AWS endpoint policies.
- **Idempotency**: Each S3 event carries a unique `requestId`. The Lambda handler should be designed to be idempotent (re-processing the same object produces the same output).

### NFR Considerations

- **Performance**: Streaming `GetObject` avoids loading the entire file into Lambda memory at once, supporting large files.
- **Security**: Data never leaves AWS; all transfers are within the same region over private AWS network paths.
- **Reliability**: S3 event delivery has at-least-once semantics; the Lambda handler must be idempotent.

---

## Key Workflows

### File Processing Sequence

![Sequence Diagram](diagrams/05_sequence.svg)

### Overview

The Sequence diagram shows the complete interaction flow for the primary use case: a developer uploads a file, the event chain fires, Lambda processes the file, and the result is stored.

### Flow of Operations

| Step | Actor → Actor | Description |
|---|---|---|
| 1 | Operator → Source Bucket | `PutObject` — file uploaded |
| 2 | Source Bucket → S3 Event | `ObjectCreated` event published |
| 3 | S3 Event → Lambda | Lambda invoked asynchronously with JSON payload |
| 4 | Lambda → CloudWatch | Log entry: processing started, with `requestId` |
| 5 | Lambda → Source Bucket | `GetObject` call to read the uploaded file |
| 6 | Source Bucket → Lambda | File bytes returned as a stream |
| 7 | Lambda → Lambda | Business logic / transformation applied in memory |
| 8 | Lambda → Destination Bucket | `PutObject` — processed file written |
| 9 | Destination Bucket → Lambda | Success response with `ETag` |
| 10 | Lambda → CloudWatch | Log entry: processing complete with duration and output key |

### Design Decisions

- Lambda is invoked **asynchronously** by S3. The operator does not wait for the processing result inline with the upload.
- Lambda writes to CloudWatch both at the **start** (step 4) and **end** (step 10) of execution to enable latency measurement and error correlation.
- The ETag returned by the destination bucket (step 9) can be stored for downstream audit purposes.

---

## Non-Functional Requirements Analysis

### Scalability

- **Horizontal scaling**: Lambda scales to handle multiple concurrent file uploads automatically. Each S3 `ObjectCreated` event maps to an independent Lambda invocation.
- **Reserved concurrency**: Set to 10 by default to control blast radius. Can be increased to match expected peak throughput.
- **S3 scalability**: Amazon S3 is designed to scale to any request rate with automatic partitioning.
- **Throughput limit**: At the default Lambda reserved concurrency of 10, the system can process 10 files concurrently. Increase this value to match actual workload.

### Performance

- **Cold start**: Python Lambda cold starts are typically 100–400 ms. For latency-sensitive workloads, enable **Provisioned Concurrency** on the Lambda function.
- **Memory and CPU**: 512 MB Lambda allocation provides sufficient CPU for most file transformations. Increase memory to improve CPU-bound performance (Lambda allocates CPU proportionally to memory).
- **Large files**: Use S3 streaming (`get_object` with `Body.read()` chunked) to avoid Lambda memory exhaustion for files larger than ~100 MB.
- **S3 Transfer Acceleration**: Can be enabled on either bucket if uploads originate from geographically distant locations.

### Security

- **Least privilege IAM**: The execution role grants `s3:GetObject` only on the source bucket and `s3:PutObject` only on the destination bucket. Cross-bucket write from destination to source is explicitly not granted.
- **Private buckets**: Both buckets have `BlockPublicAcls`, `IgnorePublicAcls`, `BlockPublicPolicy`, and `RestrictPublicBuckets` set to `true`.
- **Encryption at rest**: SSE-S3 on both buckets. For regulated workloads, upgrade to **SSE-KMS** with a customer-managed key (CMK) and key rotation.
- **Encryption in transit**: All AWS SDK calls use HTTPS. Add a bucket policy condition `"aws:SecureTransport": "true"` to deny unencrypted connections.
- **No public Lambda URL**: Lambda is only invoked via S3 event notification — there is no HTTP endpoint exposed.
- **VPC isolation** (optional enhancement): Lambda can be deployed inside a VPC with S3 Gateway Endpoints to ensure traffic never traverses the public internet.

### Reliability

- **S3 durability**: 99.999999999% (11 nines) for S3 Standard objects.
- **Lambda retries**: Asynchronous Lambda invocations are retried twice on service errors by AWS before the event is discarded.
- **Dead-letter queue (DLQ)**: Configure an SQS DLQ on the Lambda function to capture failed events for re-processing or investigation.
- **Bucket versioning**: Enabled on both buckets to protect against accidental overwrites and enable point-in-time recovery.
- **CloudWatch Alarms**: Alert on `Errors > 0` and `Duration > 80% of timeout` on the Lambda function for proactive incident management.

### Maintainability

- **Serverless operations**: No patching, scaling, or capacity planning required for the compute layer.
- **Infrastructure as Code**: All resources are defined in Terraform (this repository) for reproducible deployments.
- **Structured logging**: Lambda handler should emit structured JSON logs to CloudWatch for easy querying with CloudWatch Logs Insights.
- **Log retention**: 30-day retention policy on the CloudWatch Log Group balances debugging capability with cost.
- **Environment variables**: Lambda configuration (bucket names, processing options) injected via environment variables for easy updates without code changes.

### Cost Efficiency

- Lambda pricing: **$0.0000166667 per GB-second** + **$0.20 per 1M invocations**.
- S3 Standard pricing: **$0.023 per GB/month** stored + **$0.005 per 1,000 PUT requests** + **$0.0004 per 1,000 GET requests**.
- CloudWatch Logs: **$0.50 per GB ingested** + **$0.03 per GB stored**.

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Lambda timeout on large files | Medium | Medium | Increase timeout up to 15 min; use S3 streaming; for very large files consider ECS Fargate |
| S3 event delivery failure (at-least-once) | Low | Medium | Design Lambda handler to be idempotent; use DLQ to capture unprocessable events |
| Lambda cold start latency | Low | Low | Enable Provisioned Concurrency if consistent sub-second latency is required |
| Accidental public bucket exposure | Low | High | Enforce SCPs at the AWS Organizations level to deny `s3:PutBucketAcl` with public grants |
| Cost runaway on large file burst | Low | Medium | Set reserved concurrency + configure S3 event filter (e.g., only trigger on `.csv` suffix) |
| IAM privilege escalation | Very Low | High | Use resource-level ARN constraints in the IAM policy; enable AWS CloudTrail for audit |

---

## Technology Stack Recommendations

| Layer | Recommended Service | Rationale |
|---|---|---|
| Object Storage | **Amazon S3 Standard** | 11-nines durability, native event notifications, pay-per-use |
| Compute | **AWS Lambda (Python 3.12)** | Zero server management, per-invocation billing, native S3 integration |
| Event Routing | **S3 Event Notifications** | Native, zero-cost, sub-second delivery to Lambda |
| Identity & Access | **AWS IAM** | Native AWS service, fine-grained resource-level permissions |
| Observability | **Amazon CloudWatch** | Native Lambda/S3 integration, metric filters, alarms, Logs Insights |
| Infrastructure as Code | **Terraform / OpenTofu** | Already adopted in this repository |
| Runtime | **Python 3.12** | Actively supported, rich AWS SDK (`boto3`) ecosystem |

---

## Cost Estimate

Based on a sample workload of **10,000 file uploads/month**, average file size **5 MB**, average Lambda execution duration **2 seconds** at **512 MB** memory:

| Service | Usage | Estimated Monthly Cost (USD) |
|---|---|---|
| **Lambda invocations** | 10,000 requests | ~$0.00 (within free tier: 1M req/month) |
| **Lambda compute** | 10,000 × 2s × 0.5GB = 10,000 GB-s | ~$0.17 |
| **S3 Source storage** | 50 GB (10k × 5MB) | ~$1.15 |
| **S3 Destination storage** | 50 GB | ~$1.15 |
| **S3 PUT requests** | 10,000 | ~$0.05 |
| **S3 GET requests** | 10,000 | ~$0.004 |
| **CloudWatch Logs ingestion** | ~0.5 GB | ~$0.25 |
| **CloudWatch Logs storage** | ~0.5 GB × 30 days | ~$0.02 |
| **Total** | | **~$2.80/month** |

> Costs calculated using [AWS Pricing Calculator](https://calculator.aws/) for the `eu-west-1` region as of May 2026. Actual costs depend on file sizes, processing duration, and data transfer.

---

## Next Steps

1. **Define business logic**: Implement the specific file transformation logic inside the Lambda Python handler.
2. **Create Terraform resources**: Define `aws_s3_bucket`, `aws_lambda_function`, `aws_iam_role`, `aws_s3_bucket_notification` in this repository.
3. **Configure DLQ**: Add an SQS Dead-Letter Queue to the Lambda function for fault tolerance.
4. **Add S3 event prefix/suffix filter**: Restrict the event notification to specific file types (e.g., `*.csv`, `*.json`) to avoid triggering on unintended objects.
5. **Enable CloudWatch alarms**: Create alarms for `Errors > 0`, `Throttles > 0`, and `Duration > 240s` on the Lambda function.
6. **Add bucket lifecycle policies**: Automatically transition or expire objects in the source bucket after a defined retention period.
7. **Security review**: Add bucket policies enforcing `aws:SecureTransport = true` on both buckets.
8. **Load test**: Simulate peak upload load to validate reserved concurrency settings.

---

## References

- [AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)
- [Using AWS Lambda with Amazon S3](https://docs.aws.amazon.com/lambda/latest/dg/with-s3.html)
- [S3 Event Notifications](https://docs.aws.amazon.com/AmazonS3/latest/userguide/NotificationHowTo.html)
- [IAM Best Practices — Least Privilege](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#grant-least-privilege)
- [S3 Security Best Practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html)
- [Lambda Async Invocation and Retries](https://docs.aws.amazon.com/lambda/latest/dg/invocation-async.html)
- [Lambda Reserved Concurrency](https://docs.aws.amazon.com/lambda/latest/dg/configuration-concurrency.html)
- [AWS Pricing Calculator](https://calculator.aws/)
- [Serverless Event-Driven Architecture — AWS Well-Architected](https://docs.aws.amazon.com/wellarchitected/latest/serverless-applications-lens/welcome.html)
