# AWS + Cloudflare Hybrid Cost and Residency Check

| Field | Value |
| --- | --- |
| Research date | 2026-08-11 (Asia/Hong_Kong) |
| Status | Advisory research only; no procurement, account access, migration, cloud operation, deployment, or release authority |
| Question | Whether AWS + Cloudflare is more suitable than the current AWS Hong Kong production direction because Cloudflare has no egress fee |
| Evidence rule | Cloudflare and AWS first-party documentation and official price lists only |
| Price basis | USD, on-demand public list prices before tax/support/enterprise contracts; not a quote |

## 1. Decision

**Do not replace the current AWS Hong Kong sensitive-data plane with a cross-cloud Cloudflare core.** Keep RDS PostgreSQL, private documents, scanner state, audit, logs, and backups in AWS `ap-east-1`.

The statement "Cloudflare's database has no egress fee" conflates different products:

- **R2** is S3-compatible object storage. R2 Standard charges `$0.015/GB-month`, Class A writes at `$4.50/million`, Class B reads at `$0.36/million`, and no direct R2 Internet egress. It is the product that can materially reduce object-download cost.
- **D1** is a managed database with **SQLite semantics**, not object storage and not PostgreSQL. It also has no data-transfer fee, but that fact does not make it a replacement for RDS PostgreSQL.
- **Workers** is compute. The Paid plan has a `$5/month` minimum, included requests/CPU, and metered overages. A Worker is not required for direct R2 S3 API access, but it is billed when introduced for authorization, transformations, cache logic, or database access.
- **Hyperdrive** is a connection pool/cache in front of an existing PostgreSQL/MySQL database. It does not move the system of record out of RDS and cannot erase AWS-origin network transfer, cross-cloud latency, or residency exposure.

Cloudflare is appropriate for a **strictly separated public plane**: immutable public assets, approved anonymous school facts, generic health pages, DNS/WAF/CDN after a data-flow review. It is not currently acceptable for authenticated Tianxing requests or sensitive documents because R2 offers only an `apac` best-effort location hint, not a Hong Kong jurisdiction, while Cloudflare's Hong Kong Regional Services controls do not regionalize Worker subrequests, Queue/Cron triggers, globally deployed code/secrets, or all metadata.

## 2. Scope, invariants, and exclusions

### In scope

- R2, D1, Workers, and Hyperdrive public pricing and relevant product contracts.
- S3 Standard in `ap-east-1` versus R2 Standard under the shared 50 GB / 500 GB / 2.5 TB model.
- Direct R2 storage, S3 plus Cloudflare public caching, and the proposed sensitive-document cross-cloud path.
- Migration, API compatibility, event, version, scanning, recovery, residency, and failure risks.

### Out of scope

- No AWS or Cloudflare account, calculator, credential, secret, API, provisioning, copy, benchmark, or deployment was used.
- Enterprise pricing for Cloudflare Regional Services, Customer Metadata Boundary, support, WAF, Cache Reserve, Argo, and private networking is not public and is excluded.
- Scanner/OCR compute, CloudTrail/CloudWatch, KMS, SQS, RDS, ECS, support, tax, and engineering labor are outside the narrow storage table unless explicitly stated.
- This is not legal advice. Privacy/legal owners must approve the exact content, metadata, logging, support, and subprocessor boundary.

### Non-negotiable invariants

| Invariant | Enforcement owner |
| --- | --- |
| Sensitive content, metadata, authorization, audit, logs, backup, and processing remain in the approved Hong Kong boundary | Infrastructure + Privacy; a cheaper storage adapter cannot waive this |
| Private bytes are never downloadable before a typed clean-scan receipt | Documents module + scanner state machine |
| Every logical version is immutable and independently recoverable; active version is a DB pointer | Documents module + object store + recovery runbook |
| Tenant/case authorization is checked at request time against PostgreSQL truth | Access module + RDS transaction |
| Public Cloudflare routes receive no cookie, PII, private preview, tenant response, or sensitive log | Routing allowlist + release policy |

## 3. Verified unit prices and product facts

### 3.1 Object storage and transfer

| Meter | AWS S3 Standard `ap-east-1` | Cloudflare R2 Standard | Consequence |
| --- | ---: | ---: | --- |
| Storage | `$0.025/GB-month` first 50 TB | `$0.015/GB-month` | R2 storage is `$0.010/GB-month` lower before free tier |
| Write operations | `$0.005/1,000` PUT/COPY/POST/LIST | `$4.50/million` Class A | Almost identical at this workload (`$5.00` versus `$4.50` per million) |
| Read operations | `$0.004/10,000` GET/other | `$0.36/million` Class B | R2 is slightly cheaper (`$0.36` versus `$0.40` per million) |
| Internet egress | `$0.12/GB` first 10 TB in Hong Kong | Free when egressing directly from R2 | This is the large recurring difference |
| Monthly free tier | AWS DTO: first 100 GB aggregated across eligible services/Regions | 10 GB-month, 1M Class A, 10M Class B | AWS allowance is not S3-exclusive; R2 free tier applies only to Standard |
| Infrequent retrieval | S3 Standard: none; other classes vary | Standard: none; IA: `$0.01/GB` plus 30-day minimum | This comparison uses Standard on both sides |

Sources: [R2 pricing](https://developers.cloudflare.com/r2/pricing/), [AWS S3 `ap-east-1` price list](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonS3/current/ap-east-1/index.json), [AWS Data Transfer `ap-east-1` price list](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AWSDataTransfer/current/ap-east-1/index.json), and [AWS data transfer free allowance](https://aws.amazon.com/ec2/pricing/on-demand/#Data_Transfer).

R2's free egress starts **after bytes are in R2**. Cloudflare explicitly warns that Super Slurper/Sippy source buckets may charge as objects are copied. An S3-to-R2 copy therefore incurs AWS GET requests and AWS Data Transfer OUT; a cache miss that fetches from S3 has the same origin-side issue.

### 3.2 Workers, D1, and Hyperdrive

| Product | Public pricing / limit | Relevance |
| --- | --- | --- |
| Workers Paid | `$5/month`; 10M requests and 30M CPU-ms included; then `$0.30/M requests` and `$0.02/M CPU-ms`; direct Workers bandwidth has no additional egress charge | Add at least `$5` if storage access needs Worker code; cache hits through Workers Caching still count as Worker requests |
| D1 Paid | 25B rows read/month and 50M rows written/month included; overages `$0.001/M reads`, `$1/M writes`; first 5 GB included then `$0.75/GB-month`; no D1 transfer charge | Row/storage economics are irrelevant until the PostgreSQL contract is shown portable, which it is not |
| D1 hard topology | SQLite semantics; 10 GB maximum per database; each database is single-threaded and executes queries one at a time; 30-day Paid Time Travel | Not an in-place substitute for RDS PostgreSQL HA, extensions, roles/RLS, pooling, migrations, and restore procedures |
| Hyperdrive | Included in Workers plans; Paid query count unlimited; Free capped at 100,000 queries/day | Retains the external database; Workers and AWS-origin transfer/latency/security remain separate meters and risks |

Sources: [Workers pricing](https://developers.cloudflare.com/workers/platform/pricing/), [D1 overview](https://developers.cloudflare.com/d1/), [D1 pricing](https://developers.cloudflare.com/d1/platform/pricing/), [D1 limits](https://developers.cloudflare.com/d1/platform/limits/), [D1 Time Travel](https://developers.cloudflare.com/d1/reference/time-travel/), [Hyperdrive pricing](https://developers.cloudflare.com/hyperdrive/platform/pricing/), and [Hyperdrive PostgreSQL connection](https://developers.cloudflare.com/hyperdrive/examples/connect-to-postgres/).

## 4. Common workload calculation

### 4.1 Assumptions

- Logical retained objects at the start of the month: 50 GB, 500 GB, and 2,500 GB.
- New immutable versions during the month: 10% of starting logical bytes.
- Internet downloads: 20% of starting logical bytes.
- New bytes arrive evenly through the month, so average billed current-month storage is `logical GB * 1.05`; closing storage is `logical GB * 1.10`. Every later retained month adds another 10% unless lifecycle/retention deletes it.
- Average object/version size is 5 MB solely to estimate request counts: monthly PUTs are `10% bytes / 5 MB`; monthly GETs are `20% bytes / 5 MB`.
- One GET transfers one complete object. Range retries, HEAD/LIST, multipart operations, copies, cache revalidation, scanner reads, and failures are excluded and must be measured.
- Cloudflare rounds R2 usage up to the next billing unit. The table applies R2's 10 GB Standard free storage and operation allowances.
- AWS 100 GB free DTO is shown twice: fully available to this workload, or already consumed elsewhere. It is a shared account-level sensitivity, not a promise.

### 4.2 Monthly storage/download subtotal

| Profile | Pilot | Growth | Scale |
| --- | ---: | ---: | ---: |
| Starting logical bytes | 50 GB | 500 GB | 2,500 GB |
| New versions | 5 GB / 1,000 PUT | 50 GB / 10,000 PUT | 250 GB / 50,000 PUT |
| Downloads | 10 GB / 2,000 GET | 100 GB / 20,000 GET | 500 GB / 100,000 GET |
| Average billed storage | 52.5 GB | 525 GB | 2,625 GB |
| **Pure S3, AWS free DTO unavailable** | **`$2.52`** | **`$25.18`** | **`$125.92`** |
| Pure S3, full 100 GB free DTO available | `$1.32` | `$13.18` | `$113.92` |
| **R2 storage-only, direct S3 API** | **`$0.65`** | **`$7.73`** | **`$39.23`** |
| R2 through a Paid Worker | from `$5.65` | from `$12.73` | from `$44.23` |

Calculation details:

```text
S3 = average_GB * 0.025
    + PUT_count / 1,000 * 0.005
    + GET_count / 10,000 * 0.004
    + billable_egress_GB * 0.12

R2 direct = max(0, ceil(average_GB) - 10) * 0.015
          + max(0, ceil(Class_A / 1M) - 1) * 4.50
          + max(0, ceil(Class_B / 1M) - 10) * 0.36
```

At these object counts, all R2 operations remain inside the Standard free tier. That is **not** true for high-frequency thumbnails, HEADs, Range requests, scanner reads, or small objects: R2's own example shows 290M billable Class B reads costing `$104.40/month` even with only about 10 GB stored.

### 4.3 First S3-to-R2 migration charge

| One-time copy | Pilot 50 GB | Growth 500 GB | Scale 2.5 TB |
| --- | ---: | ---: | ---: |
| AWS DTO if free allowance unavailable | `$6.00` | `$60.00` | `$300.00` |
| AWS DTO if entire 100 GB allowance is available to migration | `$0` | `$48.00` | `$288.00` |
| S3 GET requests at 5 MB/object | `$0.004` | `$0.040` | `$0.200` |
| R2 Class A writes | `$0` under 1M free | `$0` under 1M free | `$0` under 1M free |

This is optimistic. It excludes multipart amplification, retries, listing/verification, checksums, delta copy, dual-write, retained noncurrent S3 versions, and the period when both stores are billed. If source objects are in an infrequent/archive class, retrieval and restore charges/delay must be added.

## 5. Four architecture choices

### 5.1 Pure S3 Hong Kong: approved baseline

```text
Browser -> short-lived intent -> private S3 ap-east-1
                                -> SQS/scanner ap-east-1
                                -> clean state in RDS ap-east-1
```

This is not always cheapest per GB, but it preserves the accepted Hong Kong boundary, S3 Versioning, KMS integration, same-region event destinations, Object Lock option, IAM/SCP controls, CloudTrail, and the existing recovery/scanner contracts. It has one cloud control plane and no migration.

### 5.2 R2 storage-only: economical only for approved non-sensitive objects

R2 can be accessed directly through its S3-compatible API, so a Worker is not intrinsically required. It gives the strongest narrow price result: approximately `$0.65`, `$7.73`, and `$39.23` per month for the three profiles in this model.

It is acceptable only for a separately classified public object domain whose authoritative bytes may reside outside Hong Kong. Required controls include opaque keys, explicit cache/retention rules, checksums, inventory reconciliation, deletion proof, least-privilege R2 tokens, and a tested export path.

It is not a drop-in S3 replacement. Cloudflare's current compatibility table marks S3 bucket notification configuration, bucket versioning, Object Lock, bucket policy, inventory, replication, logging, AWS KMS headers, tagging, and multiple S3 request fields as unimplemented or unsupported. The endpoint also uses region `auto`; S3 SDK compatibility does not imply semantic equivalence. [R2 S3 API compatibility](https://developers.cloudflare.com/r2/api/s3/api/)

### 5.3 S3 + Cloudflare public cache: useful, but only for public content

For anonymous cacheable objects retained in S3, Cloudflare can reduce recurring S3 origin reads and AWS DTO. With cache-hit ratio `h`:

```text
AWS origin download GB ~= monthly download GB * (1 - h) + revalidation/churn
gross DTO saving       ~= monthly download GB * h * $0.12
```

At an idealized 90% hit rate and with no AWS free DTO, the S3 subtotal becomes about `$1.44`, `$14.38`, and `$71.92` for Pilot/Growth/Scale, before Cloudflare plan/Worker/enterprise charges. Cold fills, expiry, purge, query strings, Range requests, non-cacheable responses, and multi-POP churn lower the effective hit rate.

For private documents this path is rejected: signed/user-specific responses often should not be shared-cacheable, Cloudflare would terminate/process request metadata, and a cache is neither a version store nor a backup. A Worker added only to implement cache/authorization introduces the `$5` minimum and a second authorization/runtime surface.

### 5.4 Sensitive documents in R2 across clouds: rejected

The apparent `$0.65/$7.73/$39.23` monthly subtotal is not a valid decision metric because it violates the current residency invariant:

- R2 `apac` is a **best-effort hint**, not a placement guarantee. Available guaranteed R2 jurisdictions are EU and FedRAMP; Hong Kong is absent. [R2 data location](https://developers.cloudflare.com/r2/reference/data-location/)
- Regional Services can keep TLS termination and Worker execution in Hong Kong for a configured hostname, but Worker code/secrets remain globally deployed, outgoing subrequests are not covered, and Queue/Cron triggers are not covered. [Workers localization](https://developers.cloudflare.com/data-localization/how-to/workers/)
- Customer Metadata Boundary is available for the US or EU, not Hong Kong. A Hong Kong edge location therefore does not prove Hong Kong-only logs, analytics, metadata, support, state, or storage. [Cloudflare region support](https://developers.cloudflare.com/data-localization/region-support/)

The cross-cloud design also adds two provider IAM/token systems, public/private connectivity decisions, split audit evidence, correlated outage handling, dual billing, and a harder rollback. It may save tens of dollars while expanding the security and compliance surface materially.

## 6. D1 and Hyperdrive disposition

### D1 cannot replace RDS PostgreSQL

The system's current database contract includes PostgreSQL migrations/extensions, transaction-scoped authorization context, roles and forced RLS, connection/failover behavior, and Multi-AZ/PITR recovery. D1 instead exposes SQLite semantics, limits each database to 10 GB, processes each database's queries single-threadedly, caps individual query duration at 30 seconds, and offers a 30-day Paid Time Travel window.

A per-tenant D1 sharding design would be a new database architecture with cross-database reporting, migration fan-out, tenant placement, restore, audit, consistency, and operational semantics to design. Its egress price does not answer those requirements. Therefore:

**D1 is prohibited as the Release 1 system of record and is not part of the storage-saving recommendation.** It may be evaluated later only for non-authoritative, independently rebuildable public edge data with a separate design decision.

### Hyperdrive does not remove RDS or cross-cloud risk

Hyperdrive can pool and cache PostgreSQL connections and its Paid plan does not meter query count separately, but the query still terminates at an external PostgreSQL database. In an AWS RDS + Workers design:

- cache misses and all writes cross the Cloudflare/AWS boundary;
- result bytes, query parameters, credentials/secrets, logs, and failure traces require residency review;
- Workers compute pricing still applies;
- database firewall/private connectivity, TLS, credential rotation, cache invalidation, transaction/session semantics, and fail-closed authorization require new evidence;
- Cloudflare's own recommended private Workers VPC path is Beta as documented in the Hyperdrive navigation.

Hyperdrive is therefore not a cost justification for moving authenticated application execution to Workers. Keep the application close to RDS inside AWS unless a measured latency/connection problem and approved data boundary justify a PoC.

## 7. API, version, event, scanning, and recovery differences

| Capability | S3 Hong Kong baseline | R2 implication |
| --- | --- | --- |
| API identity | AWS regional endpoint, SigV4, IAM/bucket policy, `ap-east-1` | S3-compatible endpoint, region `auto`, R2 token model; compatibility matrix has omissions |
| Version protection | Native S3 Versioning and version IDs | `PutBucketVersioning` is unimplemented in R2 S3 API; application must write immutable unique keys and own the version catalogue |
| Retention/legal hold | S3 Object Lock available where policy permits | Object Lock headers/configuration are unsupported in compatibility table |
| Encryption | SSE-KMS customer-managed regional key and Bucket Key supported | AWS KMS headers are unsupported; encryption/key-custody contract differs and needs separate approval |
| Events | S3 notifications to same-region SQS/SNS/Lambda/EventBridge; established duplicate/out-of-order handling | S3 notification configuration APIs are unimplemented; R2 Event Notifications use a Cloudflare-specific Queue/event path, so scanner wiring and retry/DLQ evidence must be redesigned |
| Inventory/audit | S3 Inventory, CloudTrail data events, server access logs and AWS Config can compose with the current control plane | S3 inventory/logging/policy APIs are not equivalent; Cloudflare logs/analytics and metadata geography require separate review |
| Malware scanning | Same-region S3 -> SQS -> scanner; bytes stay quarantined until clean | No storage provider supplies the business scan invariant; cross-cloud scanner reads add transfer/path/failure states and Queue/Cron regionalization gaps |
| Restore | Restore object version plus DB active-pointer rollback; S3 version and RDS audit remain independently testable | Without native S3 versioning semantics, recovery depends on immutable-key discipline, an external authoritative catalogue, backup/export, and tested reconciliation |

R2 documents strong consistency for object operations, but consistency is not version recovery, legal hold, an audit trail, or an exactly-once scanner event. [R2 consistency](https://developers.cloudflare.com/r2/reference/consistency/)

## 8. Break-even analysis

### R2 storage-only versus pure S3

Ignoring free tiers and request differences, with downloads equal to 20% of logical storage:

```text
monthly R2 saving per logical GB
  ~= ($0.025 - $0.015) storage + 20% * $0.12 egress
  = $0.034 / logical GB-month

one-time S3 migration DTO ~= $0.12 / GB
simple payback ~= 0.12 / 0.034 = 3.53 months
```

Using the detailed table and no AWS free DTO, estimated payback is about **3.2 / 3.4 / 3.5 months** for Pilot/Growth/Scale. If the shared AWS 100 GB allowance is genuinely available, download egress is zero at Pilot and Growth; the storage-only saving is smaller. Under the optimistic allocation used above, payback is effectively immediate for Pilot, about **8.8 months** for Growth, and about **3.9 months** for Scale. Actual migration month must allocate the one shared 100 GB allowance across migration and all other AWS traffic, so it cannot be counted twice.

These paybacks are financially interesting only for data legally and operationally eligible for R2. They exclude migration engineering, dual-run, restore redesign, compliance review, incident response, and enterprise plans. For sensitive documents the compliance blocker is terminal regardless of payback.

### S3 plus Cloudflare cache

If Cloudflare introduces an incremental monthly cost `C`, pure DTO break-even is:

```text
logical_GB * 20% download * hit_rate * $0.12 > C
logical_GB > C / (0.024 * hit_rate)
```

If a `$5` Paid Worker is the only incremental cost and hit rate is 90%, break-even is about **232 GB of logical public data**. If ordinary CDN caching is already included in an existing plan, any stable non-zero hit ratio reduces origin DTO, but the absolute saving at Pilot may be too small to justify operational complexity. Enterprise localization/support contract cost must replace `C` before approval.

## 9. Selection matrix

| Option | Recurring object economics | Hong Kong sensitive-data fit | Migration/operations | Decision |
| --- | --- | --- | --- | --- |
| Pure S3 `ap-east-1` | Highest DTO; modest absolute storage cost | Strongest current evidence | Existing IaC, IAM, scan, audit, version and restore model | **Use for sensitive documents** |
| R2 direct storage | Lowest narrow subtotal; no direct egress | No HK jurisdiction guarantee | Requires new token, audit, version, event and restore controls | **Allow only approved public/non-sensitive objects** |
| S3 + Cloudflare cache | Saves origin DTO in proportion to hit ratio | Public plane only | Cache correctness/purge/origin-failure operations; possible plan/Worker cost | **Optional public-plane optimization after measurement** |
| R2 sensitive cross-cloud | Looks cheapest in table | Fails current residency invariant | Highest split-control and rollback burden | **Prohibit** |
| D1 replacing RDS | Cheap row/storage list price | Location and metadata gaps remain | SQLite/single-threaded/10 GB architecture rewrite | **Prohibit for system of record** |
| Workers + Hyperdrive + RDS | Low compute floor, retains RDS | Worker/subrequest/metadata boundary incomplete | Cross-cloud DB path, new runtime and auth surface | **Do not adopt without a measured PoC trigger** |

## 10. Permitted and prohibited uses

### Permitted after classification and review

- Cloudflare DNS/WAF/CDN for a hostname serving only public, anonymous, non-personalized content.
- R2 for immutable public assets or approved public crawler releases where non-Hong-Kong storage is contractually acceptable.
- S3-origin public caching after measuring cache-hit ratio, origin DTO, purge correctness, stale exposure, and incremental Cloudflare plan cost.
- A reversible public-plane pilot with a route allowlist, no cookies, no PII, no private headers, no authenticated APIs, redacted logs, cost caps, and an origin-bypass rollback.

### Prohibited under the current decision

- R2 for student, guardian, case, identity, uploaded document, private preview, scan artifact, audit, log, backup, or support payloads.
- D1 as a replacement for RDS PostgreSQL or as authorization/session truth.
- Proxying authenticated ERP traffic through Workers/Hyperdrive solely to reduce egress or latency.
- Caching signed private downloads, authenticated HTML/API responses, or tenant-personalized data on the public plane.
- Claiming `apac`, a Hong Kong PoP, Regional Services, S3 compatibility, or free egress proves Hong Kong residency, semantic equivalence, backup, or recovery.

## 11. Approval gate and evidence required

A public-plane Cloudflare pilot may proceed only after all of the following are explicit:

1. Data-class and route allowlist showing no PII, cookie, authorization header, tenant content, private preview, or sensitive log reaches Cloudflare.
2. Dated Cloudflare quote/plan, cache/file limits, support/SLA, Regional Services scope if purchased, and complete subprocessor/metadata review.
3. Measured object count/size distribution, HEAD/GET/Range rate, cache-hit ratio, purge/revalidation rate, origin bytes, and scanner/retry amplification.
4. Hash/count inventory, immutable release identifier, deletion/retention contract, and restore/export drill for any R2 object domain.
5. A 30-day bill comparison in which savings exceed engineering/operational overhead and no quality/residency gate is weakened.
6. Rollback that removes the Cloudflare route without changing the authoritative AWS sensitive-data plane.

Stopping conditions: any private-data observation, non-Hong-Kong-sensitive processing, stale/private cache exposure, irreconcilable inventory, unsupported recovery requirement, or cost above the approved cap ends the pilot in `needs_human`; it must not silently expand scope.

## 12. Bottom line

**AWS + Cloudflare is more suitable than pure AWS only as a deliberately separated public-plane optimization, not as the production sensitive-data architecture.** R2's pricing can cut the narrow object subtotal by roughly 50-70% in the three modeled profiles when AWS's shared free DTO is unavailable, with a simple migration payback around 3-4 months. But the absolute saving is `$1.87`, `$17.46`, and `$86.69` per month in this model, before migration labor and new controls. That does not justify giving private documents a non-guaranteed Hong Kong location or rewriting the PostgreSQL/runtime boundary.

The pragmatic design remains:

```text
Public, anonymous, approved content
  -> optional Cloudflare CDN / R2 after measurement

Authenticated and sensitive core
  -> AWS ap-east-1 ECS + RDS PostgreSQL + S3/KMS + SQS/scanner/audit
```

Reconsider R2 for sensitive data only if Cloudflare offers a contractual Hong Kong R2 jurisdiction and Hong Kong metadata boundary, Regional Services covers all subrequests/triggers/state/logs/support paths, and the replacement passes version/event/scanning/restore and total-cost evidence gates.

