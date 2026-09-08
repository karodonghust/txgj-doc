# Production Platform, Database, and Multitenancy Synthesis

| Field | Value |
| --- | --- |
| Research date | 2026-08-11 (Asia/Hong_Kong) |
| Status | Advisory research; not a decision, budget approval, procurement, migration, deployment, or release authorization |
| Decision authority reviewed | `docs/PRD_IMPLEMENTATION_DECISIONS.md`, `docs/TECHNICAL_DECISION_PRODUCTION_PLATFORM.md`, `docs/PRD_PHASE_IMPLEMENTATION_PLAN.md` |
| Evidence | Repository source and first-party vendor/PostgreSQL/Next.js documentation only |
| Price basis | USD/month, on-demand planning ranges before tax; not a quote |

## 1. Executive recommendation

The best current production fit is still a **single-cloud AWS Hong Kong application and data plane**:

```text
Optional public-only plane after no-PII review
  -> Cloudflare or Vercel static/public content

Authenticated production plane: AWS ap-east-1
  -> ALB + WAF
  -> ECS Fargate, at least 2 Next.js web/BFF tasks across 2 AZs
  -> separate ECS tasks/jobs for scanner, OCR/index, crawler and future AI tools
  -> private RDS PostgreSQL Multi-AZ
  -> private S3 + KMS + SQS/DLQ
  -> Hong Kong logs/audit/backups with explicit retention
```

Use **RDS PostgreSQL Multi-AZ DB instance**, not Aurora, for Release 1. Use **shared database/shared schema with server authorization plus forced PostgreSQL RLS** as the standard tenancy tier. Preserve a future `TenantPlacement` mechanism for a dedicated Hong Kong database/cluster only when contractual or measured resource-isolation evidence justifies it.

Do not migrate the authenticated core to Azure, GCP, Vercel, or Cloudflare now. Azure Container Apps plus Azure PostgreSQL is the strongest alternative PoC, and GCP Cloud Run/Cloud SQL is a credible shadow candidate, but neither provides enough business benefit to offset the current identity, storage, IaC, recovery, and operational migration surface. Cloudflare and Vercel remain public-plane candidates only under the present Hong Kong-sensitive-data boundary.

The recommendation is conditional. The repository is **not production-complete and not second-tenant ready** merely because Terraform and `organization_id` columns exist. Before approval it must close the RLS, global-data, restore, production topology, OCR, cost, and open business-semantics gaps in Sections 7–10.

## 2. What should remain and what should change

### Retain

- `DEC-018`–`DEC-024`: Hong Kong sensitive-data boundary; RDS, Cognito, sensitive runtime, and S3 in `ap-east-1`.
- Complete authenticated Next.js modular monolith on ECS Fargate behind ALB.
- RDS-authoritative membership, role, case scope, capability, expiry, and session-version checks. Cognito authenticates a user; it does not own case authorization.
- Private, versioned S3 objects; DB metadata and authorization; short-lived, single-purpose upload/download intents; quarantine until a typed clean-scan receipt.
- Shared schema plus defense-in-depth RLS and organization context across DB, S3, cache, jobs, search/vector, export, and audit.
- Crawler and AI workers isolated from request-path compute; immutable source/version and human publication gates.

### Amend before production approval

1. **Separate accepted platform baseline from pending topology.** The production TDR currently presents a `recommended`/pending state while the underlying AWS/Cognito/RDS/S3 decisions are accepted. Mark the baseline accepted and list only ECS/ALB production topology, public-plane disposition, worker placement, and cost ceiling as pending.
2. **Treat instance size as a capacity hypothesis.** `db.t4g.small`, task CPU/memory, and minimum/maximum task counts are dated starting assumptions, not permanent architectural decisions.
3. **Strengthen `DEC-053`.** Require table classification, `ENABLE` and `FORCE ROW LEVEL SECURITY`, a non-owner `NOBYPASSRLS` application role, policy/owner/grant catalogue fingerprints, and real pooled-connection reuse tests.
4. **Split global facts from tenant data.** Global school facts/revisions must use explicit global relations/namespaces. Tenant overlays, corrections, notes, shortlist/case links, knowledge, and visibility remain `organization_id NOT NULL`. Do not use null/zero sentinel tenants or duplicate the global base per tenant.
5. **Add a Tenant Placement and Selective Recovery decision.** Shared Hong Kong RDS is the default placement; dedicated placement requires measured or contractual triggers. Define tenant export, isolated restore, reconciliation, termination, and rollback authority.
6. **Keep `DEC-060` open.** Subscription, suspension, support grants, retention, outage behavior, tenant-to-global knowledge, export, and termination are business/legal decisions. Engineering must not invent them.
7. **Add an OCR gate.** Amazon Textract has no `ap-east-1` endpoint in the official endpoint list as checked on 2026-08-11. Hong Kong-only OCR therefore needs a self-hosted worker or separately approved external/cross-region processing.
8. **Reduce avoidable provider leakage.** Keep AWS placement checks fail-closed, but move literal provider names, SigV4 details, and provider-specific error vocabulary behind effect adapters and an approved placement policy. This improves exit posture without pretending the current platform is provider-neutral.

## 3. Common workload assumptions

These inputs make alternatives comparable. They are replaceable planning assumptions, not forecasts, limits, or SLAs.

| Profile | Tenants | Users | Active / retained cases | Logical documents | New/version bytes monthly | Internet download sensitivity |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Pilot | 1 | 10 | 10 / 100 | up to 50 GB | 10% | 20% |
| Growth | 10 | 100 | 100 / 1,000 | about 500 GB | 10% | 20% |
| Scale | 50 | 500 | 500 / 5,000 | about 2.5 TB | 10% | 20% |

The following must be measured independently rather than inferred from case count: HTTP rate and concurrency, structured DB size/TPS, vector volume and recall, log bytes, OCR pages, scanner concurrency, backup growth, and tenant skew. One abusive tenant can cost more than 50 light tenants.

## 4. Production runtime comparison

| Option | Product fit | HK boundary | Operational burden | Migration/lock-in | Recommendation |
| --- | --- | --- | --- | --- | --- |
| AWS ECS Fargate + ALB | Native container fit for Next.js web/BFF and workers | Strongest fit with current `ap-east-1` RDS/S3/Cognito/IaC | No guest OS; still requires multi-AZ, autoscaling, alarms, drain and rollback work | Existing AWS coupling; OCI image remains portable | **Select** |
| AWS EC2/ASG + ALB | Full control; useful for native scanner/OCR or high steady utilization | Same regional boundary | Team owns AMI, OS patching, capacity, drain and host incidents | Lower compute may be offset by labor | Revisit after measured 25–30% total saving or Fargate blocker |
| AWS Lambda | Good for short idempotent event consumers | Available in HK, service-specific verification still required | Strong scaling/rollback; timeout, native binary and DB connection limits | Function/event wiring specific | Use only for bounded workers after tests |
| AWS App Runner | Managed container platform | Official endpoint list omits `ap-east-1` | Irrelevant while region absent | Container portable | Reject |
| Azure Container Apps | Best alternative container/apps/jobs topology | Compute/data can be East Asia (HK); Entra HK-only identity not proven | Lower than AKS; private endpoint and network charges matter | 10–20 engineer-weeks plus review | PoC only when a formal trigger occurs |
| Azure App Service | Viable web container | Similar Azure data boundary; worker platform remains separate | Fixed plan and two compute models | 10–22 engineer-weeks | Fallback Azure option |
| Azure AKS | Technically complete | Can be private in East Asia | Highest platform/on-call burden | 18–36 engineer-weeks | Reject at current scale |
| GCP Cloud Run | Credible container/jobs in `asia-east2` HK | Runtime/SQL/storage have positive evidence; identity gap remains | Managed, but creates a new control plane | 12–24 engineer-weeks | Shadow comparator only |
| Vercel | Lowest Next.js adaptation | `hkg1` execution does not prove request/log/support/failover HK-only processing | Excellent developer platform; Enterprise controls/procurement | 8–16 engineer-weeks for full core | Public/static plane only |
| Cloudflare Workers/R2 | Cheap global public edge | R2 has no HK jurisdiction; HK regional controls exclude important state/subrequests/triggers | Low edge burden; different `workerd` runtime | 16–30 engineer-weeks and no compliant core target | Public/static plane only |

The existing Terraform service has `desired_count = 1`, a small health task, and a health-only listener slice. It proves staging infrastructure intent, not two-AZ production availability, full application routing, autoscaling, or capacity.

## 5. Database decision

### Recommended database

Use private **RDS PostgreSQL Multi-AZ DB instance** in `ap-east-1` for Release 1.

| Criterion | RDS PostgreSQL Multi-AZ | Aurora PostgreSQL HA | Azure PostgreSQL Flexible Server | Neon |
| --- | --- | --- | --- | --- |
| Current compatibility | Existing target, IaC, IAM, migrations and tests | PostgreSQL-compatible but needs extension/query/RLS/failover validation | PostgreSQL target but IAM/network/backup/monitoring migration required | Application adapter exists, but no official HK region |
| HA/read behavior | Synchronous standby; standby does not serve reads; AWS says typical failover 60–120s | Existing replica can promote faster and serve reads | HA modes available, but current East Asia zone-HA availability/capacity must be reconfirmed before any plan | Fails current residency gate |
| Cost floor | Lowest current managed HA fit; accepted dated DB-only baseline about $87/month | Small useful HA topology previously estimated about $163–185/month | Exact East Asia calculator export unavailable in this research; total is unknown | Not applicable under HK gate |
| Best trigger | Current pilot/growth compatibility | Sub-60s RTO, sustained read replica need, operational I/O burden, or cost parity within 15% | Formal Azure superseding decision | Region/residency evidence changes |

Do not adopt Aurora merely for pgvector: RDS PostgreSQL supports the extension. Reconsider Aurora only with measured read scaling, failover, storage/I/O, connection, compatibility, and total-cost evidence.

Do not add RDS Proxy by default. PostgreSQL `SET`/`set_config` can pin Proxy sessions, reducing multiplexing. The application correctly uses transaction-local `set_config(..., true)` for tenant and actor context; test real connection reuse, pool exhaustion, failover recovery, and Proxy pinning before paying the additional cost.

### Standard tenancy topology

Use shared DB/shared schema through the measured 50-tenant profile. Reject schema-per-tenant as the default: it retains shared-resource contention while multiplying migration/search-path/schema-drift risk. Database/cluster-per-tenant is an exception tier for contractual keys/retention/restore isolation, incompatible maintenance, or a measured noisy tenant.

```text
CustomerOrganization
  -> TenantPlacement(placement_id, class, region, database_target, policy_version)
       -> shared-hk-primary       # default
       -> dedicated-hk-*          # future approved exception
```

The browser never selects a database target. A trusted control-plane/repository path resolves placement after active-organization authorization. Cross-placement joins are forbidden; logical module schemas and event contracts remain compatible.

## 6. Monthly production cost model

### Planning ranges

| Profile | AWS recommended topology | Main drivers |
| --- | ---: | --- |
| Pilot | **$300–650/month** | 2 web tasks, ALB/WAF, small Multi-AZ RDS, private networking, S3/KMS/SQS, logs, scanner duty cycle, support sensitivity |
| Growth | **$650–1,900/month** | web/worker utilization, vertical DB scaling, 500 GB documents, egress, OCR pages, logs/retention |
| Scale | **$1,600–6,400/month** | peak compute/TPS, DB/vector volume, 2.5 TB versions, egress, OCR, observability, backup and support |

These are budget envelopes, not quotes. AI model tokens and migration labor are outside them unless an exact line item is added. The previous roughly `$194.56–199.42/month` AWS figure is only a narrow core subtotal for two web tasks, ALB, small RDS, and a small S3 scenario; it excludes several required production categories.

### Cost equation

```text
Production monthly cost =
  web + workers + load balancer/WAF
  + DB HA compute/storage/I/O/backup/proxy-if-proven
  + object current/noncurrent/quarantine storage
  + object requests/download/egress
  + queue/DLQ/scanner/OCR/index
  + private endpoints or approved NAT
  + logs/metrics/audit/alerts
  + keys/secrets/DNS/registry
  + support/tax

Total ownership cost =
  production monthly cost
  + engineering/on-call/security/privacy/restore labor
  + migration/dual-run/rollback amortization
```

Azure does not receive a numeric total in this research because the harness could not obtain a dated East Asia SKU calculator export. Treating scale-to-zero compute or free Entra MAU as an end-to-end price would be false precision. A competing quote must use the same workload and include private endpoints, gateway/WAF, database HA, backup, objects/download, queue, jobs, logs, OCR, support, tax, and migration labor.

## 7. Migration cost and risk

### Current coupling surface

Repository inspection found AWS/Cognito/S3/RDS references across 117 source/test/infrastructure files and about 1,218 Terraform lines. This is a navigation signal, not a rewritten-line estimate. High-impact coupling includes:

- RDS `ap-east-1` hostname enforcement and `rds_iam` grants;
- Cognito provider subjects, cookies, revoke flow, sessions, MFA and error contracts;
- AWS SigV4 document intents, S3 versions, KMS, SQS/DLQ, scanner states and literal region checks;
- AWS-shaped Terraform, release, restore, logs, alarms, IAM and evidence;
- tests that intentionally prove the approved Hong Kong/provider boundary.

### Rough-order engineering effort

| Target | Effort band | Dominant uncertainty |
| --- | ---: | --- |
| Azure Container Apps full core | 10–20 engineer-weeks | Identity geography, private network, storage events, DB auth/restore, dual-run |
| Azure App Service full core | 10–22 engineer-weeks | Same data/identity work plus separate worker platform |
| GCP Cloud Run full core | 12–24 engineer-weeks | Identity, complete replacement of AWS operations/evidence |
| Vercel full core | 8–16 engineer-weeks | Residency/Enterprise contract, storage/queue/DB/identity rather than Next.js |
| Cloudflare full core | 16–30 engineer-weeks | Runtime rewrite plus unresolved residency target |
| AKS | 18–36 engineer-weeks | Migration plus permanent Kubernetes platform ownership |

Ranges have at least ±100% uncertainty before PoC/discovery and exclude procurement, customer downtime, training, penetration tests, migration egress, duplicate production environments, and delivery opportunity cost.

### Migration work packages

1. Build one pinned container/image and inventory provider-specific behavior.
2. Implement target IaC, region policy, network, IAM, secrets, logs and support controls.
3. Decide IdP retention or migration; map provider subjects; handle MFA/password re-enrollment, session invalidation and revoke evidence.
4. Validate PostgreSQL version/extensions, roles/owners/RLS/policies, query plans, logical copy/replication, checksums and rollback window.
5. Inventory/copy immutable object versions with hashes, scan states, metadata, delta reconciliation and orphan cleanup plan.
6. Drain/replay queues and jobs with idempotency keys, DLQ ownership, cancellation and no duplicate external effects.
7. Rebuild audit/monitoring/backup/PITR/selective restore and incident runbooks.
8. Run synthetic and empty-tenant rehearsals, then a human-approved canary with objective rollback thresholds.

Rollback after new writes is not a DNS switch. Database, object, queue, identity and audit divergence make it a recovery workflow. Avoid unbounded dual writes; use a bounded compatibility/delta window and a declared point after which rollback becomes restore/forward repair.

## 8. Multitenancy decisions that must be explicit

| Decision | Required semantics | Enforcement owner |
| --- | --- | --- |
| Organization identity | Stable UUID, legal identity, duplicate/domain rules, merge prohibition/process | TenantAccess + DB constraints |
| Membership lifecycle | Invite/accept/disable/revoke, last owner, ownership transfer, multi-org switching, session invalidation | TenantAccess + Identity |
| Authorization | IdP authenticates only; RDS owns membership, role, case/capability/expiry; support is time-bound | Server policy + repository transaction + RLS |
| Tenant row locality | `organization_id NOT NULL`, composite FK/unique, no cross-tenant parent/child reference | Owning module + DB |
| RLS contract | Table catalogue, canonical policies, FORCE RLS, non-owner NOBYPASSRLS app role, missing context denies | DB migration/evidence owner |
| Global vs tenant facts | Separate global revisions from tenant extension; version, compatibility, dispute, withdrawal and orphan rules | SchoolIntelligence + Knowledge |
| Knowledge/RAG | Approved immutable versions, author cannot self-approve, tenant private is never automatically global, citation/purge/rebuild | Knowledge + ReportingAI |
| Placement | Shared default, dedicated triggers/cost owner, migration, rollback, no cross-placement join | Control Plane + Operations |
| Noisy tenant | Request/upload/export/OCR/vector/job quotas; fair queue; security/exit operations remain available | TenantAccess + Operations |
| Subscription | trial/active/past_due/suspend/terminate behavior and exact capability effects | Product/Legal/Finance, enforced server-side |
| Support access | Requester/approver/scope/expiry/customer notice/post-review | Platform Operations + Audit |
| Recovery/export/exit | Who may restore, overwrite/merge rule, tenant export format, retention, termination/purge proof | Data owner + Operations + Legal |
| Region outage | Degrade/stop/manual recovery versus approved secondary geography | Privacy/Security/Ops |
| Telemetry | Organization-labelled but PII-allowlisted logs; audit retention and fail-closed mutation behavior | AuditOperations |
| AI/provider | Exact endpoint/model/location, DPA, logs/retention/support, tool scopes and cost budgets | ReportingAI + Privacy/Security |

## 9. Deterministic gates before production and tenant 2

### Production topology gate

- At least two web tasks across two AZs; forced task failure, drain, autoscaling and rollback evidence.
- Immutable build identity, consistent Server Action key, multi-instance cache/tag behavior and mixed-version deployment tests.
- Exact HK resource/data-flow manifest for DB, object, queue, keys, logs, backup, email/SMS, support, OCR and model calls.
- Network choice costed: endpoints where available; NAT only for an approved dependency; no accidental dual path.
- DB failover, pool recovery, backup/PITR and restore drills with approved RPO/RTO.
- Quarantine/scan duplicate, timeout, poison/DLQ, retry and reconciliation tests.
- Dated calculator export and alarm/budget thresholds; measured pilot replaces planning assumptions.

### Second-tenant gate

1. Close the identity, membership, subscription, support, retention, export/exit, outage and tenant-to-global items in `DEC-060`.
2. Classify every table as global, tenant-owned, derived-tenant or privileged-control; unclassified tables fail the migration gate.
3. Prove every tenant table has tenant `NOT NULL`, composite locality, canonical policy, `ENABLE` + `FORCE RLS`, non-owner app role and `NOBYPASSRLS`.
4. Run real PostgreSQL pool tests for absent context, commit/rollback reuse, cross-tenant ID guessing, concurrent membership/grant revoke and failover.
5. Pass cross-tenant UI/API/search/vector/export/S3/cache/job/support negative tests. Any leak is a blocker.
6. Split global school revisions from tenant extension and prove tenant knowledge cannot publish globally without explicit rights and review.
7. Restore the shared database to an isolated HK target, selectively recover one tenant through owning modules, reconcile DB/S3/audit/index, and produce zero unexplained differences.
8. Load-test pilot/growth/scale envelopes including skew/noisy tenant, 500 MB file lifecycle, queue backlog, vector recall and cost receipts.
9. Provide a minimal control plane or controlled runbook for onboarding, placement, entitlement/quota, support grant, incident, restore and termination.

## 10. Risk register

| Risk | Consequence | Required control |
| --- | --- | --- |
| App/table owner bypasses RLS | Cross-tenant disclosure despite policy | FORCE RLS, non-owner app role, catalogue fingerprint |
| Later table misses RLS | New feature silently leaks | Final-state unclassified-table failure gate |
| Pool session context leaks | Request B inherits tenant A | Transaction-local context and actual commit/rollback reuse tests |
| Global base duplicated per tenant | Divergent truth and unsafe promotion | Explicit global revision + tenant extension schema |
| Shared PITR treated as tenant restore | Other tenants overwritten/disclosed | Isolated full restore plus selective module-owned recovery |
| Noisy tenant exhausts DB/jobs | Shared outage and unpredictable cost | Per-tenant admission, fair queue, timeouts and attribution |
| RDS Proxy pins sessions | Paid proxy without multiplexing; saturation | Measure pinning or omit Proxy |
| OCR/model leaves HK | Regulatory/privacy breach | Self-host or exact separately approved data-flow |
| Single staging task called HA | False production-readiness claim | Two-AZ failure/drain evidence |
| Alternative chosen on headline price | Underbudget and migration failure | Same-workload TCO plus dual-run/rollback labor |
| IdP change assumed portable | Lockout, MFA reset, revocation gap | Subject map, enrollment plan, parallel validation and rollback cutoff |
| Object copy omits versions/scan state | Stale or unsafe document served | Immutable inventory/hash/version/state reconciliation |

## 11. Recommended decision sequence

1. Approve or reject the AWS production topology separately from the already accepted AWS service baseline.
2. Approve amended `DEC-053` enforcement language and the global-fact/tenant-extension ownership model.
3. Resolve `DEC-060` business/legal semantics; do not encode defaults before approval.
4. Create the Tenant Placement and Selective Recovery DEC.
5. Produce production Terraform/runtime evidence for two-AZ web, worker boundaries, network path, alarms and rollback.
6. Run database/RLS/pool/restore and cross-tenant negative evidence before tenant 2.
7. Replace planning inputs with measured pilot receipts and approve a budget ceiling.
8. Re-evaluate Aurora/EC2/alternative cloud only when a named trigger fires.

Re-evaluation triggers include: RDS failover cannot meet the approved RTO; sustained read/IO/connection demand crosses measured limits; a customer contract requires dedicated placement or another ecosystem; AWS price/service/support violates a budget/SLO; a competitor proves the full HK identity/data/support boundary and materially lower same-workload TCO; or annual review on 2027-08-11.

## 12. Research limitations

- No cloud account, live usage, production workload, secrets, Terraform plan/apply, migration, deployment, data movement, or purchase was accessed or performed.
- Public prices and feature availability are time-sensitive. Exact production approval requires dated calculator exports, plan/SKU identifiers, contracts and measured usage.
- The effort bands are engineering planning estimates, not vendor facts or delivery commitments.
- A Hong Kong region is necessary but not sufficient for end-to-end residency. Identity, support, logs, backup, CDN/TLS, email/SMS, OCR and model inference require service-specific review.
- The root instruction references `RTK.md`, but that file was not found in the workspace. This report follows the supplied root `AGENTS.md` and records the missing file as a harness gap rather than inventing its contents.

## 13. Supporting deep dives and primary sources

Local research artifacts:

- `docs/research/2026-08-11_AWS_PRODUCTION_PLATFORM_DEEP_DIVE.md`
- `docs/research/2026-08-11_ALTERNATIVE_PRODUCTION_PLATFORMS_DEEP_DIVE.md`
- `docs/research/2026-08-11_DATABASE_AND_MULTITENANCY_DEEP_DIVE.md`
- `docs/research/2026-08-11_PRODUCTION_HOSTING_AUTH_OPTIONS.md`
- `docs/research/aws-postgresql-hong-kong-assessment.md`
- `docs/research/aws-s3-hong-kong-assessment.md`

Selected first-party sources:

- [Next.js self-hosting](https://nextjs.org/docs/app/guides/self-hosting)
- [AWS Fargate regions](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate-Regions.html)
- [AWS App Runner endpoints](https://docs.aws.amazon.com/general/latest/gr/apprunner.html)
- [RDS Multi-AZ](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZSingleStandby.html)
- [RDS failover](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.Failover.html)
- [Aurora regions](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.RegionsAndAvailabilityZones.html)
- [RDS Proxy pinning](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy-pinning.html)
- [PostgreSQL row security](https://www.postgresql.org/docs/17/ddl-rowsecurity.html)
- [PostgreSQL configuration functions](https://www.postgresql.org/docs/current/functions-admin.html)
- [Azure regions](https://learn.microsoft.com/en-us/azure/reliability/regions-list)
- [Azure Container Apps networking](https://learn.microsoft.com/en-us/azure/container-apps/networking)
- [Azure PostgreSQL high availability](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-high-availability)
- [Microsoft Entra residency](https://learn.microsoft.com/en-us/entra/fundamentals/data-residency)
- [Cloud Run locations](https://cloud.google.com/run/docs/locations)
- [Cloud SQL PostgreSQL locations](https://cloud.google.com/sql/docs/postgres/locations)
- [Cloudflare R2 data location](https://developers.cloudflare.com/r2/reference/data-location/)
- [Vercel compliance](https://vercel.com/docs/security/compliance)

