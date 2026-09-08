# AWS Hong Kong Production Platform Deep Dive

| Field | Value |
| --- | --- |
| Research date | 2026-08-11 (Asia/Hong_Kong) |
| Status | Advisory research; not a cloud, procurement, migration, deployment, or release authorization |
| Workload | `erp-frontend/` Next.js 16.2.7 authenticated UI/BFF, private PostgreSQL, private documents, asynchronous scanner/OCR, and future AI workers |
| Decision reviewed | `docs/TECHNICAL_DECISION_PRODUCTION_PLATFORM.md` |
| Evidence policy | Existing repository source plus AWS and Next.js first-party documentation and public pricing material only |
| Price basis | USD, on-demand planning ranges before tax; accessed 2026-08-11; not a quote |

## 1. Executive conclusion

Retain the current decision to run the complete authenticated Next.js application in AWS Hong Kong (`ap-east-1`) on **ECS Fargate behind an Application Load Balancer**, with at least two web tasks across two Availability Zones, private RDS PostgreSQL Multi-AZ, private S3/KMS, SQS/DLQ, and separate container workers.

Modify the decision package in four places before production approval:

1. Treat the existing Terraform ECS service as a staging health slice only. Its `desired_count = 1`, narrow health-only ALB rule, and 0.25 vCPU/0.5 GiB task are not evidence of a two-AZ production deployment.
2. Keep **RDS PostgreSQL Multi-AZ DB instance** as the Release 1 database baseline. Aurora PostgreSQL is available in Hong Kong and has stronger storage/failover/read-scaling properties, but its minimum useful HA topology and compatibility validation are not justified by the current unmeasured workload.
3. Make the network choice explicit and costed: use gateway/interface VPC endpoints for approved AWS dependencies where available; add NAT only for an approved outbound dependency. Do not silently pay for, or rely on, both paths.
4. Replace any implied Amazon Textract path with a decision gate. As of 2026-08-11, the official Textract endpoint list does **not** include `ap-east-1`; Hong Kong-only OCR therefore requires a self-hosted worker or a separately approved cross-region/provider data flow.

App Runner is not an alternative for this residency boundary: its official endpoint list does not include Hong Kong. EC2 Auto Scaling is technically viable but transfers guest-OS patching, AMI hardening, capacity packing, and host incident work to the team without a demonstrated cost or capability need. Lambda is available in Hong Kong and is useful for bounded event workers, but adopting it for the full Next.js server would add a framework adapter and materially different timeout, payload, streaming, concurrency, and connection behavior.

Under the replaceable assumptions in Section 5, the recommended topology has a broad monthly planning range of approximately:

| Stage | Planning range (USD/month) | Primary uncertainty |
| --- | ---: | --- |
| Pilot: 1 tenant | **$300-$650** | private networking topology, paid support, logs, scanner duty cycle |
| Growth: 10 tenants | **$650-$1,900** | web/worker utilization, DB class, OCR pages, log volume |
| Scale: 50 tenants | **$1,600-$6,400** | peak concurrency, DB/vector workload, OCR, retention/versioning, egress |

These figures are not a budget approval or AWS quote. They deliberately retain wide ranges until a pinned image is load-tested and an exact AWS Pricing Calculator export is reviewed.

## 2. Problem, stakeholders, and boundaries

### Problem being solved

Choose a production platform that can run the repository's authenticated UI and BFF while keeping authorization, documents, queues, audit, logs, backups, and operations inside the approved Hong Kong boundary. The platform must also support multi-instance Next.js behavior, controlled deployment/rollback, asynchronous document processing, and future isolated AI workers.

### Stakeholder outcomes

- Customers and data subjects need private case/document data to remain inside the approved boundary and to fail closed on authorization failure.
- Advisors need predictable interactive latency and no document availability before clean scan state.
- Security and Privacy need one inspectable VPC/IAM/logging boundary and explicit approval for every cross-region service.
- Operations need bounded failover, deploy, rollback, restore, capacity, and cost procedures.
- Engineering needs to preserve the existing application contracts without introducing an unproved runtime adapter or second cloud control plane.

### Out of scope

- No AWS account inventory, API call, provisioning, Terraform plan/apply, migration, secret inspection, deployment, DNS change, or purchase was performed.
- This is not a legal conclusion that an AWS Region alone proves end-to-end residency. Support, telemetry, email/SMS, model inference, and subprocessors still require service-specific review.
- This does not choose commercial plan limits, customer pricing, a managed AI model, or a cross-region disaster-recovery region.

## 3. Current repository facts and migration surface

The current system is not a generic static frontend:

- `erp-frontend/package.json` pins Next.js `16.2.7`, React `19.2.4`, Node `>=20.11.0`, `pg`, and the existing Neon adapter. The repository contains UI, Route Handlers/BFF, authorization, documents, audit, and workers in one deployable application boundary.
- `infra/terraform/modules/web-runtime/main.tf` already declares ECS Fargate `awsvpc`, private tasks, an ALB, CloudWatch Logs, task/execution roles, a deployment circuit breaker, and `ap-east-1` log routing.
- `infra/terraform/modules/rds/main.tf` declares private PostgreSQL 17, Multi-AZ, gp3, managed master secret, IAM database authentication, forced TLS, backups, deletion protection, and a Hong Kong `rds-db:connect` ARN.
- `infra/terraform/modules/document-store/main.tf` declares a private, versioned, SSE-KMS S3 bucket, SQS scan queue and DLQ, opaque document prefix, and no cross-region replication, CloudFront, MRAP, or transfer acceleration.
- Application runtime and tests validate Hong Kong RDS hostnames, AWS SigV4 upload capabilities, and literal `ap-east-1` document/storage boundaries. Cognito is represented in routes, error contracts, and revocation work.

A repository scan found AWS/Cognito/S3/RDS references across 117 source, test, and infrastructure files and approximately 1,218 Terraform lines. These counts show a meaningful tested AWS boundary; they do **not** imply every matching file must be rewritten for another platform. A migration estimate must classify each occurrence as provider adapter, policy invariant, test fixture, IaC, or provider-neutral domain logic before assigning effort.

The present staging ECS slice is intentionally incomplete for production: `desired_count = 1`, a health-only listener rule, and a small health task prove static intent, not multi-AZ availability, full route exposure, production autoscaling, or production capacity.

## 4. Non-negotiable invariants and enforcement owners

| Invariant | Enforcement owner | Required evidence |
| --- | --- | --- |
| Every authenticated request revalidates session version, membership, organization, case scope, capability, and expiry from RDS. | Identity/access repository transaction | revocation and cross-tenant negative tests under concurrency |
| Web, DB, document, queue, scanner, audit, logs, and backups stay in approved Hong Kong services. | IaC + Privacy inventory | exact regional resource/data-flow manifest and endpoint probes |
| At least two healthy web instances remain available across an AZ/task failure. | ECS service + ALB + Operations | forced task stop, AZ impairment scenario, drain and alarm receipts |
| Multi-instance Next.js uses one immutable build identity, Server Action encryption key, and coordinated cache/tag invalidation. | Release/runtime composition | mixed-version deploy and rollback tests; Next.js self-hosting checklist |
| Uploaded bytes remain quarantined until an authorized scanner result; retries are idempotent and poison work reaches a DLQ. | Documents module + worker | duplicate/out-of-order event, timeout, malware, and DLQ replay tests |
| DB failover never turns stale browser/IdP claims into authorization truth. | Application + RDS | failover test with connection recovery and request-time policy checks |
| Mandatory audit failure rolls back mutation; optional telemetry failure degrades explicitly without blocking business flow. | Audit/telemetry repositories | failure injection and reconciliation evidence |
| AI workers receive scoped tools, not ambient DB/S3 access, and model inference residency is approved separately. | Agent/tool gateway + Privacy | tool policy, data-class, budget, exact-payload approval, redacted receipts |

## 5. Replaceable workload assumptions

The following common case/document assumptions are planning inputs, not forecasts:

| Input | Pilot | Growth | Scale |
| --- | ---: | ---: | ---: |
| Tenants | 1 | 10 | 50 |
| Users | 10 | 100 | 500 |
| Active cases | 10 | 100 | 500 |
| Retained cases | 100 | 1,000 | 5,000 |
| Logical documents | up to 50 GB | about 500 GB | about 2.5 TB |
| New bytes/new versions per month | 10% of logical bytes | 10% | 10% |
| Internet download sensitivity | 20% of logical bytes/month | 20% | 20% |

The 10% write/version and 20% download factors are sensitivity parameters. They must be replaced by measured pilot values. S3 billed bytes may exceed logical bytes because versioning, quarantine, retention, incomplete multipart uploads, and noncurrent versions have separate lifecycles.

Do not infer the following dimensions from case count. The low/high bands are independent capacity probes:

| Independent dimension | Pilot low-high | Growth low-high | Scale low-high |
| --- | ---: | ---: | ---: |
| HTTP requests/month | 0.5M-5M | 5M-25M | 25M-100M |
| Peak authenticated requests/second | 5-25 | 25-100 | 100-400 |
| Structured DB data | 20-100 GB | 100-500 GB | 0.5-2 TB |
| Average DB transactions/second | 5-30 | 25-150 | 100-750 |
| Derived vector/index data | 1-5 GB | 10-50 GB | 50-250 GB |
| CloudWatch log ingestion/month | 5-20 GB | 20-100 GB | 80-400 GB |
| OCR pages/month | 0-10k | 10k-100k | 50k-500k |
| Concurrent scanner/OCR jobs | 1-2 | 2-10 | 5-30 |

Required capacity test: replay representative authenticated pages, API mutations, 500 MB document lifecycles, scan backlog, DB failover, and a noisy tenant against a pinned image. Record CPU, memory, event-loop delay, open DB connections, ALB target latency, queue age, DB load/IOPS, log bytes, and cost per completed workflow.

## 6. Compute option comparison

### 6.1 Decision matrix

| Option | Hong Kong availability (2026-08-11) | Next.js 16 authenticated BFF fit | Private RDS/S3/SQS/scanner | Availability and operations | Lock-in / rollback | Verdict |
| --- | --- | --- | --- | --- | --- | --- |
| **ECS Fargate + ALB** | Available in `ap-east-1` | Runs the standard Node/Docker server without a serverless framework adapter; supports long-running process and streaming | Each task receives an ENI and security groups; gateway/interface endpoints or approved NAT provide outbound paths | ECS service, ALB health, target tracking, deployment circuit breaker; no guest OS management; retain two-task minimum | Container image is portable; ECS/IAM/ALB/IaC are AWS-specific; roll back task definition/image digest | **Recommended** |
| **App Runner** | **Not available**: official endpoint list omits `ap-east-1` | Managed container fit in supported regions | VPC connector supports outbound private resources, but private/public ingress and PrivateLink add distinct controls | Lower platform work, but unsupported region is terminal for HK-only runtime | App Runner service config is AWS-specific; container remains portable | **Reject for current boundary** |
| **EC2 + ASG + ALB** | Available | Standard Node/Docker or ECS-on-EC2; maximum host/control flexibility | Full VPC access and local scanner optimizations | ASG health/instance refresh work, but team owns AMI, guest OS patching, agents, host capacity, bin-packing, and drain correctness | Easiest host-level portability; AMI/runbook debt increases rollback surface | **Reserve for measured need** |
| **Lambda + API Gateway/Function URL** | Available | Requires packaging/adapter and a production feature matrix; invocation model differs from a resident Next.js server | VPC attachment and SQS events supported; concurrency can multiply DB connections | Automatic request scaling; regular function max 15 minutes, cold starts and quotas must be tested | Function/API/event wiring is AWS-specific; version/alias rollback is strong | **Use selectively for bounded workers, not default web plane** |

Primary sources: [Next.js self-hosting](https://nextjs.org/docs/app/guides/self-hosting), [Fargate regions](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate-Regions.html), [Fargate task networking](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-task-networking.html), [App Runner endpoints](https://docs.aws.amazon.com/general/latest/gr/apprunner.html), [Lambda endpoints](https://docs.aws.amazon.com/general/latest/gr/lambda-service.html), [Lambda timeout](https://docs.aws.amazon.com/lambda/latest/dg/configuration-timeout.html), and [EC2 shared responsibility](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-security.html) (availability and product behavior checked 2026-08-11).

### 6.2 Why Fargate remains the best current fit

- It matches the repository's existing immutable container, task IAM role, private subnet, ALB, CloudWatch, RDS, S3, SQS, and KMS model.
- Fargate `awsvpc` gives each task an ENI and security-group boundary. The same network path can be inspected with VPC Flow Logs, while endpoints can keep supported AWS service traffic off public internet paths.
- ECS target tracking can scale on CPU, memory, or ALB requests per target. Scale-out does not remove the requirement for a two-task floor across AZs.
- ECS deployment circuit breaker rollback is useful but insufficient alone: application health, schema compatibility, cache/build identity, background jobs, and data correctness need independent canary gates.
- The team avoids guest OS and node capacity management while keeping ordinary OCI container portability.

### 6.3 When EC2/ASG becomes justified

Reassess an ECS-on-EC2/ASG pool only when measured steady utilization shows at least 25-30% compute savings after including patch/on-call labor, or when a required scanner/OCR/native dependency cannot run within Fargate limits. Any proposal must include hardened AMI ownership, SSM access, patch SLA, instance refresh with checkpoints/auto-rollback, capacity rebalancing, drain behavior, and host-compromise blast radius. AWS explicitly assigns guest OS and software patching to the EC2 customer; ASG replacement does not remove that responsibility ([EC2 security](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-security.html), [instance refresh](https://docs.aws.amazon.com/autoscaling/ec2/userguide/instance-refresh-overview.html)).

### 6.4 Where Lambda is appropriate

Lambda is a reasonable Hong Kong option for short, idempotent SQS consumers, metadata extraction, notification fan-out, and reconciliation ticks. It is not automatically suitable for malware engines, large OCR batches, long agent runs, or the complete Next.js runtime. Regular Lambda functions have a 900-second maximum; buffered responses are limited to 6 MB, while response streaming has separate regional/VPC constraints and a 200 MB maximum ([Lambda timeout](https://docs.aws.amazon.com/lambda/latest/dg/configuration-timeout.html), [response streaming](https://docs.aws.amazon.com/lambda/latest/dg/configuration-response-streaming.html)).

Before using Lambda for any path, test cold/warm latency, native binary size, `/tmp`, event duplicate handling, partial batch failure, reserved concurrency, DLQ/reconciliation, VPC start/connect time, DB connection pressure, and feature availability specifically in `ap-east-1`.

## 7. Database: RDS PostgreSQL Multi-AZ vs Aurora PostgreSQL

| Criterion | RDS PostgreSQL Multi-AZ DB instance | Aurora PostgreSQL (writer + cross-AZ reader) |
| --- | --- | --- |
| Hong Kong | Available; already represented in repository IaC | Available in `ap-east-1` as of 2026-08-11 ([Aurora regions](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.RegionsAndAvailabilityZones.html)) |
| HA topology | Primary plus synchronous standby in another AZ; standby cannot serve reads | Shared cluster volume stores six copies across three AZs; reader can serve reads and be promoted |
| Typical failover | AWS documents 60-120 seconds, workload/recovery dependent | With an existing replica, AWS documents typically under 60 seconds and often under 30 seconds |
| Read scaling | Add separate read replicas; standby is unavailable for reads | Reader endpoint and up to 15 replicas |
| Storage/I/O | Provisioned gp3 size/IOPS/throughput; predictable for a small DB | Storage grows automatically; Standard charges I/O requests, I/O-Optimized prices compute/storage higher but removes I/O request charges |
| PostgreSQL compatibility | Standard managed PostgreSQL; least change from current migrations/extensions | PostgreSQL-compatible, not identical; extension/version/parameter/query-plan validation required |
| Cost floor | Small burstable Multi-AZ classes fit the pilot baseline | Meaningful HA requires at least writer + reader compute; generally a higher small-workload floor |
| Operations | Simple endpoint; connection pool must recover from DNS failover | Cluster/reader endpoints, failover tiers, replica lag and extra topology; RDS Proxy optional |
| Migration/rollback | Current target; standard `pg_dump`/restore/logical replication options | Snapshot/replication migration exists, but rollback after writes requires a planned reverse path, not a button |

Sources: [RDS Multi-AZ single standby](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZSingleStandby.html), [RDS failover](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.Failover.html), [Aurora architecture](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.html), [Aurora high availability](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html), and [Aurora pricing modes](https://aws.amazon.com/rds/aurora/pricing/) (checked 2026-08-11).

### Database recommendation

Use RDS PostgreSQL Multi-AZ for Release 1 and measure it. The current authorization-heavy modular monolith benefits more from compatibility and a lower cost floor than from speculative read replicas. Use connection pooling with a hard per-task budget, short acquisition timeout, transaction timeout, and retry only for explicitly classified transient failures. A two-task web floor plus workers must not be allowed to exhaust DB connections during autoscaling.

Reassess Aurora only when one of these is measured:

- DB failover recovery objective is below 60 seconds and RDS Multi-AZ drills cannot meet it;
- sustained read demand needs at least one full read replica;
- storage growth/IOPS administration becomes material operational work;
- monthly RDS compute/storage/IO plus replica cost is within 15% of a validated Aurora HA topology;
- Aurora-specific capability has an approved business requirement.

Aurora migration evidence must include extension/version compatibility, query plans, RLS/session-variable behavior, IAM auth, migration duration, replication lag, writer endpoint failover, connection recovery, backup/PITR, and a reverse/restore cutoff. Do not claim rollback after dual writes without an approved conflict model.

## 8. Recommended production topology

```text
Internet
  -> Route 53 / ACM
  -> regional WAF
  -> public ALB in >=2 ap-east-1 AZs
  -> ECS web service, private subnets, minimum 2 Fargate tasks
       -> Cognito ap-east-1 (authentication only)
       -> RDS PostgreSQL Multi-AZ, private subnets
       -> S3 gateway endpoint -> private versioned SSE-KMS documents
       -> SQS interface/public-via-approved-egress -> scan queue + DLQ
       -> approved interface endpoints: ECR, Logs, Secrets/SSM as required

S3 object-created event
  -> SQS scan queue
  -> independent ECS scanner/OCR service, private subnets
       -> malware scanner
       -> HK self-hosted OCR if approved
       -> transactional result command + audit/outbox

Future AgentRun
  -> HK SQS queue
  -> isolated ECS agent worker
       -> versioned tool gateway
       -> separately approved model adapter
       -> no ambient RDS/S3 access
```

Use one immutable image digest per service and make DB migrations backward-compatible across the whole rollback window. Keep web, scanner/OCR, reconciliation, and AI worker services separate so a document burst or agent loop cannot consume the interactive request envelope.

## 9. Availability, capacity, operations, and rollback

### Availability baseline

- Web: minimum two tasks across two AZs, ALB health checks, graceful SIGTERM/draining, autoscaling headroom, and deployment circuit breaker.
- DB: Multi-AZ, seven-day minimum automated backup during pilot, deletion protection, final snapshot, and measured failover/restore. Multi-AZ is not a backup and does not protect against bad writes.
- Documents: S3 versioning and explicit current/noncurrent lifecycle; no replication outside Hong Kong. Versioning is not a retention/legal-hold policy.
- Queue: bounded retries, visibility timeout greater than measured job heartbeat/recovery policy, DLQ alarms, idempotent result command, and replay runbook.
- Region failure: the current Hong Kong-only requirement accepts a regional outage unless a separate privacy/legal decision approves an external region. Do not silently create cross-region replicas.

### Deployment and rollback

1. Build once; sign/scan and record Git SHA, image digest, SBOM, Next.js build ID, configuration checksum, Server Action encryption key version, and migration compatibility window.
2. Deploy a canary task set; check synthetic auth, revocation, case, mutation/audit, upload-intent, and queue probes.
3. Shift traffic only after ALB, application, DB, and error-budget gates pass.
4. Roll back to the previous image digest while schema remains backward-compatible. Database rollback is an approved restore/forward-fix operation, never an implicit application-image action.
5. If cache/build/server-action behavior differs across mixed versions, stop rollout rather than forcing client retries to hide incompatibility.

### Operational ownership

Fargate removes host patching but not application/runtime upgrades, capacity measurement, IAM, networking, image vulnerability response, log redaction, cost control, backup restore, or incident response. The production owner needs alarms for ALB 5xx/latency, task churn, CPU/memory/event loop, RDS CPU/free memory/connections/storage/replica state, SQS age/DLQ, S3 growth, KMS errors, WAF blocks, audit persistence, and budget anomaly.

## 10. Three-stage monthly cost model

### Method and interpretation

The model uses 730 hours/month and on-demand public pricing as a planning basis. The prior repository research records these `ap-east-1` baselines: two ARM Fargate tasks at 1 vCPU/2 GiB are about $79/month; one ALB plus average 1 LCU is about $27/month; RDS PostgreSQL Multi-AZ `db.t4g.small` plus 20 GiB gp3 is about $87/month. Current rates must be refreshed from the public [AWS Price List API](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/price-changes.html) or [AWS Pricing Calculator](https://calculator.aws/) before approval.

The low/high ranges are capacity scenarios, not confidence intervals. They exclude tax, staff/on-call labor, CI, domain registration, email/SMS, legal review, penetration testing, third-party model tokens, and cross-region DR. Free tiers and credits are not assumed.

### Cost breakdown (USD/month)

| Component | Pilot | Growth: 10 tenants | Scale: 50 tenants | Included assumptions / main sensitivity |
| --- | ---: | ---: | ---: | --- |
| Web + ALB compute | 105-150 | 190-450 | 450-1,500 | two-task floor; higher bands add task size/count and ALB LCUs |
| RDS PostgreSQL Multi-AZ | 87-160 | 170-500 | 450-1,500 | instance class, structured DB size, gp3/IOPS, connections; Aurora not included |
| S3/KMS/SQS storage + requests | 3-15 | 15-60 | 70-260 | 50 GB/500 GB/2.5 TB logical docs plus version/lifecycle multiplier |
| Internet egress | 0-5 | 5-25 | 35-140 | 20% download sensitivity; no free-tier assumption; direct presigned S3 path |
| Logs/metrics/alarms | 5-35 | 20-120 | 80-500 | 5-20/20-100/80-400 GB log ingestion plus retention/query metrics |
| WAF | 10-30 | 12-50 | 20-120 | web ACL, rule groups, request volume; managed-rule subscriptions can add cost |
| NAT **or** VPC endpoints | 40-130 | 50-180 | 80-350 | topology choice, endpoints x AZ, NAT hours/data; do not add both blindly |
| Paid AWS Support | 29-60 | 60-170 | 140-580 | Business Support+ is greater of $29/account or current tiered percentage |
| Backup/PITR growth | 2-15 | 10-60 | 40-220 | DB backup beyond allowance, snapshots, document noncurrent versions |
| Scanner compute | 10-45 | 30-180 | 100-750 | independent worker duty cycle, binary/signature updates, backlog concurrency |
| OCR | 0-20 | 15-200 | 75-900 | **HK self-hosted compute estimate only**; pages/quality/languages dominate |
| **Rounded planning total** | **300-650** | **650-1,900** | **1,600-6,400** | rounded after correlated sizing; do not sum every independent maximum as a forecast |

Official price surfaces used for category construction, checked 2026-08-11: [ECS/Fargate](https://aws.amazon.com/fargate/pricing/), [Elastic Load Balancing](https://aws.amazon.com/elasticloadbalancing/pricing/), [RDS PostgreSQL compute/storage/backup](https://aws.amazon.com/rds/postgresql/pricing/), [S3](https://aws.amazon.com/s3/pricing/), [SQS](https://aws.amazon.com/sqs/pricing/), [KMS](https://aws.amazon.com/kms/pricing/), [data transfer](https://aws.amazon.com/ec2/pricing/on-demand/#Data_Transfer), [CloudWatch](https://aws.amazon.com/cloudwatch/pricing/), [WAF](https://aws.amazon.com/waf/pricing/), [NAT Gateway](https://aws.amazon.com/vpc/pricing/), [PrivateLink](https://aws.amazon.com/privatelink/pricing/), and [AWS Support](https://aws.amazon.com/premiumsupport/pricing/).

### Network cost choice

S3 gateway endpoints do not carry hourly gateway endpoint charges. Interface endpoints normally charge per endpoint-AZ-hour and data processed; NAT Gateway charges per gateway-hour and data processed. A private task that needs ECR image pull, logs, secrets, SQS, Cognito/public identity endpoints, external malware signature updates, or an overseas model may need multiple endpoints and/or NAT.

Create an explicit matrix of each outbound hostname/service, data class, endpoint availability, required AZs, expected GB, fail-closed behavior, and owner. Compare:

- endpoint-first: S3 gateway plus only the approved interface endpoints;
- one NAT with acknowledged AZ dependency for pilot;
- one NAT per AZ where NAT availability is itself part of the target;
- controlled proxy/egress firewall for approved external destinations.

Do not claim a fully private path for Cognito or another service until its specific application endpoint and DNS path are tested from the task subnet.

### OCR and scanner cost warning

Amazon Textract's official endpoint list omits Hong Kong as of 2026-08-11 ([Textract endpoints](https://docs.aws.amazon.com/general/latest/gr/textract.html)). Therefore the OCR row is not Textract pricing and does not authorize sending document bytes to Singapore, Seoul, or another region. A Hong Kong self-hosted OCR benchmark must measure language accuracy, CPU/memory, page time, native dependencies, malware isolation, timeout, and human-review rate.

Scanner cost similarly depends on duty cycle and signature distribution. Keep scanner tasks separate from web tasks, use immutable definitions with a controlled signature update path, and fail a document to review/DLQ rather than make it available when the scanner is unavailable.

### Aurora cost sensitivity (not in totals)

For planning only, replace the RDS row with approximately **$300-$650 pilot**, **$450-$1,300 growth**, or **$900-$3,200 scale** for an Aurora PostgreSQL HA topology. The range reflects at least writer + cross-AZ reader compute, storage, backup, and Standard I/O versus I/O-Optimized uncertainty. It is not a quote. Generate an exact `ap-east-1` Calculator export using the measured DB load and compare total DB cost, not instance headline price.

## 11. AI worker fit

Initial AI orchestration should remain provider-neutral ECS workers in Hong Kong:

- `AgentRun` fixes organization/case scope, policy version, model adapter, tool schema, source snapshot, expiry, idempotency key, and cost/time/turn budget.
- Agent workers consume an HK queue and call only versioned tool-gateway commands. They do not receive general DB credentials, bucket list permission, or the launching user's ambient access.
- High-impact mutations require exact-payload human approval and the same application authorization/audit/outbox transaction as non-AI actions.
- Model inference location is a separate decision. A Hong Kong ECS worker calling an overseas model is still a cross-border data flow.
- Long or stateful agent runs fit ECS better than regular Lambda's 15-minute invocation. Lambda remains suitable for bounded fan-out/evaluation steps with idempotent checkpoints.

Amazon Bedrock AgentCore availability must not drive the core platform until its runtime, model, trace, state, support, and networking services all meet the approved regional boundary. The prior first-party review found no Hong Kong AgentCore region; recheck its [official region list](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agentcore-regions.html) at the time of any proposal.

## 12. Lock-in, migration, and exit

### Current lock-in

- Low/moderate: OCI image, Node server, PostgreSQL SQL/domain model, HTTP contracts, and provider adapters are portable in principle.
- Moderate: ECS service definitions, ALB/WAF, IAM task roles, CloudWatch, KMS policies, S3 events, SQS/DLQ, Cognito operations, RDS IAM authentication, and Terraform resources are AWS-specific.
- High-risk if unmanaged: runtime code that assumes Cognito error shapes, AWS SigV4 URLs, `ap-east-1` hostnames, or provider event delivery without adapters.

The correct mitigation is to keep business state/authorization in PostgreSQL and provider operations behind tested ports. Do not create a lowest-common-denominator abstraction before a second provider is approved; preserve exportable data schemas, documented event contracts, and deterministic restore tests instead.

### Migration/rollback requirements

Any move to EC2, Lambda, Aurora, or another cloud needs:

1. inventory/classification of the 117 matching files and Terraform surface rather than a raw replacement count;
2. parallel environment from one immutable image or explicit adapter build;
3. auth/session/revocation, RLS, document capability, queue duplicate, audit, and telemetry parity tests;
4. DB snapshot/logical replication with a write cutover and reverse cutoff;
5. object manifest/count/checksum and version/retention reconciliation;
6. queue drain/freeze/replay rules and side-effect idempotency receipts;
7. DNS/session/cache drain and exact rollback decision time;
8. measured RTO/RPO and evidence that rollback does not reintroduce stale authorization or lose audit facts.

## 13. Decision changes and reassessment triggers

### Retain

- AWS Hong Kong as the complete authenticated production plane.
- ECS Fargate + ALB for Next.js web/BFF and separate long-running workers.
- Cognito `ap-east-1` for authentication, opaque BFF sessions, and RDS-authoritative business authorization.
- RDS PostgreSQL Multi-AZ, private S3/KMS, SQS/DLQ, quarantine-first documents, and transaction-bound audit.
- Provider-neutral AI/model/tool adapters and explicit approval for overseas inference.

### Modify before approval

- Production IaC must declare at least two web tasks across two AZs; do not promote the staging health slice unchanged.
- Add ECS autoscaling limits, task draining, full ALB/WAF routes, endpoint/NAT matrix, endpoint policies/security groups, and alarms.
- Add exact S3 current/noncurrent/incomplete-multipart lifecycle and cost evidence for the 50 GB/500 GB/2.5 TB scenarios.
- Add self-hosted scanner/OCR capacity and accuracy evidence; leave OCR disabled/manual if no Hong Kong path passes.
- Add a pinned load/cost report and exact Calculator export for each gate. Budget owner approves a ceiling and anomaly/stop owner.

### Explicit triggers

Reopen the platform decision when any one occurs:

- measured web utilization stays above 60% CPU or 70% memory at the approved task maximum, or P95 API latency exceeds 500 ms for two review windows;
- the service needs more than 12 always-on 2 vCPU/4 GiB web tasks, or an EC2/ASG model demonstrates at least 25-30% total compute savings after operational labor and HA capacity;
- RDS connections exceed 70% of the configured maximum, DB CPU exceeds 60% sustained, storage/IOPS alarms recur, or read demand justifies a full replica;
- RDS Multi-AZ drills cannot meet the approved RTO, particularly if an RTO below 60 seconds is adopted;
- scanner queue oldest-message age exceeds the approved document availability SLO or 24-hour worker utilization exceeds 60%;
- actual billed document bytes exceed logical bytes by 1.5x, monthly new versions exceed 10%, or download exceeds 20% for two months;
- CloudWatch/log cost exceeds 15% of AWS spend, NAT/endpoints exceed 20%, or the monthly total exceeds the approved stage ceiling by 20%;
- a required native binary, GPU, filesystem, connection, or execution duration cannot be supported safely on Fargate;
- App Runner adds `ap-east-1` and passes the full private networking, WAF, observability, deployment, and Next.js matrix;
- a managed OCR/model/agent service offers complete Hong Kong runtime/state/log/support evidence and passes privacy approval;
- a second tenant, regulated customer, or cross-region DR requirement changes isolation, RTO/RPO, support, or residency contracts.

## 14. Approval evidence still required

- Exact `ap-east-1` service/feature availability inventory and redacted Terraform plan hash.
- AWS Calculator exports for pilot, growth, and scale with rate date, account support plan, endpoint/NAT topology, backup and egress.
- Pinned container load report and web/worker/DB connection capacity model.
- Next.js multi-instance build ID, deployment ID, Server Action key, cache/tag, streaming, graceful drain, and rollback evidence.
- RDS failover and point-in-time restore drills; backup alone is not restore evidence.
- 500 MB document upload, multipart cleanup, scan backlog, duplicate event, malware, OCR failure, DLQ, and lifecycle cost tests.
- WAF/rate-limit, log redaction/retention, alert delivery, budget anomaly, and incident runbooks.
- Service-specific privacy/legal review for Cognito optional features, support access, malware signature source, OCR, AI inference, and any external egress.

## 15. Research limitations and terminal state

- Public Price List JSON could not be fetched from the restricted research shell; the model therefore uses dated repository Price List results and official public price pages. Exact approval requires a preserved Price List/Calculator export and checksum.
- Relevant local Next.js documentation files under `node_modules/next/dist/docs/` were identified, but filesystem reads did not complete in the research environment. The official Next.js self-hosting URL and the prior repository review of the same guide were used; production implementation must reopen the pinned local 16.2.7 guide.
- No performance, failover, restore, scanner, OCR, or cost benchmark was run. The numeric ranges are scenario envelopes, not measured demand.
- No secrets, cloud account, customer data, or production resources were accessed.

Terminal state: **research_passed_advisory**. ECS Fargate + ALB and RDS PostgreSQL Multi-AZ remain the recommended Release 1 baseline, with the production availability, networking, OCR, capacity, cost, and restore gates above still pending human approval and deterministic evidence.

## 16. Primary source index

All time-sensitive product and price claims were checked on 2026-08-11.

- Next.js: [Self-hosting](https://nextjs.org/docs/app/guides/self-hosting), [Deploying](https://nextjs.org/docs/app/getting-started/deploying)
- ECS/Fargate: [Regions](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate-Regions.html), [task networking](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-task-networking.html), [service autoscaling](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/target-tracking-create-policy.html), [deployments](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/create-service-console-v2.html)
- App Runner: [endpoints](https://docs.aws.amazon.com/general/latest/gr/apprunner.html), [VPC outbound](https://docs.aws.amazon.com/apprunner/latest/dg/network-vpc.html), [private ingress](https://docs.aws.amazon.com/apprunner/latest/dg/network-pl.html)
- EC2/ASG: [shared responsibility](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-security.html), [Auto Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html), [instance refresh](https://docs.aws.amazon.com/autoscaling/ec2/userguide/instance-refresh-overview.html)
- Lambda: [endpoints](https://docs.aws.amazon.com/general/latest/gr/lambda-service.html), [timeout](https://docs.aws.amazon.com/lambda/latest/dg/configuration-timeout.html), [response streaming](https://docs.aws.amazon.com/lambda/latest/dg/configuration-response-streaming.html), [quotas](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html)
- PostgreSQL: [RDS Multi-AZ](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZSingleStandby.html), [RDS failover](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.Failover.html), [Aurora regions](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.RegionsAndAvailabilityZones.html), [Aurora HA](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html), [Aurora pricing](https://aws.amazon.com/rds/aurora/pricing/)
- Data and processing: [S3 endpoints](https://docs.aws.amazon.com/general/latest/gr/s3.html), [SQS endpoints/quotas](https://docs.aws.amazon.com/general/latest/gr/sqs-service.html), [Textract endpoints](https://docs.aws.amazon.com/general/latest/gr/textract.html)
- Cost: [Fargate](https://aws.amazon.com/fargate/pricing/), [ELB](https://aws.amazon.com/elasticloadbalancing/pricing/), [RDS PostgreSQL](https://aws.amazon.com/rds/postgresql/pricing/), [S3](https://aws.amazon.com/s3/pricing/), [CloudWatch](https://aws.amazon.com/cloudwatch/pricing/), [WAF](https://aws.amazon.com/waf/pricing/), [VPC](https://aws.amazon.com/vpc/pricing/), [PrivateLink](https://aws.amazon.com/privatelink/pricing/), [Support](https://aws.amazon.com/premiumsupport/pricing/)
