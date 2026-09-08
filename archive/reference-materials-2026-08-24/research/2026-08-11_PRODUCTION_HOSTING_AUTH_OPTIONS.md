# Production Hosting, Identity, and Future Agent Runtime Options

> Research date: 2026-08-11 (Asia/Hong_Kong)  
> Scope: Tianxing K12 production ERP, private student/case data, large private document workflows, future multi-customer use, and future native AI agents.  
> Evidence rule: official product documentation, official price pages, and first-party price-list APIs only. No account access, provisioning, deployment, or secret inspection was performed.  
> Decision context: the repository already selects AWS Hong Kong `ap-east-1`, Cognito, private RDS PostgreSQL, private S3/KMS/SQS/scanner, application audit fail-closed, and product telemetry fail-open. This note evaluates whether new evidence justifies changing those decisions.

## Executive recommendation

**Do not move the authenticated ERP wholesale to Cloudflare, Azure, or another frontend platform. Keep the production system on AWS Hong Kong and self-host the existing Next.js 16 application as a container on ECS Fargate behind an Application Load Balancer. Keep Cognito `ap-east-1` for authentication and keep tenant, role, case, session-version, and capability authorization in Hong Kong RDS.**

Recommended production shape:

```text
Browser
  -> DNS / TLS / WAF
  -> ALB in ap-east-1 (two or more AZs)
  -> ECS Fargate Next.js 16 web/BFF tasks in private subnets
       -> Cognito User Pool ap-east-1 (authentication only)
       -> private RDS PostgreSQL ap-east-1 (authorization + business truth)
       -> private S3/KMS ap-east-1 (document bytes)
       -> SQS/DLQ + scanner/worker tasks ap-east-1
       -> Hong Kong audit/log/alert stores

Optional isolated public plane
  -> Vercel Pro, Cloudflare, or S3/CloudFront
  -> static, public, non-personalized assets only
  -> no cookie, authenticated request, PII, ERP API, preview data, or sensitive log
```

This is not an argument that AWS is universally best. It is the lowest-risk choice for this system because the current architecture, migrations, adapters, accepted decisions, and Hong Kong residency controls already converge on one AWS VPC. Next.js officially supports Docker/self-hosting, including streaming, while warning that multi-instance deployments must coordinate build identity, Server Action keys, cache, and tag invalidation ([Next.js self-hosting guide](https://nextjs.org/docs/app/guides/self-hosting), [Next.js deployment guide](https://nextjs.org/docs/app/getting-started/deploying)). ECS Fargate is available in `ap-east-1`, gives every task its own VPC network interface, and supports private-subnet connectivity to RDS and VPC endpoints ([Fargate regions](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate-Regions.html), [Fargate networking](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-task-networking.html)).

**Keep Cognito. Do not replace it with Auth.js.** Auth.js is an application authentication/session library, not a managed credential authority. Its own documentation says credential-provider users are not persisted and that the application assumes the password, abuse prevention, reset, and MFA burden; it recommends established providers for those controls ([Auth.js Credentials provider](https://authjs.dev/getting-started/providers/credentials)). Auth.js can wrap Cognito as an OAuth provider, but the current Tianxing opaque BFF session already owns that boundary, so adding Auth.js would create a second session abstraction without removing Cognito ([Auth.js Cognito provider](https://authjs.dev/getting-started/providers/cognito)). Auth.js also states that `@auth/*` packages other than database adapters are still under development and generally not production-ready ([Auth.js security policy](https://authjs.dev/security)).

**Do not choose the core cloud based on future managed-agent marketing.** As of the research date, Amazon Bedrock AgentCore does not list Hong Kong, Bedrock Agents Classic does not list Hong Kong and stopped accepting new customers on 2026-07-30, and Microsoft Foundry hosted agents do not list Azure East Asia/Hong Kong ([AgentCore regions](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agentcore-regions.html), [Bedrock Agents regions and maintenance status](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-supported.html), [Foundry hosted-agent regions](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/hosted-agents)). For Hong Kong-only PII, start with provider-neutral agent orchestration in Hong Kong ECS worker containers; expose narrow, policy-checked application tools and require a separate approval for any model or managed agent service that processes data outside Hong Kong.

## Decision criteria and non-negotiable invariants

The platform choice must preserve all of the following:

1. Student, Guardian, case, document metadata/content, identity/session state, authorization, audit, logs, backups, and support traces stay in the approved Hong Kong boundary.
2. RDS and document buckets remain private. The browser may receive short-lived, narrowly authorized S3 upload/download intents, never general storage credentials.
3. Authentication proves actor identity only. `organization_id`, membership, role, case scope, capability, expiry, and `session_version` are freshly enforced by RDS; browser claims and identity-provider groups are not business authorization evidence.
4. Mandatory mutation audit is transaction-bound and fail-closed. Optional product telemetry uses opaque allowlisted fields, fails open into an explicit degraded state, and never becomes an authorization oracle.
5. A second customer is not enabled merely by changing hosting. Shared-schema multi-tenancy still requires RLS, an unprivileged application role, organization-scoped S3 keys/presigned intents, cache/search/job/export isolation, negative cross-tenant tests, and support-access controls.
6. AI agents never receive ambient database or object-store access. They call versioned, least-privilege tools that repeat tenant/case authorization and create auditable side-effect receipts.

## Hosting comparison

| Option | Next.js 16 fit | Hong Kong / private-data fit | Documents, queues, workers | Multi-tenant and operations | Future agents | Tianxing verdict |
| --- | --- | --- | --- | --- | --- | --- |
| **AWS ECS Fargate `ap-east-1`** | Native Node/Docker self-hosting; no framework adapter. Multi-instance cache/build/key coordination is explicit. | Strongest fit with the accepted same-region VPC, private RDS, S3, KMS, and Cognito design. Fargate and Cognito have HK regional support. | S3/KMS + SQS/DLQ + ECS worker/scanner is already the accepted document state machine. | One IAM/VPC/logging/IaC control plane; application/RLS still owns tenant isolation. | Self-host agent workers in HK now. Managed AgentCore is not in HK, so it remains gated. | **Recommended authenticated production plane.** |
| **Azure East Asia (Hong Kong)** | Container Apps or App Service can run a Node/Docker application; Container Apps supports VNet environments, revisions, jobs, and KEDA event scaling ([overview](https://learn.microsoft.com/en-ie/azure/container-apps/overview), [jobs](https://learn.microsoft.com/en-us/azure/container-apps/jobs)). | East Asia is physically Hong Kong and has three zones, but Microsoft's infrastructure table describes its at-rest boundary as **Asia Pacific**, not Hong Kong-only ([Azure infrastructure table](https://datacenters.microsoft.com/globe/explore/?view=table)). Service-by-service contractual evidence would be required. | Azure PostgreSQL supports private VNet access; Blob/Queue or Service Bus can implement the workflow ([PostgreSQL private networking](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-networking-private), [Azure Storage redundancy](https://learn.microsoft.com/en-us/azure/storage/common/redundancy-migration)). | Functionally capable, but requires rewriting Terraform, IAM/RBAC, identity, storage events, scanner, logging, backup/restore, and evidence probes. | Foundry hosted agents do not currently list East Asia. Container-hosted custom agents remain possible. | **Capable greenfield alternative, unjustified migration now.** Revisit only if a customer/MS contract or Microsoft ecosystem requirement dominates. |
| **Cloudflare Workers** | Cloudflare supports most Next.js features through the OpenNext adapter, but production runs on `workerd`, not the Node runtime used in local `next dev`; Node.js middleware is not yet supported ([Cloudflare Next.js guide](https://developers.cloudflare.com/workers/framework-guides/web-apps/nextjs/)). | Hong Kong Regional Services exists but is an Enterprise add-on. Worker code/secrets are deployed globally; outgoing subrequests, Queues, and Cron triggers are not covered by Worker Regional Services ([region support](https://developers.cloudflare.com/data-localization/region-support/), [Worker localization caveats](https://developers.cloudflare.com/data-localization/how-to/workers/)). | Hyperdrive can tunnel to private PostgreSQL, but this adds Cloudflare/Tunnel/Access to the private DB path ([private DB connection](https://developers.cloudflare.com/hyperdrive/configuration/connect-to-private-database/)). R2 has only best-effort `apac` placement; its guaranteed jurisdictions are EU and FedRAMP, not Hong Kong ([R2 data location](https://developers.cloudflare.com/r2/reference/data-location/)). | Workers is attractive for public edge/static traffic, but core use would add an adapter, a second runtime, a tunnel, and Enterprise localization contracts. | Agents SDK provides durable sessions on Durable Objects, but that state inherits the same regional/metadata limitations ([Agents SDK](https://developers.cloudflare.com/agents/), [Durable Objects](https://developers.cloudflare.com/durable-objects/)). | **Do not use for authenticated core or private documents.** Optional public CDN/WAF only after a no-PII data-flow review. |
| **Vercel Pro (managed alternative)** | Best managed Next.js experience and minimal code change. | Commercial use is allowed on Pro, but this does not change the repository's accepted restriction: authenticated/PII routes and sensitive logs must not use Vercel without contract and data-flow evidence. | Not the system of record for RDS/S3/SQS/scanner. Private connectivity and residency controls would require a separate enterprise assessment. | Low frontend operations burden, but creates a second production vendor and support/log boundary. | Convenient UI hosting does not solve agent data residency or tool authorization. | **Pragmatic only for an isolated public plane.** Otherwise remove Vercel and serve the full ERP from AWS. |

### Why AWS Amplify is not the managed alternative

Amplify Hosting currently documents managed SSR support only for Next.js 12 through 15, while this repository is on Next.js 16.2.7. It also lists unsupported features such as on-demand ISR and streaming, and its documentation notes Lambda@Edge resources in `us-east-1` for some Next.js paths ([Amplify Next.js support](https://docs.aws.amazon.com/amplify/latest/userguide/ssr-amplify-support.html), [Amplify SSR features and limitations](https://docs.aws.amazon.com/amplify/latest/userguide/ssr-supported-features.html)). It is therefore not an acceptable production shortcut for the current application or Hong Kong-only authenticated processing.

### What “move the frontend” should mean

The current Next.js repository contains both UI and server-side BFF/Route Handlers. Moving it as a single artifact moves sensitive server execution too. The safe choices are:

- **Preferred:** deploy the complete Next.js server to ECS Fargate in Hong Kong. Serve immutable assets from the same origin initially; add a reviewed CDN only after cache-key and log analysis.
- **Optional split:** extract a truly public, static marketing/documentation plane and host only that plane on Vercel Pro or Cloudflare. No authenticated layout, server action, preview, personalization, crawler review data, or ERP API call crosses that boundary.
- **Avoid:** serve authenticated pages from an edge provider and call Hong Kong RDS through a public endpoint, database proxy, or tunnel merely to preserve frontend convenience.

## Identity comparison

| Requirement | Cognito `ap-east-1` | Auth.js | Microsoft Entra External ID |
| --- | --- | --- | --- |
| Product category | Managed regional user pool / IdP. | Open-source application authentication and session library; it delegates to a provider or makes the application own credentials. | Managed CIAM external tenant and OIDC/SAML IdP. |
| Hong Kong evidence | Cognito publishes a Hong Kong regional endpoint; its regional-data guide warns that optional features can cause cross-region transfers ([endpoints](https://docs.aws.amazon.com/general/latest/gr/cognito.html), [regional data considerations](https://docs.aws.amazon.com/cognito/latest/developerguide/security-cognito-regional-data-considerations.html)). | Data lives wherever the application, adapter, email, and upstream providers run; no managed residency commitment exists because Auth.js is software. | Tenant creation selects a geographic location, but Entra core-store scale units use two or more Azure regions; official material found in this review does not establish Hong Kong-only storage/processing ([Entra residency](https://learn.microsoft.com/en-us/entra/fundamentals/data-residency), [external tenant creation](https://learn.microsoft.com/ga-ie/entra/external-id/customers/how-to-create-external-tenant-portal)). |
| MFA | Supports authenticator-app TOTP; hardware MFA is not supported by Cognito TOTP ([TOTP guide](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-settings-mfa-totp.html)). | Credentials provider can accept a second factor, but the application/provider must implement enrollment, recovery, anti-abuse, storage, and audit. | External tenants currently document email OTP, paid SMS, and passkey/FIDO2 as second factors; the published list does not include authenticator-app TOTP ([External ID MFA](https://learn.microsoft.com/en-us/entra/external-id/customers/concept-multifactor-authentication-customers)). |
| Revocation | `RevokeToken`, `GlobalSignOut`, and `AdminUserGlobalSignOut`; locally verified JWTs can still appear cryptographically valid, so Tianxing's RDS `session_version` remains the immediate-denial control ([Cognito revocation](https://docs.aws.amazon.com/cognito/latest/developerguide/token-revocation.html)). | Database sessions can be changed/revoked server-side; JWT sessions require a blocklist or waiting for expiry ([Auth.js session strategies](https://authjs.dev/concepts/session-strategies)). | Managed token/session controls plus Conditional Access; application-side RDS denial would still be required for business access and provider partial failure. |
| B2B / customer roles | Cognito can identify/federate users, but Tianxing roles must stay in RDS. | Callbacks can attach claims, but using them as role truth would duplicate and weaken the accepted policy boundary. | Strong Microsoft-enterprise federation story and external tenants support customer/business identities; useful if customers demand Entra federation ([External tenant overview](https://learn.microsoft.com/en-us/entra/external-id/customers/overview-customers-ciam)). App roles still must not replace case policy. |
| Audit / operations | Cognito API and managed-login activity integrates with CloudTrail; AWS warns it does not automatically mask PII placed in arbitrary non-private fields ([Cognito CloudTrail](https://docs.aws.amazon.com/cognito/latest/developerguide/logging-using-cloudtrail.html)). | All auth audit, abuse detection, upgrades, adapters, credential flows, and incident response are application/team responsibilities. | Entra supplies sign-in/audit logs; External ID Basic retention is seven days, so longer retention needs Azure Monitor export ([Entra retention](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/reference-reports-data-retention)). |
| Current operational change | Already designed and locally implemented behind an adapter. | Adds a second session abstraction or replaces tested code; does not eliminate the IdP. | Cross-cloud identity migration, new admin plane, new log export, different MFA semantics, and unresolved HK-only evidence. |
| Recommendation | **Keep for Release 1.** Use TOTP, suppress unapproved email/SMS flows, and retain RDS immediate denial. | **Do not adopt now.** Reassess only as a thin protocol adapter if it replaces code rather than duplicates it and its production status changes. | **Do not migrate now.** Consider later federation for Microsoft-centric customer organizations after residency and MFA requirements are resolved. |

### Identity invariant for many customers

Neither one Cognito pool nor one Entra tenant automatically creates safe SaaS isolation. Use a stable provider identity key such as `(provider, provider_subject)` and map it to internal `User` and `OrganizationMembership` rows. Email is mutable contact data, not identity or tenant scope. A user may belong to more than one customer organization; the active organization must be selected and authorized from server-side membership state, not accepted from a browser claim.

For future enterprise SSO, prefer federation behind the same identity adapter rather than replacing authorization. A customer Entra tenant can authenticate its workforce through a supported OIDC/SAML path while the Tianxing RDS policy still decides organization, case, and capability access.

## Baseline pricing and assumptions

All figures are USD, on-demand, before tax, support, commitments, WAF, NAT/PrivateLink, logs, backups beyond baseline, scanner/OCR, AI/model tokens, build CI, email/SMS, and internet transfer unless stated. They are planning figures, not quotes. Recheck official calculators and preserve exact price-list checksums before approval.

### Recommended AWS baseline

Assumption: two always-on ARM Fargate tasks, each `1 vCPU + 2 GiB`, across two AZs; one ALB with an average of one LCU; existing approved RDS Multi-AZ `db.t4g.small + 20 GiB gp3`; the repository's 100-case S3 scenario.

| Component | Planning baseline | Official source |
| --- | ---: | --- |
| Two Fargate tasks | about **$79.18/month** | `ap-east-1` public [Amazon ECS Price List API](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonECS/current/ap-east-1/index.json): ARM Fargate `0.04449/vCPU-hour + 0.00487/GB-hour`, 730 hours. |
| ALB + average 1 LCU | about **$26.64/month** | `ap-east-1` public [ELB Price List API](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AWSELB/current/ap-east-1/index.json): `0.0277/ALB-hour + 0.0088/LCU-hour`, 730 hours. |
| RDS PostgreSQL Multi-AZ + 20 GiB gp3 | about **$87.03/month** | `ap-east-1` public [RDS Price List API](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonRDS/current/ap-east-1/index.json); detailed assumptions are already recorded in `docs/research/aws-postgresql-hong-kong-assessment.md`. |
| S3/KMS, 100-case document scenario | about **$1.71-$6.57/month** | `ap-east-1` [S3 Price List API](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonS3/current/ap-east-1/index.json), [KMS pricing](https://aws.amazon.com/kms/pricing/), and assumptions recorded in `docs/research/aws-s3-hong-kong-assessment.md`. |
| Cognito direct/social MAU | **$0 at current pilot scale** if the account/organization has not consumed the allowance | Cognito Essentials/Lite include 10,000 direct/social MAU per month indefinitely; federation via SAML/OIDC has a 50-MAU free allowance ([Cognito pricing](https://aws.amazon.com/cognito/pricing/)). |

**Core planning subtotal: approximately $194.56-$199.42/month.** A realistic approved budget must add CloudWatch/CloudTrail, WAF, Route 53, secrets/KMS keys, image registry, scanner/OCR, queue requests, backups, network egress, and support. NAT Gateway can materially change this number; prefer VPC endpoints for AWS dependencies and introduce NAT only for explicitly approved outbound destinations.

### Other platform baselines

- **Vercel Pro:** Hobby is explicitly personal/non-commercial. Pro charges `$20 per member/month`; usage beyond included credit is additional ([Vercel Hobby and upgrade terms](https://vercel.com/docs/plans/hobby), [Vercel pricing](https://vercel.com/pricing)). This price is relevant only to the optional public plane, not a substitute for AWS runtime/storage/identity costs.
- **Cloudflare Workers:** Workers Paid has a `$5/month` minimum including 10 million requests and 30 million CPU-ms; additional usage is `$0.30/million requests` and `$0.02/million CPU-ms`. Static asset requests are free ([Workers pricing](https://developers.cloudflare.com/workers/platform/pricing/)). However, Hong Kong Regional Services/Data Localization Suite is an Enterprise paid add-on with contract pricing, so the `$5` plan is not the compliant-core price ([Data Localization Suite](https://developers.cloudflare.com/data-localization/)). R2 Standard is `$0.015/GB-month`, `$4.50/million` Class A, `$0.36/million` Class B, with free egress, but it lacks a Hong Kong jurisdiction guarantee ([R2 pricing](https://developers.cloudflare.com/r2/pricing/), [R2 data location](https://developers.cloudflare.com/r2/reference/data-location/)).
- **Azure:** Container Apps publishes monthly free grants of 180,000 vCPU-seconds, 360,000 GiB-seconds, and 2 million requests, then bills per second/request ([Container Apps pricing](https://azure.microsoft.com/en-us/pricing/details/container-apps/)). Azure PostgreSQL bills provisioned compute per vCore-hour and storage/backup per GiB-month ([PostgreSQL pricing](https://azure.microsoft.com/en-us/pricing/details/postgresql/flexible-server/)). Microsoft's public HTML returned region-dependent prices as placeholders during this review, so no false dollar total is quoted; an East Asia calculator export is required. Entra External ID advertises the first 50,000 MAU free, but premium security, SMS, log export, and residency options can add cost ([External ID pricing](https://azure.microsoft.com/en-us/pricing/details/microsoft-entra-external-id/)). The dominant cost is still migration and duplicated operations, not pilot compute.

## Future native AI agents

The future agent architecture should be an application capability, not a reason to relocate the transactional system:

```text
User command
  -> normal BFF authentication + fresh RDS authorization
  -> create AgentRun with organization/case scope, policy version, budget, expiry
  -> HK queue
  -> isolated HK agent worker
       -> model adapter (approved data class and region only)
       -> tool gateway with explicit schemas
            -> read-only case/document retrieval tools
            -> separately approved mutation tools
       -> append-only trace/audit receipts with PII redaction
  -> human approval for high-impact actions
```

Required controls before any agent handles customer data:

1. Separate model inference residency from agent-runtime residency. An ECS process in Hong Kong does not make an overseas model call Hong Kong-resident.
2. Version prompts, model, tool schemas, policy, knowledge snapshot, evaluator, and cost budget. Do not place raw documents, DOM/input capture, or free text in product telemetry.
3. Use organization- and case-scoped retrieval namespaces. An agent does not inherit all data visible to the human who launched it.
4. Treat every tool mutation as an ordinary application command: authorization, idempotency, audit transaction, outbox, bounded retry, reconciliation, and human approval still apply.
5. Keep model/provider adapters portable. AgentCore supports multiple frameworks and models and VPC access, Azure and Cloudflare also have useful agent runtimes, but none currently removes the Hong Kong evidence gap ([AgentCore overview](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/), [AgentCore VPC connectivity](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agentcore-vpc.html), [Cloudflare Agents](https://developers.cloudflare.com/agents/)).

## Decision and reassessment triggers

### Approve now

1. **Hosting:** AWS ECS Fargate + ALB in `ap-east-1` for the complete authenticated Next.js/BFF runtime and separate worker services.
2. **Documents:** retain private S3/KMS/SQS/DLQ/scanner in `ap-east-1`.
3. **Identity:** retain Cognito `ap-east-1`, TOTP-only for Release 1, opaque BFF cookie, and RDS-authoritative authorization/session revocation.
4. **Vercel:** either upgrade only an isolated public static plane to Pro or remove Vercel from production. Do not send authenticated ERP traffic to it.
5. **AI:** self-host initial orchestration/workers in Hong Kong and keep model/agent providers behind adapters and explicit data approvals.

### Reassess Azure when

- a material customer requires Microsoft Entra federation, Microsoft 365/Graph, or a Microsoft contractual/compliance package;
- Microsoft provides service-specific written evidence that all required identity, runtime, database, storage, logs, backups, support, and agent processing meet the project's Hong Kong boundary;
- an East Asia costed architecture plus migration/rollback rehearsal beats the AWS total cost and risk.

### Reassess Cloudflare core hosting when

- self-serve or contractually affordable Hong Kong localization covers Workers execution, subrequests, queues/cron, logs/metadata, Durable Objects, R2, support, and backups;
- R2 offers a Hong Kong jurisdiction rather than a best-effort APAC hint;
- the Next.js OpenNext production test matrix passes all Node/runtime, streaming, cache, server-action, document, and failure semantics.

### Reassess Cognito when

- customers require federation features it cannot meet, Cognito's regional/optional-feature behavior changes, or provider outages violate measured objectives;
- Entra/another managed provider proves Hong Kong-only data handling and the required MFA/revocation/audit behavior;
- the approved exit rehearsal accounts for non-exportable password hashes and forces a controlled credential reset.

## Evidence gaps and next decision artifacts

This review does **not** authorize provisioning or deployment. Before a production apply, owners still need:

- an exact `ap-east-1` Terraform plan hash and redacted resource/data-flow inventory;
- a pinned Git SHA and immutable container digest built once and promoted across environments;
- an East/West AZ network plan, WAF/rate-limit rules, VPC endpoint/NAT decision, and monthly calculator export;
- Cognito optional-feature inventory proving Pinpoint, SMS, email, and unapproved cross-region dependencies are disabled;
- a Vercel removal or public-plane allowlist decision;
- multi-instance Next.js evidence for build ID, `deploymentId`, Server Action encryption key, cache/tag coordination, graceful draining, and rollback;
- service-specific DPA/subprocessor/legal review; this research is technical evidence, not a PDPO legal opinion;
- a separate agent data-classification and model-residency decision before any real case/document reaches an AI model.

