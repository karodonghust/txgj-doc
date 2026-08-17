# Alternative Production Platforms Deep Dive

> Research date: 2026-08-11 (Asia/Hong_Kong)  
> Status: advisory research; no decision, procurement, provisioning, migration, deployment, data movement, or release authority  
> Current authority: `docs/TECHNICAL_DECISION_PRODUCTION_PLATFORM.md` and accepted `DEC-018`–`DEC-024` continue to govern  
> Evidence rule: only first-party Microsoft, Cloudflare, Vercel, Google Cloud, AWS, and Next.js documentation, official price pages, and local repository evidence are used. No third-party article is evidence.  
> Pricing basis: USD, on-demand, before tax, commitments, support, WAF, private networking, observability, backup beyond baseline, OCR/AI, migration labor, and incident/on-call labor unless a row says otherwise

## 1. Executive conclusion

The new official evidence does **not** justify superseding the current AWS Hong Kong decision.

Azure East Asia is a credible Hong Kong infrastructure region: Microsoft's region list identifies East Asia as Hong Kong, paired with Southeast Asia, and shows availability-zone support ([Azure regions list](https://learn.microsoft.com/en-us/azure/reliability/regions-list)). Azure Container Apps, Azure Database for PostgreSQL Flexible Server, Blob Storage, Log Analytics, and Container Apps Jobs can form a technically coherent Hong Kong application/data plane. The strongest Azure topology for Tianxing is:

```text
public plane (optional, public-only)
  -> Cloudflare or Vercel

authenticated plane
  -> Azure Front Door/Application Gateway decision gate
  -> Azure Container Apps, East Asia, custom VNet, public access disabled
       -> Next.js 16 web/BFF app
       -> Container Apps Jobs for crawler/scanner/batch workers
       -> PostgreSQL Flexible Server, private access, East Asia
       -> Blob/Queue, private endpoints, East Asia, LRS/ZRS only
       -> East Asia Log Analytics, no cross-region replication
```

This topology is not yet an acceptable replacement for AWS because the **identity plane is not proven Hong Kong-local**. Microsoft says Entra core-store scale units use two or more Azure regions, and Entra External ID's Go-Local add-on is available only in Australia and Japan, not Hong Kong ([Entra residency](https://learn.microsoft.com/en-us/entra/fundamentals/data-residency), [External ID pricing and Go-Local](https://learn.microsoft.com/en-us/entra/external-id/external-identities-pricing)). Selecting East Asia for compute does not repair that gap.

The other alternatives are narrower:

- **Azure Container Apps:** only Azure runtime worth a production PoC now. It preserves the container model, supports private networking and run-to-completion/event jobs, and has substantially lower platform burden than AKS.
- **Azure App Service:** viable for the Next.js web/BFF container, but workers still require another service and fixed plan capacity can be inefficient at pilot scale. It is a fallback if App Service operational maturity is valued above one unified apps/jobs environment.
- **Azure AKS:** technically capable, but unjustified for the pilot, 10-tenant, or 50-tenant workload unless a platform team, Kubernetes portability requirement, unsupported Container Apps feature, or measured workload constraint appears.
- **Cloudflare Workers/R2:** compelling public-plane and low-variable-cost platform, but unsuitable for authenticated/private Tianxing data. Hong Kong Regional Services does not cover Worker subrequests, Queues, or Cron; code/secrets remain globally deployed; Hong Kong metadata boundary is unavailable; R2 has no Hong Kong jurisdiction guarantee.
- **Vercel:** best zero-adapter Next.js experience and it has Hong Kong function/Blob regions, but Vercel's own compliance documentation permits transfers to the United States and other processing locations. Secure Compute and contractual support are Enterprise concerns. It remains a public-plane candidate, not a strict Hong Kong authenticated plane.
- **Google Cloud `asia-east2`:** the only additional hyperscaler candidate that merits a shadow comparison because Cloud Run, Cloud SQL, and Cloud Storage explicitly offer Hong Kong locations and Google's service terms cover selected-region at-rest storage. It still does not beat the current AWS decision: Identity Platform is absent from Google's configurable data-residency service list, and the repository migration surface is at least as broad as Azure.

The decision is therefore:

| Plane | Current recommendation |
| --- | --- |
| Public, anonymous, static, non-personalized | Optional Cloudflare or Vercel after an allowlist/data-flow review; AWS-only remains simpler for Release 1 |
| Authenticated UI/BFF/API | Keep AWS ECS Fargate `ap-east-1`; Azure Container Apps is the first alternative to PoC if a trigger occurs |
| Identity/session | Keep Cognito `ap-east-1` plus RDS-authoritative business authorization and immediate session denial |
| Database/documents/audit/logs | Keep private RDS/S3/AWS Hong Kong design; do not mix R2 or Vercel Blob into the sensitive document domain |
| Crawler/scanner/AI worker | Keep isolated Hong Kong containers; model inference remains a separate, unapproved boundary |

## 2. Problem framing, scope, and invariants

### 2.1 Problem and stakeholders

This research asks whether another production platform can provide a lower-risk or materially lower-total-cost Hong Kong system for sensitive student, guardian, case, identity, document, audit, and future AI workloads.

- Founder/Product need a defensible release path and an exit option, not a vendor feature list.
- Security/Privacy/Data need service-by-service evidence for every primary and secondary copy.
- Operations/Budget need capacity, restore, support, observability, and migration costs, not compute-only pricing.
- Advisors and customer organizations need stable authenticated workflows and strict case/tenant isolation.

### 2.2 In scope

- Azure East Asia Container Apps, App Service, AKS, PostgreSQL Flexible Server, Blob/Queue, Entra External ID, logs, backup, private networking, support, and worker patterns.
- Cloudflare Workers and R2; Vercel Functions/Blob/Secure Compute.
- Google Cloud Hong Kong as a shadow candidate where official evidence is genuinely comparable.
- One common pilot/10-tenant/50-tenant workload, cost structure, uncertainty, migration, and rollback.
- Explicit separation of public and authenticated planes.

### 2.3 Out of scope

- Changing the TDR/DEC ledger, purchasing an enterprise plan, legal advice, mainland deployment, cloud account inspection, calculator exports from an account, provisioning, benchmark execution, migration, or deployment.
- Treating a cloud region label, certification, or low list price as end-to-end residency evidence.
- Selecting an AI/model provider. Hong Kong worker placement and model-inference residency remain separate approvals.

### 2.4 Non-negotiable invariants and enforcement owners

| Invariant | Enforcement owner |
| --- | --- |
| Sensitive content, metadata, authorization, identity/session records, audit, operational logs, backup, and support traces remain inside the approved boundary | Infrastructure + Privacy; application cannot compensate for a provider copy outside the boundary |
| IdP proves identity only; organization, membership, role, case capability, expiry, and `session_version` remain database truth | Identity adapter + Access module + PostgreSQL/RLS |
| Mandatory mutation audit is transaction-bound and fail-closed | Owning application command + AuditOperations repository transaction |
| Browser document access uses a short-lived single-action/single-object intent; private object storage is not a public origin | Documents module + storage policy/private endpoint |
| Public plane receives no cookie, authenticated request, PII, preview data, ERP response, or sensitive log | Routing/CDN allowlist + build/release policy |
| Worker retries are bounded/idempotent; crawler/scanner/AI cannot share request-path resource limits or ambient access | Job/queue contract + owning application command |
| Backup is restorable evidence, not merely replication; cross-region replication is forbidden unless separately approved | Data/Operations restore gate |

## 3. Common workload and uncertainty model

These are replaceable planning assumptions supplied for this comparison. They are not forecasts, quotas, or release approval.

| Profile | Tenants | Users | Active / retained cases | Logical private documents |
| --- | ---: | ---: | ---: | ---: |
| Pilot | 1 | 10 | 10 / 100 | 50 GB |
| Growth | 10 | 100 | 100 / 1,000 | 500 GB |
| Scale | 50 | 500 | 500 / 5,000 | 2.5 TB |

Sensitivity parameters:

- monthly new/uploaded document versions: **10% of logical document stock**;
- monthly Internet downloads: **20% of logical document stock**;
- these two ratios test cost sensitivity; they are not inferred user behavior;
- backup/soft-delete/versioning multipliers must be added separately and can make physical billed storage exceed logical storage materially.

Structured DB, vector/search, logs, OCR, scanner compute, and HTTP request rates are intentionally independent low/high bands. Case count does not determine them.

| Meter | Pilot low–high | Growth low–high | Scale low–high |
| --- | ---: | ---: | ---: |
| Dynamic HTTP requests/month | 0.25–1 M | 2.5–10 M | 12.5–50 M |
| Active runtime CPU-hours/month | 20–80 | 200–800 | 1,000–4,000 |
| Provisioned memory GB-hours/month | 100–400 | 1,000–4,000 | 5,000–20,000 |
| Structured DB allocated storage | 50–150 GB | 150–500 GB | 500 GB–2 TB |
| DB compute starting sensitivity | 2 vCPU / 8 GiB | 4 vCPU / 16 GiB | 8 vCPU / 32 GiB |
| Vector/search derived data | 1–5 GB | 10–50 GB | 50–250 GB |
| Ingested logs/month | 5–25 GB | 50–250 GB | 250 GB–1.25 TB |
| OCR pages/month | 0–500 | 0–5,000 | 0–25,000 |
| Scanner/batch CPU-hours/month | 5–25 | 50–250 | 250–1,250 |

No platform receives a pass from list-price arithmetic. A release estimate must bind an exact application build to measured CPU/memory/request/log/DB/scan data and a dated vendor calculator or price-list checksum.

## 4. Public plane versus authenticated plane

The existing repository is a full-stack Next.js application: moving the repository moves UI, BFF, Route Handlers, server actions, cookies, cache behavior, and logs together unless code is explicitly split.

### 4.1 Public plane contract

Allowed:

- immutable public assets and a separately built static public site;
- anonymous school facts already approved for public release;
- generic availability/health response containing no tenant or case state.

Denied:

- authentication pages that receive credentials or session cookies;
- authenticated layouts, Route Handlers, Server Actions, previews, personalization, document links, tenant-specific search, crawler review queues, or admin UI;
- cache keys, analytics, errors, traces, support payloads, or build previews containing personal or tenant data;
- proxying authenticated requests through a public-plane vendor merely to reach a Hong Kong origin.

### 4.2 Authenticated plane contract

The complete TLS-decryption, request execution, identity, authorization, DB, object, queue, log, backup, support, and restore path requires service-specific evidence. A globally distributed CDN in front of it is not automatically forbidden, but it becomes part of the sensitive processing boundary and must prove exactly what it terminates, stores, logs, and transfers.

This distinction matters because Cloudflare can restrict HTTPS decryption/execution to Hong Kong Regional Services, while its Customer Metadata Boundary cannot be set to Hong Kong; Vercel can run a Function in `hkg1`, while its core data plane and permitted processing locations remain broader. A compute-region selector is not a data-flow proof.

## 5. Azure East Asia deep dive

### 5.1 Region and topology facts

Microsoft identifies `eastasia` as Hong Kong, pairs it with Southeast Asia (Singapore), and marks it availability-zone capable ([Azure regions list](https://learn.microsoft.com/en-us/azure/reliability/regions-list)). This supports an in-region zone-resilient design, but the paired region creates a critical configuration trap: geo-redundant storage or backup can leave Hong Kong.

The Azure candidate must therefore use:

- East Asia placement and Azure Policy/resource-location controls;
- zone redundancy or local redundancy only where the service supports it;
- no GRS/GZRS/RA-GRS, cross-region read replica, workspace replication, or automatic out-of-region failover without a new approval;
- an approved manual region-outage operating state rather than silent Singapore failover.

### 5.2 Runtime choice

| Runtime | Fit | Private network / reliability | Worker model | Operations and cost shape | Verdict |
| --- | --- | --- | --- | --- | --- |
| **Container Apps** | Runs the existing Linux/Node container without a framework adapter; revision testing and traffic split are native platform concepts | Workload-profile environments support custom VNet, private endpoints, UDR/NAT, public-access disablement, and zone-redundant Front Door/private-link patterns ([networking](https://learn.microsoft.com/en-us/azure/container-apps/networking), [private endpoints](https://learn.microsoft.com/en-us/azure/container-apps/private-endpoints-with-dns)) | Manual, scheduled, and event-driven run-to-completion Jobs share environment networking/logging; KEDA scalers support queue-driven work ([Jobs](https://learn.microsoft.com/en-us/azure/container-apps/jobs)) | Consumption can scale to zero; dedicated profiles add predictable capacity. Private endpoints add both Private Link and Container Apps dedicated management charges. | **Azure PoC candidate.** Best balance of container parity, jobs, and managed operations. |
| **App Service** | Linux supports Node stacks and custom Docker images ([custom container](https://learn.microsoft.com/en-us/azure/app-service/quickstart-custom-container)) | Private endpoints are available from Basic through Premium/Isolated plans; outbound VNet integration uses a separate subnet ([private endpoint](https://learn.microsoft.com/en-us/azure/app-service/overview-private-endpoint)) | Does not by itself replace crawler/scanner/event jobs; pair with Container Apps Jobs/Functions or dedicated worker App Service | Fixed plan/instance billing is easier to forecast but can overprovision pilot idle time. Multiple production instances are still required for zone/instance resilience. | **Fallback.** Prefer only if the team chooses simpler always-on web operations and accepts a second worker service. |
| **AKS** | Native container/Kubernetes portability and maximum scheduling/network control | Private clusters keep API-server/node traffic on private networking; Standard/Premium tiers provide 99.9%/99.95% API-server SLA depending on zones ([private clusters](https://learn.microsoft.com/en-us/azure/aks/private-clusters), [pricing tiers](https://learn.microsoft.com/en-us/azure/aks/free-standard-pricing-tiers)) | CronJobs, Deployments, autoscalers, queue consumers, GPU/node pools if available | Production requires paid control-plane tier plus nodes, disks, load balancers, registry, upgrades, policy, ingress, autoscaling, monitoring, and Kubernetes expertise. | **Reject for current three profiles.** Revisit only for a measured Container Apps blocker, existing platform team, or contractual Kubernetes requirement. |

Container Apps has a non-obvious network cost: enabling a private endpoint bills both Azure Private Link and dedicated private-endpoint infrastructure even for Consumption profiles ([Container Apps private endpoint billing](https://learn.microsoft.com/en-us/azure/container-apps/private-endpoints-with-dns)). The low compute price cannot be evaluated without that fixed boundary cost.

### 5.3 Next.js 16 compatibility

Container Apps, App Service custom containers, and AKS run the standard self-hosted Node/Docker artifact; they do not require an OpenNext/workerd adapter. This preserves Next.js self-hosting behavior but does not remove multi-instance obligations: consistent build ID/deployment ID, Server Action encryption key, cache/tag invalidation, streaming proxy behavior, graceful draining, and rollback must be tested against the pinned 16.2.7 build ([Next.js self-hosting](https://nextjs.org/docs/app/guides/self-hosting)).

App Service's built-in Node stack should not be treated as lower risk than the repository's container until its exact Node version, standalone output, image processing, filesystem assumptions, and startup contract pass. A custom container gives closer parity and a clearer rollback digest.

### 5.4 Data, storage, backup, and network boundaries

| Service | Hong Kong-positive evidence | Boundary/failure issue | Required configuration/evidence |
| --- | --- | --- | --- |
| PostgreSQL Flexible Server | Private VNet injection gives no public endpoint; Private Link traffic stays on Microsoft's backbone ([private access](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-networking-private), [Private Link](https://learn.microsoft.com/en-us/azure/postgresql/network/concepts-networking-private-link)) | Private networking is selected at creation and has subnet/DNS constraints. Managed backup files cannot be exported. Geo-backup asynchronously copies to the paired region and can have up to one-hour RPO. | East Asia, private-only, same-/zone-redundant HA if supported, local/ZRS backup only, 7–35-day retention, plus tested logical export/restore owned by Tianxing. No geo-backup. |
| Blob Storage | Regional account plus LRS/ZRS can keep copies local/in-region; private endpoints, versioning, soft delete, lifecycle, and WORM are available | GRS/GZRS copy to a geographically distant paired region. Physical billed bytes exceed logical bytes under versions/soft delete. Storage-account region migration is a copy to a new account and can require downtime ([redundancy migration](https://learn.microsoft.com/en-us/azure/storage/common/redundancy-migration)). | East Asia LRS/ZRS only; public access disabled; private endpoint; CMK decision; version/soft-delete/retention and malware quarantine contract; inventory and restore probes. |
| Queue/worker state | Storage Queue or Service Bus can feed Container Apps Jobs; event jobs have bounded timeout/retry settings | Queue payload, DLQ, scaler metadata, logs, and poison-message receipts all become sensitive secondary copies | Opaque IDs only where possible; same-region queue/DLQ; explicit idempotency key; no PII in scaler/config; reconciliation and replay tests. |
| Log Analytics | East Asia supports zone-redundant Azure Monitor Logs. A workspace stores/processes logs in chosen primary/secondary regions ([availability zones](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/availability-zones), [workspace replication](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/workspace-replication)) | Workspace replication would select a second region. High-volume request/free-text logs can dominate cost and privacy exposure. | One East Asia workspace, no replication, PII allowlist/redaction, 30-day product telemetry, separately retained mandatory audit, ingestion budget and export rules. |
| Support | Customer Lockbox lets an approver accept/reject rare support data-access requests and supports App Service, AKS, PostgreSQL, Storage, Log Analytics, and Entra diagnostics ([Customer Lockbox](https://learn.microsoft.com/en-us/azure/security/fundamentals/customer-lockbox-overview)) | Requires at least Developer support plan; does not cover external legal demands, and not every service/path is necessarily in the supported list | Contract/DPA/subprocessor/legal review, Lockbox enabled, named approvers, ticket-redaction runbook, evidence that Container Apps support paths are covered or otherwise constrained. |

Azure PostgreSQL backup deserves explicit treatment. The service provides PITR and local/zone/geo backup options, but PITR creates a new server, geo backup copies to the paired region, managed backup files are not exportable, and an HA restore returns a single instance until HA is re-enabled ([backup and restore](https://learn.microsoft.com/en-us/azure/postgresql/backup-restore/concepts-backup-restore)). A successful portal backup icon is therefore insufficient rollback evidence.

### 5.5 Identity: the blocking Azure gap

Entra External ID is attractive for Microsoft-centric customer federation and the first 50,000 MAU core tier is free ([External ID pricing](https://azure.microsoft.com/en-us/pricing/details/microsoft-entra-external-id/)). Cost is not the blocker.

The blocker is residency and behavior:

- Entra core-store scale units use two or more Azure regions.
- External tenants let the creator select a geographic location, but that is not evidence of Hong Kong-only identity processing.
- External ID Go-Local is currently available only in Australia and Japan.
- Basic External ID logs are retained for seven days; longer retention requires export to Azure Monitor ([Entra log retention](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/reference-reports-data-retention)).
- Support troubleshooting may access/copy identity objects and diagnostic logs; Customer Lockbox and ticket controls must be configured ([Entra support data](https://learn.microsoft.com/en-us/troubleshoot/entra/entra-id/ad-dmn-services/support-data-collection-diagnostic-logs)).

Possible Azure runtime options do not erase the issue:

1. **Keep Cognito while moving runtime/data to Azure:** preserves a known HK IdP but adds cross-cloud authentication, egress, incident, IAM, support, and availability boundaries. It gives up the main operational argument for Azure.
2. **Use Entra External ID:** requires Privacy/Security to approve a broader identity geography and different MFA/log semantics. That would supersede, not satisfy, the current HK invariant.
3. **Self-host credentials/auth in Azure PostgreSQL:** maximizes HK control but transfers password hashing, MFA enrollment/recovery, anti-abuse, credential breach response, email delivery, and support burden to Tianxing. This is not a small substitute for Cognito.

For Release 1, none beats Cognito plus RDS-authoritative authorization.

### 5.6 Crawler, scanner, OCR, and AI workers

Container Apps Jobs fit the finite worker model: manual, scheduled, or event-driven execution; apps and jobs share environment networking/logging; timeouts, retry limits, parallelism, and completion counts are explicit. Use separate job definitions and managed identities for crawler, scanner, OCR/index, and future agent work.

Required separations:

- scanner gets one quarantined object/version and emits a typed scan receipt;
- crawler reads approved public sources and publishes only through the existing review/release contract;
- OCR/index receives only approved document purpose/scope and writes derived, rebuildable data;
- AI worker receives no ambient DB/Blob access and calls versioned policy-checked tools;
- request-path Container Apps replicas do not share worker minimum/maximum replica limits;
- queue/DLQ retry and cost budgets are independent per worker.

Azure model hosting does not currently repair AI residency. Microsoft documents Standard/Regional model deployments as processing in the deployment region, but model/region availability must be checked per model; Asia Pacific data-zone definitions are broader than Hong Kong ([Foundry model region availability](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/models-sold-directly-by-azure-region-availability)). No real case/document may reach a model until the exact endpoint, model, inference location, logs, abuse monitoring, retention, support, and DPA are separately approved.

## 6. Cloudflare deep dive

### 6.1 What is technically strong

Cloudflare's official Next.js path uses `@opennextjs/cloudflare`; most App Router, Route Handler, RSC, SSR, ISR, Server Action, streaming, middleware, and cache features are listed as supported. Production runs on `workerd`, not the Node.js runtime used by `next dev`, and Node.js middleware is not yet supported ([Cloudflare Next.js](https://developers.cloudflare.com/workers/framework-guides/web-apps/nextjs/)). Node compatibility remains a subset with some polyfills that throw when called ([Node.js compatibility](https://developers.cloudflare.com/workers/runtime-apis/nodejs/)).

Worker Standard pricing is low: $5/month minimum, 10 million requests and 30 million CPU-ms included, then $0.30/million requests and $0.02/million CPU-ms; static-asset requests are free ([Workers pricing](https://developers.cloudflare.com/workers/platform/pricing/)). R2 Standard is $0.015/GB-month, $4.50/million Class A, $0.36/million Class B, with 10 GB, 1 million A, and 10 million B free; direct R2 Internet egress is free ([R2 pricing](https://developers.cloudflare.com/r2/pricing/)).

Those are public-plane strengths, not proof of sensitive-core fitness.

### 6.2 Residency and state blockers

Cloudflare supports Hong Kong Regional Services for HTTPS decryption and servicing, but the controls do not compose into a Hong Kong-only core:

- Worker code and secrets are deployed globally even when execution is regionalized.
- Regional Services does not extend to outgoing subrequests.
- Regional Services does not apply to Queue or Cron triggers.
- Customer Metadata Boundary supports only the United States or European Union; Hong Kong is explicitly unavailable ([Workers localization caveats](https://developers.cloudflare.com/data-localization/how-to/workers/), [region support](https://developers.cloudflare.com/data-localization/region-support/)).
- R2 `apac` is only a best-effort location hint. Guaranteed jurisdictions are EU and FedRAMP; there is no Hong Kong jurisdiction ([R2 data location](https://developers.cloudflare.com/r2/reference/data-location/)).

Therefore Workers/R2 cannot own authenticated requests, identity/session state, private documents, job payloads, audit, or sensitive logs under the current invariant. A Hong Kong edge location or TLS regionalization is not object/log/state residency.

### 6.3 Permitted role and migration implication

Cloudflare may host a separately built public/static plane or provide reviewed DNS/WAF/CDN services. It must not tunnel to the private database, hold authenticated cookies, or proxy ERP traffic merely for edge performance.

A full Workers move would require:

- OpenNext adapter and `workerd` compatibility changes/tests;
- replacement of Node-specific libraries and middleware assumptions;
- cache/ISR/Server Action/streaming and preview test matrix;
- a new private DB connection path and threat model;
- replacement of S3/SQS/Cognito/AWS audit/log adapters;
- an Enterprise localization contract that still cannot currently solve R2 and metadata residency.

It is a broader rewrite than a container move and has no residency-complete target today.

## 7. Vercel deep dive

### 7.1 Strengths and real Hong Kong capabilities

Vercel is the lowest application-code migration option for Next.js. `hkg1` maps to AWS `ap-east-1`; Vercel Functions and Blob stores can be placed in Hong Kong ([Vercel regions](https://vercel.com/docs/regions), [Vercel Blob](https://vercel.com/docs/vercel-blob)). Hong Kong Fluid Compute list prices are $0.176/active CPU-hour and $0.0146/provisioned-memory GB-hour, with the first million invocations included and $0.60/million thereafter ([Fluid Compute pricing](https://vercel.com/docs/functions/usage-and-pricing)).

Vercel Blob supports private stores, but private delivery flows through a Function and charges both Blob transfer and Function/CDN transfer. Storage starts at $0.023/GB-month; Blob data transfer starts at $0.05/GB, while Hong Kong Fast Data Transfer is $0.16/GB. Cache misses and Fast Origin Transfer add cost ([Blob pricing](https://vercel.com/docs/vercel-blob/usage-and-pricing), [regional pricing](https://vercel.com/docs/pricing/regional-pricing)).

### 7.2 Why `hkg1` is insufficient

Vercel's compliance documentation says customer data may be transferred to and in the United States and anywhere else Vercel or its service providers maintain processing operations. Its core database/data plane is globally replicated, and failover can replicate/reroute across regions ([Vercel compliance](https://vercel.com/docs/security/compliance)). Every request also traverses Vercel's global CDN before the configured Function region ([Vercel CDN](https://vercel.com/docs/how-vercel-cdn-works)).

Vercel Queues explicitly state that strict residency is not supported: during a regional outage, messages may be stored temporarily in a neighboring region ([Vercel Queues residency](https://vercel.com/docs/queues/concepts)). That is incompatible with an unapproved silent failover for sensitive job state.

Secure Compute provides private connectivity/regional placement but is currently Enterprise; Enterprise pricing and support are contractual, and Pro/Hobby do not have a support SLA ([Secure Compute](https://vercel.com/changelog/secure-compute-is-now-self-serve), [support response](https://vercel.com/kb/guide/vercel-support-queue-time), [Enterprise](https://vercel.com/docs/accounts/plans/enterprise)). A private connection to RDS does not narrow Vercel's request/log/support/data-plane processing.

### 7.3 Permitted role

Use Vercel only for a separately built public plane unless an Enterprise order form/DPA and service-specific architecture prove:

- `hkg1` execution without out-of-region failover;
- acceptable CDN/TLS/request metadata handling;
- Hong Kong logs, support, build/preview, Blob versions/backups, and deletion behavior;
- private connectivity that does not expose RDS;
- deterministic Next.js rollback and a no-PII preview policy.

That evidence is not presently available in public documentation.

## 8. Google Cloud Hong Kong shadow candidate

Google Cloud is included because official evidence, unlike Cloudflare R2, establishes true Hong Kong regional services:

- Cloud Run services/jobs reside in a selected region and associated customer data is stored there; `asia-east2` is Hong Kong ([Cloud Run locations](https://cloud.google.com/run/docs/locations)).
- Cloud SQL PostgreSQL is available in `asia-east2` with HA editions and synchronous zone replication options ([Cloud SQL region availability](https://cloud.google.com/sql/docs/postgres/region-availability-overview), [locations](https://cloud.google.com/sql/docs/postgres/locations)).
- Cloud Storage states object data is stored in the selected location and lists `ASIA-EAST2` Hong Kong ([bucket locations](https://cloud.google.com/storage/docs/bucket-locations)).
- Google's current service terms state that for listed data-residency services, customer data at rest is stored only in the selected region/multi-region, while permitting replication within another region in the same country for support/reliability/etc. ([service terms](https://cloud.google.com/terms/service-terms), [listed services](https://cloud.google.com/terms/data-residency)).
- Organization Policy can restrict resources to the Hong Kong location group ([resource locations](https://docs.cloud.google.com/organization-policy/restrict-locations)).

It remains a **shadow candidate**, not a recommendation:

1. Identity Platform is not listed among Google's configurable data-residency services. Its multi-tenancy feature is an IdP directory/config boundary, not Tianxing business authorization and not Hong Kong residency evidence ([Identity Platform multi-tenancy](https://cloud.google.com/identity-platform/docs/multi-tenancy)).
2. `asia-east2` is Tier 2 Cloud Run pricing, and exact Cloud SQL/Storage/network/support totals require a calculator export tied to the same workload ([Cloud Run pricing](https://cloud.google.com/run/pricing), [Cloud Storage pricing](https://cloud.google.com/storage/pricing)).
3. AWS/Cognito/S3/RDS contracts, migrations, Terraform, restore evidence, and tests still require provider replacement.
4. It offers no customer/business requirement identified in the current record that outweighs migration and dual-operation risk.

GCP should be evaluated with Azure only if a formal multi-cloud tender or AWS exit trigger occurs. It should not be introduced as a third live control plane for optionality.

## 9. Common cost construction

### 9.1 Cost categories that must appear in every quote

| Category | Required meters |
| --- | --- |
| Web/BFF compute | minimum instances/replicas, active CPU, provisioned memory, requests, concurrency, cold starts, image registry |
| Ingress/security | load balancer/gateway, WAF, TLS, DDoS tier, public/private IP, DNS |
| Private networking | private endpoints/links, NAT/egress, tunnel/peering/VPN, DNS zones, cross-vendor transfer |
| Database | primary + standby/HA, storage/IOPS, connections/proxy, backup/PITR, logical export, restore test |
| Documents | logical storage × versions/soft-delete multiplier, writes/reads/list, retrieval, Internet transfer, CMK, inventory |
| Jobs | queue/API operations, DLQ, scanner/OCR CPU/memory, retry amplification, scheduler/scaler, orphan cleanup |
| Identity | MAU/federation, SMS/email, MFA, premium risk/governance, log export |
| Observability/audit | log ingestion/query/retention/export, traces/metrics/alerts, immutable mandatory audit |
| Reliability/support | multi-zone premium, support plan/SLA, Lockbox/access approval, backup copy, restore drill |
| Delivery/operations | CI/build minutes, artifact storage, SBOM/scanning, on-call, upgrades, incident and FinOps labor |

### 9.2 Storage/download sensitivity using published unit prices

The following isolates only logical document storage and 20% monthly Internet download. It excludes versions/soft delete, requests, KMS/CMK, scanner/OCR, private networking, DB, and support.

| Platform component | Pilot 50 GB / 10 GB download | Growth 500 GB / 100 GB | Scale 2.5 TB / 500 GB | Interpretation |
| --- | ---: | ---: | ---: | --- |
| Cloudflare R2 Standard | about **$0.60** storage; egress $0 | about **$7.35**; egress $0 | about **$37.35**; egress $0 | Uses 10 GB free then $0.015/GB-month. Attractive economics, but no HK jurisdiction; not a valid sensitive store. |
| Vercel Blob storage only | from **$1.15** | from **$11.50** | from **$57.50** | Uses global starting storage rate $0.023/GB-month; exact HK regional rate must be selected in calculator. |
| Vercel private delivery transfer floor | about **$2.10–2.77** | about **$21–27.70** | about **$105–138.50** | Blob transfer $0.05–0.117/GB plus HK Fast Data Transfer $0.16/GB; excludes Fast Origin Transfer/cache misses. |
| Azure Blob | `GB-month × East Asia LRS/ZRS rate + operations + transfer + private endpoint` | same formula | same formula | Official public HTML returned region-dependent placeholders; exact East Asia Retail Prices API/calculator export required. Do not substitute another region's rate. |
| Google Cloud Storage | `GB-month × asia-east2 class rate + operations + transfer` | same formula | same formula | Official page is region-selectable/dynamic; exact calculator export required. |
| AWS S3 baseline | See current AWS TDR and S3 assessment | Recalculate at 500 GB | Recalculate at 2.5 TB | Current `$1.71–6.57` scenario is 100-case-specific and must not be linearly presented as a full production quote. |

### 9.3 Publicly calculable serverless compute sensitivity

These figures use the independent HTTP bands in Section 3. They are not full-stack totals.

For Vercel, the low/high bands use the stated active CPU-hours and memory GB-hours:

```text
Vercel hkg1 variable compute
  = CPU-hours * $0.176
  + memory GB-hours * $0.0146
  + max(0, invocations - 1M) * $0.60/M
```

| Profile | Vercel `hkg1` variable compute sensitivity |
| --- | ---: |
| Pilot | about **$4.98–19.92/month** |
| Growth | about **$50.70–204.60/month** |
| Scale | about **$255.90–1,025.40/month** |

Add Pro/Enterprise plan/seat, CDN transfer, Blob, DB, identity, logs, WAF, private connectivity, support, and workers. The apparent zero-idle advantage does not solve residency.

For Cloudflare Workers, use 5–20 CPU-ms per dynamic request as a separate CPU sensitivity. With the $5 minimum, 10M included requests, and 30M included CPU-ms:

| Profile | Workers compute/request sensitivity |
| --- | ---: |
| Pilot | about **$5/month** |
| Growth | about **$5–8.40/month** |
| Scale | about **$6.40–36.40/month** |

This excludes an Enterprise Hong Kong Regional Services contract, R2, external PostgreSQL, private connection, identity, logs, workers, and support. It is evidence that edge compute can be cheap, not that a compliant system is cheap.

### 9.4 Azure same-workload cost envelope

Azure public pages expose the charging dimensions and free grants but returned region-dependent price placeholders in this review. A direct East Asia Retail Prices API query was unavailable in the research harness. Reporting a numeric Azure total would therefore be false precision.

The required East Asia calculator configurations are:

| Profile | Container Apps web/BFF | PostgreSQL | Blob/download | Jobs/logs | Fixed boundary costs to expose |
| --- | --- | --- | --- | --- | --- |
| Pilot | Consumption, min replicas 0/1 sensitivity; 20–80 CPU-h, 100–400 GiB-h, 0.25–1M requests | 2 vCPU/8 GiB start, HA off/on sensitivity, 50–150 GB, local backup | 50 GB LRS/ZRS; 5 GB writes; 10 GB download | 5–25 scanner CPU-h; 5–25 GB logs | Private endpoint management, Private Link, gateway/WAF, ACR, DNS, support |
| Growth | Consumption or Dedicated crossover; 200–800 CPU-h, 1,000–4,000 GiB-h, 2.5–10M | 4 vCPU/16 GiB, zone HA, 150–500 GB | 500 GB; 50 GB writes; 100 GB download | 50–250 worker CPU-h; 50–250 GB logs | Same, plus queue/DLQ, backup and alert/export |
| Scale | Dedicated profile comparison; 1,000–4,000 CPU-h, 5,000–20,000 GiB-h, 12.5–50M | 8 vCPU/32 GiB starting sensitivity, zone HA, 500 GB–2 TB | 2.5 TB; 250 GB writes; 500 GB download | 250–1,250 worker CPU-h; 250 GB–1.25 TB logs | Same; log ingestion and support can dominate |

Official pricing structure:

- Container Apps Consumption bills resource allocation/requests after monthly free grants of 180,000 vCPU-seconds, 360,000 GiB-seconds, and 2 million requests; Dedicated bills workload-profile instances ([Container Apps pricing](https://azure.microsoft.com/en-us/pricing/details/container-apps/)).
- PostgreSQL bills provisioned compute, storage, and backup; geo-redundant backup is twice the backup storage price ([PostgreSQL pricing](https://azure.microsoft.com/en-us/pricing/details/postgresql/flexible-server/)). Geo-backup is not allowed by the current boundary regardless of price.
- App Service bills provisioned plan instances; private endpoints and worker services are additional ([App Service Linux pricing](https://azure.microsoft.com/en-us/pricing/details/app-service/linux/)).
- AKS production adds Standard/Premium control-plane pricing and underlying nodes/resources. Free has no production SLA ([AKS pricing](https://azure.microsoft.com/en-us/pricing/details/kubernetes-service/)).
- External ID core is free through 50,000 MAU, so all three user profiles fit the free MAU quantity; premium/SMS/log export and the unresolved residency issue remain ([External ID pricing](https://azure.microsoft.com/en-us/pricing/details/microsoft-entra-external-id/)).

The cost decision gate is a versioned East Asia calculator export with resource IDs/SKUs, unit prices, quantity assumptions, support/tax, and checksum. Until then the Azure total is `unknown`, not `$0` and not “cheaper than AWS.”

### 9.5 Baseline comparison to current AWS planning figure

The current AWS research estimates approximately `$194.56–199.42/month` for two always-on Fargate tasks, one ALB/LCU, RDS Multi-AZ `db.t4g.small + 20 GiB`, and its 100-case S3 scenario. It explicitly excludes several production costs. That figure is a Release 1 baseline, not a 10- or 50-tenant quote.

An alternative only wins on cost if the same measured workload includes every category in Section 9.1 and migration/dual-run/rollback labor. Comparing Azure scale-to-zero compute or Workers' `$5` minimum to AWS's HA web+DB subtotal is invalid.

## 10. Operations, lock-in, migration, and rollback cost

### 10.1 Current coupling surface

The local code scan found 117 source/test/infra files containing AWS/Cognito/S3/RDS terms and approximately 1,218 lines of Terraform. These counts are navigation evidence, **not rewritten-line estimates**. High-impact couplings include:

- `shared/db.ts` validates an `ap-east-1` RDS hostname;
- migrations grant `rds_iam`;
- document upload contracts validate AWS SigV4 and literal `ap-east-1`;
- Cognito names appear in routes, revocation worker, and error contracts;
- existing restore and release evidence is AWS-shaped.

Migration must cover contracts, adapters, migrations, IaC, tests, runbooks, and evidence. Replacing resource names while retaining AWS behavior assumptions is a defect.

### 10.2 Work packages and planning effort

The following is a rough-order local engineering estimate, not vendor evidence or a commitment. Range uncertainty is at least ±100% until discovery/PoC closes unknowns.

| Target | Required work packages | Planning effort / dominant uncertainty |
| --- | --- | --- |
| Azure Container Apps full authenticated plane | Azure IaC/network/policy; container registry/release; DB IAM/network/migration; Blob intent/event/scan adapters; queue/jobs; Cognito-retain vs Entra decision; logs/audit/alerts; backup/restore; negative auth/security/performance; dual-run/rollback | **10–20 engineer-weeks + independent Security/Privacy/Ops review.** Identity decision, private-endpoint topology, restore semantics, and production evidence dominate. |
| Azure App Service | Same data/identity/storage migration plus App Service deployment slots/VNet and a separate worker platform | **10–22 engineer-weeks.** Web hosting is simpler; two compute models and fixed capacity offset that gain. |
| AKS | Azure data/identity/storage migration plus cluster/ingress/node/policy/autoscaling/upgrades/observability/on-call platform | **18–36 engineer-weeks plus ongoing Kubernetes ownership.** Not justified by current scale. |
| Cloudflare Workers full core | OpenNext/workerd compatibility, storage/queue/DB/identity replacements, localization contract, runtime test matrix, all common migration work | **16–30 engineer-weeks**, but no compliant target exists while R2/metadata/trigger residency gaps remain. |
| Vercel full core | Minimal Next.js app adaptation; Enterprise Secure Compute/network; DB/storage/identity/log/support/preview policies; all release/restore evidence | **8–16 engineer-weeks**, excluding Enterprise procurement. Blocked by public residency evidence, not app deployment effort. |
| GCP Cloud Run full core | GCP IaC/network/policy; Cloud SQL/GCS/Pub/Sub/jobs; identity decision; logs/support/restore; all contract/test/evidence replacement | **12–24 engineer-weeks.** Identity and new operational control plane dominate. |

These ranges do not price customer downtime, engineer opportunity cost, training, support contracts, security review, penetration testing, migration egress, duplicate environments, or pilot delay.

### 10.3 Rollback is a separate product

A credible platform switch needs a predeclared rollback threshold and the following receipts:

1. **Build:** one pinned Git SHA and provider-neutral container digest where possible; platform configuration separately checksummed.
2. **Identity:** provider-subject mapping and session invalidation plan. Password hashes are generally not portable; an IdP change may require controlled credential reset and parallel federation.
3. **Database:** source snapshot/checksum, migration transform version, read-only validation, delta/cutover mechanism, reverse compatibility window, and tested restore. No unbounded dual write.
4. **Objects:** immutable inventory/hash/version/scan-state manifest, bounded copy and delta reconciliation, abort/orphan handling, and proof that rollback does not serve stale/unscanned bytes.
5. **Jobs:** stop/drain/replay position, idempotency keys, DLQ ownership, late result cancellation, and no double side effects.
6. **DNS/traffic:** TTL, canary cohort, exact health/error/latency thresholds, and one-command/config-receipt return to the last known origin.
7. **Audit/evidence:** both platforms' audit timelines remain queryable and correlated; mandatory audit never becomes best effort during cutover.

Rollback after new writes is not “point DNS back.” DB/object/job divergence makes it a data recovery operation. The safest migration is a synthetic/staging rehearsal, then an empty-tenant production deployment, before any customer case is cut over.

### 10.4 Lock-in comparison

| Platform | Main lock-in | Exit posture |
| --- | --- | --- |
| AWS current | Cognito credential store, RDS IAM, SigV4/S3/SQS/KMS/IAM/Terraform/evidence | Keep application authorization provider-neutral; container image and PostgreSQL logical export are strongest escape points; maintain object inventory and IdP reset plan. |
| Azure Container Apps | ARM/managed identity/Private Link, Entra, Blob events/SAS, Container Apps/KEDA revisions/jobs, Azure Monitor | Container and PostgreSQL remain portable; isolate provider APIs behind existing effect adapters; export application-owned audit and logical DB/object manifests. |
| App Service | App settings/slots/VNet/managed identity plus same Azure data services | Similar to Container Apps; web container portable, worker topology less uniform. |
| AKS | Kubernetes APIs reduce scheduler lock-in, but Azure CNI/identity/load balancer/disks/policy/monitoring remain Azure-specific | Higher nominal portability but higher day-2 platform cost; portability must be rehearsed to count. |
| Cloudflare | OpenNext/workerd, bindings, R2/Durable Objects/Queues/Hyperdrive, localization contract | S3-compatible R2 helps object API portability, but placement/state/runtime semantics are provider-specific. |
| Vercel | Next.js platform primitives, Fluid Compute, Blob/Queues/cache/build/preview, Enterprise networking | Application code is easiest to move; platform behavior, cache, preview, logs, and managed state still require exit evidence. |
| GCP | Cloud Run IAM/revisions, Cloud SQL IAM, GCS signed URLs/events, Pub/Sub, Logging, Identity Platform | Container/PostgreSQL portable; cloud policy, events, identity, and operational evidence require replacement. |

## 11. What would and would not overturn AWS

### 11.1 Evidence insufficient to overturn the current decision

- Azure East Asia, Vercel `hkg1`, or Cloudflare Hong Kong edge appearing in a region list.
- A provider's general compliance certification, DPA, or “data residency” marketing page without service-specific copy/processing/support behavior.
- Entra's 50,000 free MAU, Workers' `$5` minimum, R2 free egress, or a scale-to-zero compute estimate.
- A successful hello-world container or framework deployment.
- An S3-compatible API claim without versioning, intent, event, scan, retention, restore, and residency parity.
- An HA/backup checkbox without a measured restore and proof of where every copy lives.
- Kubernetes portability without a second-provider deployment/restore rehearsal.
- Lower steady-state bill that excludes migration, dual operation, support, logs, network, scanner/OCR, and rollback.

### 11.2 Evidence sufficient to open a superseding decision

All of the following are required, not any one item:

1. **Business trigger:** material customer contract for Microsoft/GCP ecosystem, an AWS service/cost/availability failure, or measured workload threshold violation.
2. **Service inventory:** written provider/contract evidence for runtime, identity, DB, objects, queue, keys, logs, backup, support, build, CDN/TLS, and model/worker processing.
3. **HK boundary:** exact primary/secondary copies and failover behavior meet the approved Hong Kong definition; any broader identity geography is explicitly approved rather than hidden.
4. **PoC parity:** pinned Next.js 16 build passes auth, revocation, document intents, scanner/DLQ, audit failure, concurrency, multi-instance cache/actions, crawler, restore, and rollback tests.
5. **Cost parity:** dated calculator/price-list evidence for the same Section 3 low/high workload and every Section 9.1 category, plus migration/dual-run/support labor.
6. **Migration design:** versioned transform, identity reset/federation plan, object manifest, job drain/replay, cutover/rollback thresholds, and rehearsal evidence.
7. **Independent approval:** Security, Privacy, Data, Operations, Budget, Product, and Founder sign the exact payload; changed service/region/plan invalidates approval.

Azure would then be the first deep PoC, using Container Apps rather than AKS unless a measured blocker says otherwise. GCP is the comparator. Cloudflare/Vercel remain public-plane candidates unless their residency evidence changes materially.

### 11.3 Re-evaluation triggers

Reopen this research when any of these occur:

- Entra External ID Go-Local adds Hong Kong with acceptable MFA, log, support, and export behavior.
- Azure provides a service-by-service contractual Hong Kong package covering Container Apps, identity, PostgreSQL, Blob/Queue, Log Analytics, backup, support, and the selected model/worker services.
- Cloudflare adds an R2 Hong Kong jurisdiction and Hong Kong Customer Metadata Boundary, and Regional Services covers subrequests, Queues/Cron, logs, state, and support.
- Vercel offers an approved Hong Kong processing/support/backup boundary for authenticated Functions, Blob, queues, previews/builds, logs, CDN, and failover at an acceptable Enterprise price.
- Google Identity Platform or another managed CIAM publishes enforceable Hong Kong residency, or a customer contract requires GCP.
- AWS `ap-east-1` price, service availability, Cognito behavior, support, or outage record violates approved budget/SLO/residency thresholds.
- Measured pilot/growth data exceeds Fargate/RDS connection, latency, throughput, scanner, restore, or total-cost thresholds.
- `DEC-060` multi-customer semantics require isolation or federation topology the current design cannot provide.
- Annual review date: **2027-08-11**, even if no trigger fires, because product regions, prices, and contracts are time-sensitive.

## 12. Evidence gaps and next artifacts

This research is complete as a comparative advisory note, but it does not establish production readiness for any alternative.

Still required before any superseding proposal:

- exact Azure East Asia/GCP Hong Kong/Vercel Enterprise calculator exports with price date/currency/offer and checksums;
- provider sales/legal written answers for identity, support, backup, subprocessors, and region-outage behavior;
- architecture-specific data-flow diagrams separating content, metadata, credentials, logs, support, and derived data;
- benchmark and restore results from the pinned Tianxing build and synthetic workload;
- exact migration file/module inventory and adapter contract plan; 117 scan hits are not an implementation plan;
- independent legal/privacy assessment tied to real data classes and contracts;
- customer requirement evidence strong enough to justify migration risk.

Research limitations:

- Azure public pricing HTML returned region-dependent placeholders, and the direct Retail Prices API query was unavailable in this harness. Azure numerical totals are deliberately not fabricated.
- No vendor account, quote, enterprise contract, support ticket, DPA portal, cloud resource, secret, or production data was accessed.
- No legal opinion was obtained.
- `RTK.md`, referenced by the root instruction include, was not found in the workspace; the complete root instructions supplied in the task were followed.
- No lint/build/test was run because only a research Markdown file changed and `erp-frontend/AGENTS.md` prohibits those broad checks without explicit request.

## 13. Source index

All sources were accessed on 2026-08-11.

### Microsoft Azure / Entra

- [Azure regions list](https://learn.microsoft.com/en-us/azure/reliability/regions-list)
- [Container Apps networking](https://learn.microsoft.com/en-us/azure/container-apps/networking)
- [Container Apps private endpoints and billing](https://learn.microsoft.com/en-us/azure/container-apps/private-endpoints-with-dns)
- [Container Apps Jobs](https://learn.microsoft.com/en-us/azure/container-apps/jobs)
- [Container Apps pricing](https://azure.microsoft.com/en-us/pricing/details/container-apps/)
- [App Service custom containers](https://learn.microsoft.com/en-us/azure/app-service/quickstart-custom-container)
- [App Service private endpoints](https://learn.microsoft.com/en-us/azure/app-service/overview-private-endpoint)
- [App Service Linux pricing](https://azure.microsoft.com/en-us/pricing/details/app-service/linux/)
- [AKS private clusters](https://learn.microsoft.com/en-us/azure/aks/private-clusters)
- [AKS pricing tiers](https://learn.microsoft.com/en-us/azure/aks/free-standard-pricing-tiers)
- [AKS pricing](https://azure.microsoft.com/en-us/pricing/details/kubernetes-service/)
- [PostgreSQL private networking](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-networking-private)
- [PostgreSQL Private Link](https://learn.microsoft.com/en-us/azure/postgresql/network/concepts-networking-private-link)
- [PostgreSQL backup/restore](https://learn.microsoft.com/en-us/azure/postgresql/backup-restore/concepts-backup-restore)
- [PostgreSQL HA](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-high-availability)
- [PostgreSQL pricing](https://azure.microsoft.com/en-us/pricing/details/postgresql/flexible-server/)
- [Azure Storage redundancy migration](https://learn.microsoft.com/en-us/azure/storage/common/redundancy-migration)
- [Azure Monitor workspace replication/data residency](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/workspace-replication)
- [Azure Monitor availability zones](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/availability-zones)
- [Customer Lockbox](https://learn.microsoft.com/en-us/azure/security/fundamentals/customer-lockbox-overview)
- [Entra residency](https://learn.microsoft.com/en-us/entra/fundamentals/data-residency)
- [External ID pricing/Go-Local](https://learn.microsoft.com/en-us/entra/external-id/external-identities-pricing)
- [Entra log retention](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/reference-reports-data-retention)
- [Entra support diagnostics](https://learn.microsoft.com/en-us/troubleshoot/entra/entra-id/ad-dmn-services/support-data-collection-diagnostic-logs)
- [Foundry regional model processing](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/models-sold-directly-by-azure-region-availability)

### Cloudflare

- [Cloudflare Next.js/OpenNext](https://developers.cloudflare.com/workers/framework-guides/web-apps/nextjs/)
- [Workers Node.js compatibility](https://developers.cloudflare.com/workers/runtime-apis/nodejs/)
- [Workers pricing](https://developers.cloudflare.com/workers/platform/pricing/)
- [Workers localization caveats](https://developers.cloudflare.com/data-localization/how-to/workers/)
- [Data Localization region support](https://developers.cloudflare.com/data-localization/region-support/)
- [R2 data location/jurisdictions](https://developers.cloudflare.com/r2/reference/data-location/)
- [R2 pricing](https://developers.cloudflare.com/r2/pricing/)

### Vercel

- [Vercel regions](https://vercel.com/docs/regions)
- [Vercel compliance and processing locations](https://vercel.com/docs/security/compliance)
- [Vercel CDN request path](https://vercel.com/docs/how-vercel-cdn-works)
- [Fluid Compute pricing](https://vercel.com/docs/functions/usage-and-pricing)
- [Vercel Blob regions/private storage](https://vercel.com/docs/vercel-blob)
- [Vercel Blob pricing](https://vercel.com/docs/vercel-blob/usage-and-pricing)
- [Vercel regional pricing](https://vercel.com/docs/pricing/regional-pricing)
- [Vercel Queues residency](https://vercel.com/docs/queues/concepts)
- [Secure Compute](https://vercel.com/changelog/secure-compute-is-now-self-serve)
- [Enterprise plan](https://vercel.com/docs/accounts/plans/enterprise)
- [Support response/SLA boundary](https://vercel.com/kb/guide/vercel-support-queue-time)

### Google Cloud

- [Cloud Run locations](https://cloud.google.com/run/docs/locations)
- [Cloud Run pricing](https://cloud.google.com/run/pricing)
- [Cloud SQL PostgreSQL region availability](https://cloud.google.com/sql/docs/postgres/region-availability-overview)
- [Cloud SQL locations/HA](https://cloud.google.com/sql/docs/postgres/locations)
- [Cloud Storage bucket locations](https://cloud.google.com/storage/docs/bucket-locations)
- [Cloud Storage pricing](https://cloud.google.com/storage/pricing)
- [GCP services with data residency](https://cloud.google.com/terms/data-residency)
- [Google Cloud Service Specific Terms](https://cloud.google.com/terms/service-terms)
- [Resource location constraints](https://docs.cloud.google.com/organization-policy/restrict-locations)
- [Identity Platform multi-tenancy](https://cloud.google.com/identity-platform/docs/multi-tenancy)

### Existing baseline

- [Next.js self-hosting](https://nextjs.org/docs/app/guides/self-hosting)
- [AWS ECS/Fargate pricing source used by current research](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonECS/current/ap-east-1/index.json)
- [AWS RDS pricing source used by current research](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonRDS/current/ap-east-1/index.json)
- [AWS S3 pricing source used by current research](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonS3/current/ap-east-1/index.json)
- [Cognito pricing](https://aws.amazon.com/cognito/pricing/)

## 14. Terminal state

Research state: `passed` for advisory comparison.

The evidence is sufficient to rank alternatives and define a superseding-decision gate. It is insufficient to approve a migration or claim an exact alternative-cloud monthly total. AWS `ap-east-1` remains the lowest-risk authenticated production direction; Azure Container Apps is the first conditional PoC candidate; Cloudflare and Vercel remain public-plane-only; GCP remains a shadow comparator.
