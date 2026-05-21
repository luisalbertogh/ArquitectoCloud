# Enterprise Web Application — AWS Architecture Plan

## Executive Summary

This document describes the reference architecture for a **multi-tier, highly available enterprise web application** deployed on AWS. The front tier is served by **Apache Tomcat / JBoss** application servers running on EC2 instances behind an **Application Load Balancer (ALB)**. The back tier uses **Amazon Aurora Serverless v2** as the relational database engine. The entire solution is designed for **Multi-AZ** deployment within a single AWS region, providing fault tolerance, automatic scaling, and defense-in-depth security.

Key architectural principles applied:

- **AWS Well-Architected Framework** — Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization, Sustainability pillars.
- **Least-privilege security** — Dedicated Security Groups per tier with strictly scoped rules; no cross-tier access except what is explicitly needed.
- **Zero single points of failure** — Redundant EC2 instances across two Availability Zones; Aurora Multi-AZ storage with a reader replica for automatic failover.
- **Immutable credentials** — DB credentials stored and rotated automatically by AWS Secrets Manager; never hard-coded in application configuration.

---

## System Context

> Overview of the system and its external actors.

![System Context Diagram](diagrams/01_system_context.svg)

### Key Components

| Actor / System | Description |
|---|---|
| **End Users** | Business users and customers accessing the application from a web browser over HTTPS. |
| **Enterprise Web Application** | The full AWS-hosted system: WAF, ALB, EC2 app servers, Aurora DB, and supporting services. |
| **DevOps / Ops Team** | Engineers who deploy, configure, and operate the application through CI/CD pipelines and AWS Console/CLI. |
| **Amazon CloudWatch** | Regional observability platform that collects metrics, logs, and fires alarms when thresholds are exceeded. |
| **AWS IAM** | Identity and access management service that vends execution roles and policies to every component. |

### Design Decisions

- **HTTPS everywhere**: all external traffic uses TLS 1.2+; HTTP is redirected to HTTPS at the ALB layer.
- **No direct DB access from the internet**: the database tier is isolated in private subnets with no route to the internet.
- **Centralized observability**: CloudWatch is the single pane of glass for all logs, metrics, and alarms.

---

## Architecture Overview

The application follows a **classic three-tier architecture** adapted for AWS-native services:

```
Internet → WAF → ALB (public subnets) → EC2/Tomcat (private app subnets) → Aurora Serverless v2 (private DB subnets)
```

| Tier | AWS Service | Subnet type |
|---|---|---|
| Edge / Security | AWS WAF + ACM | N/A (regional managed service) |
| Load Balancing | Application Load Balancer | Public (Multi-AZ) |
| Application | EC2 Auto Scaling (Tomcat/JBoss) | Private App (Multi-AZ) |
| Database | Aurora Serverless v2 | Private DB (Multi-AZ) |

---

## Component Architecture

> Internal components, their responsibilities, and interactions.

![Component Diagram](diagrams/02_component.svg)

### Key Components

| Component | Responsibility |
|---|---|
| **AWS WAF** | Evaluates every inbound HTTPS request against AWS Managed Rules (OWASP Top 10, IP reputation, rate-limiting). Blocks threats before they reach the ALB. |
| **AWS Certificate Manager (ACM)** | Issues and auto-renews the public TLS certificate bound to the ALB HTTPS listener. Zero operational overhead for certificate management. |
| **SG-ALB** | Security Group for the ALB. Allows inbound TCP 443 and 80 from `0.0.0.0/0`. Allows outbound to SG-WEB on port 8080 only. |
| **Application Load Balancer** | Layer-7 load balancer that performs TLS termination, HTTP-to-HTTPS redirect, content-based routing, and distributes traffic across healthy EC2 targets using round-robin. |
| **Auto Scaling Group** | Maintains a minimum of 2 EC2 instances (one per AZ). Scales out on CPU/memory thresholds; scales in after cooldown. Ensures at least one instance per AZ at all times. |
| **EC2 (Tomcat/JBoss)** | Runs the Java EE / Jakarta EE application. Fetches DB credentials from Secrets Manager. Emits structured logs to CloudWatch Logs. Uses an IAM instance profile for all AWS API calls. |
| **SG-WEB** | Security Group for EC2. Allows inbound TCP 8080 from SG-ALB only. Allows outbound TCP 3306/5432 to SG-DB and TCP 443 outbound via NAT Gateway (AWS API calls). |
| **Aurora Serverless v2 (Writer)** | Primary database node in AZ-1. Handles all INSERT/UPDATE/DELETE and consistent reads. Scales ACUs automatically between 0.5 and 32 based on load. |
| **Aurora Serverless v2 (Reader)** | Reader replica in AZ-2. Handles SELECT-only queries for load distribution. Promotes to writer automatically in case of primary failure (failover < 30 s). |
| **SG-DB** | Security Group for Aurora. Allows inbound TCP 3306 (MySQL) or 5432 (PostgreSQL) from SG-WEB only. No outbound rules required. |
| **AWS Secrets Manager** | Stores DB credentials (host, port, username, password). Enforces automatic rotation every 30 days using a Lambda function. EC2 caches credentials for 5 minutes to reduce API calls. |
| **Amazon CloudWatch** | Aggregates ALB access logs, EC2 application/system logs, and Aurora performance metrics. Alarms notify the Ops team via SNS on CPU > 80%, error rate > 1%, DB connections exhausted. |
| **IAM (Instance Profile)** | Grants EC2 instances least-privilege access to CloudWatch Logs (`PutLogEvents`), Secrets Manager (`GetSecretValue`), and SSM Parameter Store if needed. |
| **Amazon S3 (ALB Logs)** | Receives ALB access logs for long-term retention, compliance audit, and offline analysis with Athena. |

### Security Group Rules Summary

| Security Group | Inbound | Outbound |
|---|---|---|
| **SG-ALB** | TCP 443 from `0.0.0.0/0`; TCP 80 from `0.0.0.0/0` | TCP 8080 to SG-WEB |
| **SG-WEB** | TCP 8080 from SG-ALB | TCP 3306/5432 to SG-DB; TCP 443 to `0.0.0.0/0` (via NAT) |
| **SG-DB** | TCP 3306 or 5432 from SG-WEB | None |

### NFR Considerations

- **Scalability**: Auto Scaling Group grows from 2 to 6 EC2 instances; Aurora Serverless v2 scales independently from 0.5 to 32 ACUs without downtime.
- **Security**: Each tier has its own Security Group; the DB tier is only reachable from the app tier. WAF blocks OWASP Top 10 attacks at the edge.
- **Reliability**: Minimum 2 EC2 instances across 2 AZs; Aurora has a reader replica for automatic failover.
- **Maintainability**: ACM manages TLS renewal; Secrets Manager rotates DB credentials; CloudWatch provides centralized observability.

---

## Deployment Architecture

> Physical infrastructure layout: VPC, subnets, AZs, and network flows.

![Deployment Diagram](diagrams/03_deployment.svg)

### VPC Design

| Resource | CIDR / Value | Notes |
|---|---|---|
| VPC | `10.0.0.0/16` | Dedicated VPC for the application |
| Public Subnet AZ-1 | `10.0.1.0/24` | Hosts ALB endpoint + NAT Gateway |
| Public Subnet AZ-2 | `10.0.2.0/24` | Hosts ALB endpoint + NAT Gateway |
| Private App Subnet AZ-1 | `10.0.11.0/24` | EC2 Tomcat/JBoss instances |
| Private App Subnet AZ-2 | `10.0.12.0/24` | EC2 Tomcat/JBoss instances |
| Private DB Subnet AZ-1 | `10.0.21.0/24` | Aurora Serverless v2 Writer |
| Private DB Subnet AZ-2 | `10.0.22.0/24` | Aurora Serverless v2 Reader |

### Routing

- **Public subnets**: route `0.0.0.0/0` → Internet Gateway (IGW). Used by ALB and NAT Gateways.
- **Private app subnets**: route `0.0.0.0/0` → NAT Gateway in same AZ. Used by EC2 for outbound AWS API calls (Secrets Manager, CloudWatch, SSM). No direct internet inbound.
- **Private DB subnets**: no default route to the internet. Completely isolated; only reachable via Security Group rules from the app tier.

### High Availability Network Design

- **Two NAT Gateways** (one per AZ) avoid cross-AZ traffic for outbound calls and eliminate the NAT Gateway as a single point of failure.
- **ALB spans both public subnets** providing automatic failover at the load balancer layer.
- **Aurora DB subnet group** includes both private DB subnets, enabling automatic failover across AZs.

### NFR Considerations

- **Reliability**: Full redundancy at every tier across two AZs. In the event of a complete AZ failure, ALB drains the failed AZ, Auto Scaling launches replacement instances in the healthy AZ, and Aurora promotes the reader replica.
- **Security**: Private subnets have no IGW route. DB subnets have no NAT route. EC2 instances are never assigned public IPs.
- **Cost Efficiency**: NAT Gateways are deployed per-AZ to avoid inter-AZ data transfer charges. Aurora Serverless v2 scales to minimum ACU during off-peak hours.

---

## Data Flow

> How data moves through the system at runtime.

![Data Flow Diagram](diagrams/04_data_flow.svg)

### Flow Description

1. **User request** — browser initiates an HTTPS GET to the application's public DNS name (Route 53 → ALB).
2. **WAF inspection** — request is evaluated against OWASP managed rules, IP reputation lists, and rate-limit rules. Malicious requests receive a 403 response.
3. **TLS termination** — ALB decrypts the HTTPS request using the ACM certificate. Forwards plain HTTP to the selected EC2 target on port 8080.
4. **Credential retrieval** — EC2 checks its local in-process cache (TTL 5 min). On cache miss, calls Secrets Manager's `GetSecretValue` API to obtain DB credentials.
5. **Business logic execution** — application processes the request; executes write queries against the Aurora Writer and read-only queries against the Aurora Reader.
6. **Aurora replication** — Aurora Serverless v2 replicates storage synchronously across 6 copies in 3 AZs. This is transparent to the application.
7. **Response** — EC2 builds the response, sends it to ALB, which adds HTTP security headers and delivers HTTPS response to the user.
8. **Observability (async)** — ALB access logs, EC2 application logs, and Aurora performance metrics are pushed to CloudWatch asynchronously without affecting request latency.

### Data at Rest

| Store | Encryption | Key management |
|---|---|---|
| Aurora Serverless v2 | AES-256 (SSE) | AWS managed key (aws/rds) or CMK via KMS |
| S3 (ALB logs) | AES-256 (SSE-S3) | AWS managed key |
| Secrets Manager | AES-256 | AWS managed key (aws/secretsmanager) |

### Data in Transit

| Path | Encryption |
|---|---|
| Browser → ALB | TLS 1.2+ (ACM certificate) |
| ALB → EC2 | Plain HTTP on private subnet (acceptable; no internet exposure) |
| EC2 → Aurora | In-transit encryption (SSL required at Aurora cluster level) |
| EC2 → Secrets Manager | TLS 1.2+ (HTTPS API endpoint) |

---

## Key Workflows

> Step-by-step interaction between components for a standard HTTPS request.

![Sequence Diagram](diagrams/05_sequence.svg)

### Workflow: Standard HTTPS Request

| Step | Actor | Action |
|---|---|---|
| 1 | Browser | Sends `HTTPS GET /app` to ALB public DNS (port 443) |
| 2 | WAF | Evaluates OWASP rules, IP reputation, rate-limit; passes clean traffic |
| 3 | WAF → ALB | Forwards filtered request to ALB |
| 4 | ALB | Logs access entry to CloudWatch/S3 asynchronously |
| 5 | ALB → EC2 | Selects healthy EC2 target; forwards `HTTP :8080` with `X-Forwarded-For` header |
| 6 | EC2 | Checks in-process credentials cache |
| 7 | EC2 → Secrets Manager | `GetSecretValue` call on cache miss |
| 8 | Secrets Manager → EC2 | Returns DB credentials |
| 9 | EC2 → Aurora Writer | Executes write SQL statement |
| 10 | Aurora Writer → EC2 | Returns write acknowledgment |
| 11 | EC2 → Aurora Reader | Executes read-only SQL SELECT |
| 12 | Aurora Reader → EC2 | Returns result set |
| 13 | EC2 → CloudWatch | Emits structured app log and instance metrics (async) |
| 14 | EC2 → ALB | Sends `HTTP 200 OK` with response body |
| 15 | ALB → Browser | Delivers `HTTPS 200 OK` with security headers (`HSTS`, `X-Content-Type-Options`) |

### Workflow: Auto Scaling Scale-Out Event

1. CloudWatch alarm triggers when average CPU > 70% for 2 consecutive minutes.
2. Auto Scaling Group launches a new EC2 instance in the AZ with lower capacity.
3. EC2 bootstrap script starts Tomcat/JBoss, fetches credentials from Secrets Manager.
4. ALB health check (HTTP GET `/health`) passes after grace period (90 s).
5. ALB registers the new instance and begins routing traffic to it.

### Workflow: Aurora Failover (AZ failure)

1. Aurora detects writer instance unavailability.
2. Aurora promotes the Reader Replica (AZ-2) to Writer in < 30 seconds.
3. Aurora updates the cluster write endpoint DNS (automatic — no app-level change needed).
4. EC2 JDBC driver reconnects automatically using cluster endpoint.
5. CloudWatch alarm fires to notify Ops team.

---

## Non-Functional Requirements Analysis

### Scalability

| Dimension | Mechanism |
|---|---|
| **Horizontal web scaling** | EC2 Auto Scaling Group. Scale-out policy on CPU > 70% / RPS threshold. Scale-in after 5-min cooldown. |
| **Database scaling** | Aurora Serverless v2 scales ACUs continuously (0.5–32 ACU per instance) without restart or connection loss. |
| **Read scaling** | Aurora Reader Replica distributes read-only queries, reducing load on the Writer. Additional read replicas can be added (up to 15). |
| **Load balancing** | ALB distributes traffic evenly; supports connection draining during scale-in or deployment. |

### Performance

- **TLS offload at ALB**: EC2 instances receive plain HTTP, removing CPU-intensive TLS overhead from application servers.
- **Connection pooling**: Tomcat/JBoss should use a JDBC connection pool (e.g., HikariCP) sized appropriately for Aurora's max connections per ACU.
- **Credentials caching**: 5-minute local cache on EC2 avoids Secrets Manager API latency on every request.
- **CDN (optional enhancement)**: CloudFront can be placed in front of the ALB for edge caching of static assets, reducing EC2 load and improving global latency.

### Security

| Control | Implementation |
|---|---|
| **Network isolation** | Three-tier subnet design: public (ALB), private app (EC2), private DB (Aurora). |
| **Dedicated Security Groups** | SG-ALB, SG-WEB, SG-DB with explicit allow-rules; default deny everywhere else. |
| **Web application firewall** | AWS WAF with AWS Managed Rules for OWASP Top 10, known bad IPs, and rate-limiting. |
| **TLS in transit** | HTTPS enforced at ALB; SSL/TLS enforced at Aurora cluster level for EC2→Aurora connections. |
| **Encryption at rest** | Aurora, Secrets Manager, S3 all use AES-256 encryption. |
| **Least-privilege IAM** | EC2 Instance Profile grants only `secretsmanager:GetSecretValue` and `logs:PutLogEvents`. |
| **Credential rotation** | Secrets Manager auto-rotates DB credentials every 30 days with zero downtime. |
| **Audit trail** | ALB access logs → S3; CloudTrail for AWS API calls; CloudWatch Logs for application events. |

### Reliability

| Mechanism | RTO / RPO Target |
|---|---|
| **Multi-AZ EC2** | Loss of one AZ: ALB reroutes traffic in < 60 s. RTO ≈ 60 s. |
| **Aurora Multi-AZ** | Writer failure: automatic failover to Reader in < 30 s. RTO ≈ 30–60 s. |
| **Aurora storage** | 6 copies across 3 AZs; can lose 2 copies without data loss. RPO ≈ 0 (synchronous). |
| **Aurora Backtrack / PITR** | Point-in-time recovery up to 35 days. RPO ≈ seconds for PITR. |
| **ALB health checks** | Unhealthy EC2 targets are removed from rotation within 30–60 s. |
| **NAT Gateway HA** | One NAT GW per AZ avoids outbound connectivity loss if one AZ fails. |

### Maintainability

- **Infrastructure as Code**: provision with AWS CDK (Python/TypeScript) or CloudFormation to ensure reproducible, version-controlled deployments.
- **Blue/Green Deployments**: ALB supports weighted target groups for zero-downtime application deployments.
- **CloudWatch Dashboards**: pre-built dashboards for ALB (5xx rate, latency), EC2 (CPU, memory), and Aurora (connections, ACU utilization).
- **AWS Systems Manager (SSM)**: use Session Manager for secure shell access to EC2 instances — no SSH ports open, no bastion host required.
- **Automated patching**: use SSM Patch Manager for OS-level patches with defined maintenance windows.

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Single-AZ outage | Medium | High | Multi-AZ EC2 ASG + Aurora failover. ALB drains and reroutes traffic. |
| Aurora cold-start latency | Low | Medium | Set Aurora minimum ACU ≥ 0.5 (never scales to 0 for production). Use RDS Proxy to pool connections. |
| Secrets Manager API throttling | Low | Medium | Implement in-process credential cache with 5-min TTL. Use exponential backoff on retries. |
| DDoS / Bot attacks | Medium | High | AWS WAF rate-limiting + AWS Shield Standard (included). Shield Advanced for volumetric attacks. |
| Configuration drift | Medium | Medium | Enforce IaC-only deployments via SCPs. Use AWS Config rules to detect drift. |
| Data exfiltration via app | Low | High | SG-DB restricts outbound; VPC Flow Logs + CloudWatch anomaly detection for unusual traffic patterns. |
| Certificate expiry | Low | High | ACM auto-renews managed certificates; CloudWatch alarm if renewal fails. |

---

## Technology Stack Recommendations

| Layer | Recommendation | Justification |
|---|---|---|
| **Web/App Server** | Apache Tomcat 10.x or WildFly (JBoss) 30+ | Mature Java EE / Jakarta EE containers; wide ecosystem support; simple clustering with ALB sticky sessions if needed. |
| **Database Engine** | Aurora Serverless v2 (MySQL 8.0 or PostgreSQL 15) | Auto-scales ACUs; Multi-AZ storage; fully managed; compatible with standard JDBC drivers. |
| **Load Balancer** | Application Load Balancer (ALB) | Layer-7, native WAF integration, built-in health checks, weighted target groups for blue/green. |
| **WAF** | AWS WAF v2 with AWS Managed Rules | Zero-maintenance OWASP ruleset; integrates natively with ALB; supports custom rules. |
| **Secrets** | AWS Secrets Manager | Automatic rotation, IAM-gated access, SDK caching client available for Java. |
| **Observability** | Amazon CloudWatch + CloudWatch Container Insights | Native AWS integration, no extra agents for basic metrics; use CloudWatch Agent for custom metrics. |
| **Infrastructure as Code** | AWS CDK (TypeScript or Python) | Higher-level constructs, type-safety, native CFN backend; well suited for multi-tier VPC architectures. |
| **Shell access** | AWS Systems Manager Session Manager | No bastion host, no SSH port; full audit trail in CloudTrail. |

---

## Cost Estimate

> Estimated **monthly cost** for a baseline production deployment in **eu-west-1** (Ireland). All prices in USD. Costs vary with load and data transfer.

| Service | Configuration | Est. Monthly Cost (USD) |
|---|---|---|
| **EC2 (t3.large × 2)** | 2 instances × 730 h/mo, On-Demand | ~$120 |
| **Aurora Serverless v2** | Writer + Reader, avg 2 ACU/h, 50 GB storage | ~$180 |
| **Application Load Balancer** | 1 ALB, ~10 LCU/h average | ~$30 |
| **NAT Gateway × 2** | 2 NAT GW × 730 h/mo + 50 GB data | ~$80 |
| **AWS WAF** | 1 WebACL + 3 rule groups, 1M req/mo | ~$15 |
| **Secrets Manager** | 2 secrets, 1M API calls/mo | ~$5 |
| **CloudWatch** | Logs 10 GB/mo, 20 metrics, 5 alarms | ~$15 |
| **S3 (ALB logs)** | 5 GB/mo storage | ~$1 |
| **Data Transfer OUT** | 100 GB/mo | ~$9 |
| **AWS Certificate Manager** | Public certificate | Free |
| **Total estimate** | | **~$455 / month** |

> **Note**: Costs can be reduced by ~40–60% using EC2 Reserved Instances (1-year term) and Aurora Serverless v2 auto-scaling to minimum ACU during off-peak hours. Use the [AWS Pricing Calculator](https://calculator.aws) for production sizing.

---

## Next Steps

1. **Provision baseline VPC** using AWS CDK or CloudFormation: VPC, 6 subnets (2 public, 2 private app, 2 private DB), IGW, 2 NAT Gateways, route tables.
2. **Create Security Groups**: SG-ALB, SG-WEB, SG-DB with rules as defined in this document.
3. **Deploy Aurora Serverless v2 cluster**: create DB subnet group, cluster, writer + reader instances, enable encryption, configure parameter group.
4. **Store DB credentials in Secrets Manager**: enable auto-rotation.
5. **Create EC2 Launch Template**: instance profile, SSM agent, CloudWatch agent, Tomcat/JBoss bootstrap script.
6. **Create Auto Scaling Group**: minimum 2, desired 2, max 6, target tracking on CPU 70%.
7. **Request ACM certificate**: DNS validation via Route 53.
8. **Deploy ALB**: public subnets, HTTPS listener with ACM cert, HTTP listener with redirect rule, attach WAF WebACL.
9. **Configure CloudWatch**: log groups, alarms (CPU, 5xx rate, DB connections), dashboard.
10. **Run security review**: verify Security Group rules, IAM policies, enable VPC Flow Logs, enable AWS Config rules.

---

## References

- [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)
- [Aurora Serverless v2 — High Availability](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html)
- [Controlling access with security groups (Aurora)](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Overview.RDSSecurityGroups.html)
- [Scenarios for accessing a DB cluster in a VPC](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_VPC.Scenarios.html)
- [Application Load Balancer — Best Practices](https://docs.aws.amazon.com/wellarchitected/latest/framework/perf_networking_load_balancing_distribute_traffic.html)
- [Using Elastic Load Balancing with Auto Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/autoscaling-load-balancer.html)
- [AWS WAF — Managed Rules](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups.html)
- [AWS Secrets Manager — Rotation](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets.html)
- [VPC Design — AWS Best Practices](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-best-practices.html)
- [AWS Systems Manager Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)
