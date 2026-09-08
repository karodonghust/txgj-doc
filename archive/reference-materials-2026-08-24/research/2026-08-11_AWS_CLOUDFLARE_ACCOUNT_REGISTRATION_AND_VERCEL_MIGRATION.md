# AWS + Cloudflare Account Registration and Vercel Migration Plan

| Field | Value |
| --- | --- |
| Research date | 2026-08-11 (Asia/Hong_Kong) |
| Status | Advisory design only; not procurement, account registration, DNS change, deployment, migration, or release authority |
| Target boundary | Public Hong Kong school-admission releases on Cloudflare; authenticated CRM and all customer/personal data on AWS Hong Kong |
| Evidence rule | AWS, Cloudflare, Vercel, and Next.js first-party documentation plus current repository source only |
| Current repository | Next.js `16.2.7`, React `19.2.4`, pnpm `10.34.4`; Vercel build from GitHub; crawler snapshot currently committed into frontend Git |

> **Subsequent authority clarification:** `DEC-063` and `TECHNICAL_DESIGN_AWS_CLOUDFLARE_PRODUCTION_DEPLOYMENT.md` supersede this research wherever it suggests retaining Vercel as a post-cutover standby/rollback. After AWS cutover there are no new Vercel deployments and rollback is AWS-only. Every region-selectable AWS workload/data/identity resource must use `ap-east-1`; the design rejects the Cognito custom-domain path that requires `us-east-1` ACM/global CloudFront.

## 1. Decision and boundary

Adopt the following production target, subject to the human approvals in Section 13:

```text
Anonymous public data plane
  data.example.com
    -> Cloudflare zone, TLS, WAF/rate limits, Cache Rules
    -> R2 public-release bucket
       -> immutable release objects + checksum manifest
       -> latest.json written only after validation

Authenticated customer plane
  app.example.com
    -> DNS-only record, no Cloudflare proxy/caching
    -> AWS ALB + ACM certificate + WAF, ap-east-1
    -> >=2 ECS Fargate Next.js web/BFF tasks across >=2 AZs
       -> Cognito user pool, ap-east-1
       -> RDS PostgreSQL Multi-AZ, ap-east-1
       -> private S3/KMS for customer documents, ap-east-1
       -> SQS/DLQ + scanner/workers, ap-east-1
       -> Secrets Manager + CloudWatch/CloudTrail + backup, ap-east-1

Crawler publication
  reviewed crawler output
    -> deterministic public-release compiler
    -> no-PII/schema/provenance/count/checksum gates
    -> upload immutable release to R2
    -> verify read-back
    -> update latest.json last
```

Cloudflare is not a fallback database, identity provider, private document store, or proxy for authenticated ERP traffic. AWS remains authoritative for organizations, users, memberships, cases, authorization, audit, private files, mutable crawler operations, and recovery.

### 1.1 Public-data classification

The current four-file handoff must not be published as-is:

| Current artifact/state | Cloudflare disposition | Reason |
| --- | --- | --- |
| `records.json` | Publish only after public schema and provenance review | Intended school-admission facts |
| `publish_manifest.json` | Replace or transform into a public manifest | Existing manifest may contain internal paths/run details |
| `run_summary.json` | Optional sanitized statistics only | Operational diagnostics are not automatically public |
| `review_queue.json` | **Never public** | Contains unresolved evidence and internal review workflow |
| crawler tickets/config/review decisions/run metadata | **Remain AWS/RDS private** | Mutable operations and possible staff/customer context |
| source captures/screenshots/logs | **Never public by default** | Copyright, robots/licence, metadata, PII and operational risk |

Public school facts may still contain personal names, contact details, copyrighted extracts, or withdrawn information. Product/Privacy must approve the public schema, quote limits, provenance fields, correction procedure, licence/terms, takedown SLA, and retention before the first release.

## 2. Company registrations and ownership

### 2.1 AWS account structure

Register AWS using company-controlled aliases and payment/tax details. Do not create production in a founder's personal standalone account.

Recommended minimum organization:

```text
AWS Organization management/payer account   # no application workloads
  Security OU
    -> log-archive/security account           # recommended before production
  Workloads OU
    -> tianxing-nonprod account
    -> tianxing-prod account
```

If four accounts are operationally premature, the minimum acceptable start is management/payer + production + non-production, with the log archive added before accepting customer data. Account boundaries cost little by themselves and reduce accidental production access; services inside them create charges.

Required setup:

1. Create a paid AWS management account with a dedicated company distribution-list root email, company phone, legal entity, billing address, tax details, payment method, and invoice currency.
2. Enable AWS Organizations with all features and consolidated billing; do not run workloads in the management account.
3. Protect management root with at least two phishing-resistant MFA devices where supported; create no root access keys; split password/recovery inbox and recovery phone custody between people; record a two-person break-glass procedure.
4. Enable centralized root-access management for member accounts and remove member-account root credentials where the chosen account workflow supports it.
5. Populate company distribution lists for Billing, Operations, and Security alternate contacts on every account.
6. Enable IAM access to Billing, then grant Finance a scoped billing permission set rather than root access.
7. Enable an organization instance of IAM Identity Center. Use named workforce identities, MFA, short sessions, and permission sets such as `ReadOnly`, `DeveloperNonProd`, `DeployNonProd`, `DeployProd`, `SecurityAudit`, and `Billing`.
8. Keep `DeployProd`, database migration, DNS, KMS administration, and break-glass separate. Production CI assumes a narrowly scoped role using short-lived federation/OIDC; no long-lived AWS access keys.
9. Enable cost allocation tags/categories, Cost Explorer, Cost Anomaly Detection, AWS Budgets, invoice recipients, and monthly review ownership.
10. Select a support plan only after recording response-time requirements and named on-call ownership. The production plan must not assume paid support benefits that were not purchased.

AWS recommends group email for root, MFA, no root access keys, temporary credentials, Organizations, and IAM Identity Center for multi-account workforce access. It also distinguishes Billing, Operations, and Security alternate contacts. See [root-user best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/root-user-best-practices.html), [Organizations best practices](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_best-practices.html), [creating an organization](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_org_create.html), and [Billing setup](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/billing-getting-started.html).

### 2.2 Cloudflare company account

Create one company-owned Cloudflare account, not one account per employee. The initial sign-up email should be a controlled alias/distribution list because Cloudflare uses it for billing, usage, and recovery. Add at least two named Super Administrators, require strong MFA/passkeys on every human profile, and avoid normal operations with Super Administrator privileges.

Suggested role separation:

| Actor | Cloudflare role/scope |
| --- | --- |
| Two break-glass owners | Super Administrator; no routine token use |
| Finance | Billing only |
| DNS/release owner | Domain DNS for required zone; cache purge if required |
| Public-data publisher CI | Account-owned token restricted to one R2 bucket; object read/write only |
| IaC/release CI | Separate account-owned token restricted to exact zone/account permissions |
| Auditor | Read-only/audit-log access |

Account-owned API tokens are preferable for durable CI because they do not stop working when an employee leaves. Create separate read, publish, and infrastructure tokens; restrict bucket/zone; store only in an approved secret store; set an internal rotation/expiry schedule even when the vendor token persists until revoked. Never use the Global API Key.

Purchase/enable R2 only after Finance approves the plan, payment profile, spend alert approach, and invoice owner. Record Account ID, zone ID, bucket names, token identifiers/fingerprints, owners, and configuration hashes as evidence; never record token values.

Cloudflare's official guidance recommends a business alias for account creation and defines account members, roles, scopes, and account-owned service tokens. See [create account](https://developers.cloudflare.com/fundamentals/account/create-account/), [accounts and zones](https://developers.cloudflare.com/fundamentals/concepts/accounts-and-zones/), [account roles](https://developers.cloudflare.com/fundamentals/manage-members/roles/), [member policies](https://developers.cloudflare.com/fundamentals/manage-members/policies/), and [account-owned API tokens](https://developers.cloudflare.com/fundamentals/api/get-started/account-owned-tokens/).

### 2.3 Domain registrar and legal/procurement records

- Keep the registrar account company-owned, registrar-locked, MFA protected, and separate from hosting. Do not transfer the registration merely to change DNS.
- Confirm registrant/legal contacts, renewal payment, auto-renew, expiry alerts, recovery process, DNSSEC compatibility, and two-person authorization for nameserver or transfer-code changes.
- Execute/record AWS and Cloudflare terms, DPA/subprocessor review, seller/tax entity, support plan, data classification, public-data licence, and incident contacts.
- Retain ownership of the GitHub organization, container registry policy, source licences, and release signing identities independently of any departed employee.

## 3. DNS, hostnames, and TLS

### 3.1 Recommended hostname model

| Hostname | Target | Cloudflare mode | Cookies/data |
| --- | --- | --- | --- |
| `app.example.com` | AWS ALB `ap-east-1` | **DNS-only** | Authenticated ERP; host-only Secure cookies |
| `auth.example.com` | Cognito managed-login custom domain if approved | DNS-only unless separately reviewed | Authentication cookies/tokens |
| `data.example.com` | R2 custom domain | Proxied/custom-domain path | Anonymous public releases only |
| `www.example.com` | Separately decided public website | May be proxied | No ERP cookie or private API response |
| `status.example.com` | Approved status provider | Separate | No customer payload |

Using Cloudflare as authoritative DNS is operationally convenient because an R2 custom domain must belong to a zone in the same Cloudflare account. It does **not** imply proxying every hostname. Keep `app` and `auth` DNS-only so Cloudflare does not terminate or cache authenticated traffic. A partial/CNAME zone setup is an alternative when the purchased plan supports it; select one authoritative DNS owner and document it.

### 3.2 DNS migration controls

1. Inventory all current records without exposing TXT secret values: record name/type/TTL/owner/purpose/provider and a hash of value where sensitive. Include MX, SPF, DKIM, DMARC, CAA, verification, Vercel, GitHub and other SaaS records.
2. Lower only the future app record's TTL 24–48 hours before cutover. Nameserver migration is not required for an app-origin migration; if nameservers must change, reproduce and independently compare the full zone first.
3. Request an ACM public certificate in `ap-east-1` for the ALB hostname. Add its DNS-validation CNAME to Cloudflare DNS and leave it in place for managed renewal. Cloudflare must not proxy validation records.
4. Point a non-production hostname at the ALB and test TLS/SNI, redirect, headers, WAF, health, cookies, login callbacks and websocket/streaming behavior before production DNS.
5. At cutover, change only `app.example.com`; retain the Vercel deployment and previous DNS value during the rollback window. Do not remove the Vercel domain/project first.
6. After TTL plus observation window, verify multiple public resolvers, certificate chain, no split answer, no Cloudflare proxy on app, and no requests arriving at Vercel before retiring it.

ACM DNS validation works with non-Route 53 DNS: publish the generated CNAME once and retain it for automatic renewal. See [ACM DNS validation](https://docs.aws.amazon.com/acm/latest/userguide/dns-validation.html).

### 3.3 Cognito custom-domain exception

A Cognito user pool can remain in `ap-east-1`, but AWS officially requires a custom-domain certificate in **ACM `us-east-1`** and creates a global CloudFront distribution for managed login. This is not the same topology as ALB TLS. Before choosing `auth.example.com`, Privacy/Security must approve the certificate/control-plane and global managed-login/cookie path, or explicitly retain the provider prefix domain after testing callback and cookie behavior. See [Cognito custom domains](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-add-custom-domain.html).

## 4. AWS production configuration checklist

All resources below belong in the production account and `ap-east-1` unless an official service constraint and separate approval says otherwise. Terraform source, reviewed plan hash, exact account/region, and human approval precede any apply.

### 4.1 Network, load balancer, and web runtime

- VPC across at least two AZs; public subnets only for internet-facing ALB and approved egress components; ECS and RDS in private subnets.
- Security groups: Internet -> ALB 443 only; ALB -> web task port only; web/security-group -> RDS 5432 only; no public task or database IP.
- Decide VPC endpoints versus NAT from an exact dependency/price inventory. ECR API/DKR, S3, Secrets Manager, CloudWatch Logs, KMS, SQS and STS paths must work without accidental unapproved egress.
- ACM certificate, TLS policy, HTTP-to-HTTPS redirect, ALB access-log destination/retention, deletion protection and WAF association.
- WAF managed baseline plus rate limits sized from measured traffic. False-positive rollback and emergency bypass require named Security approval.
- ECR repository with immutable tags/digests, scan-on-push or approved enhanced scanning, lifecycle policy, SBOM/provenance retention, and cross-account CI push role.
- ECS Fargate cluster/service with at least two tasks across two AZs; pinned Linux/ARM compatibility if using ARM; CPU/memory/ephemeral storage; read-only root filesystem where compatible; non-root process; resource limits; execution and task roles separated.
- ALB target group health endpoint that proves process readiness without exposing dependency details. Configure health interval/threshold, deregistration delay, application shutdown on `SIGTERM`, and enough ECS stop timeout to drain in-flight requests.
- ECS rolling deployment circuit breaker with rollback, minimum healthy percent, maximum percent, autoscaling on CPU/memory/request or measured metric, and alarms for task churn/5xx/latency/unhealthy hosts.
- Production web uses Fargate on-demand; Spot may be used only for interruption-tolerant workers after idempotency and retry evidence.

AWS documents ALB/container health as deployment-circuit-breaker inputs and can automatically roll back a failed rolling deployment. See [ECS deployment circuit breaker](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-circuit-breaker.html) and [container health checks](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/healthcheck.html).

### 4.2 RDS PostgreSQL

- Private PostgreSQL 17 target, Multi-AZ, deletion protection, encryption with customer-managed KMS key, storage autoscaling ceiling, parameter/option groups, maintenance window and minor-version policy.
- Automated backup retention and PITR consistent with approved RPO/RTO; copy/retain final snapshots only under an approved termination policy; scheduled restore drills into an isolated environment.
- Enhanced Monitoring/Performance Insights or approved equivalent, log exports, alarms for storage, CPU, connections, replication/failover events and backup failures.
- Application role is non-owner and `NOBYPASSRLS`; every tenant table has `ENABLE` and `FORCE ROW LEVEL SECURITY`; transaction-local organization/actor context and real pool-reuse tests are release gates.
- Use IAM DB authentication only where the application adapter and pool refresh behavior are tested. Do not add RDS Proxy until measured connection pressure and `SET`/`set_config` pinning behavior justify it.
- Master credentials are never application credentials. If password-based administration remains necessary, let RDS/Secrets Manager manage and rotate the master secret; limit retrieval to migration/break-glass roles.

RDS can manage the master password in Secrets Manager and rotate it, avoiding manually managed credentials. See [RDS password management with Secrets Manager](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-secrets-manager.html).

### 4.3 Cognito and application session

- Separate production user pool and app client; no client secret in a browser client; exact callback/logout URLs; no wildcard production callback.
- TOTP/MFA, password/recovery, anti-abuse, message delivery, verified attributes, deletion/protection, token lifetimes, revocation, threat protection and log/export behavior require explicit configuration and cost/residency review.
- Stable internal identity is `(provider, provider_subject)`; email is mutable contact data. RDS remains authoritative for organization membership, role, capability, case scope, expiry and `session_version`.
- Rotate from current Vercel session/cookie configuration to the same opaque BFF session contract on AWS. Use host-only `Secure`, `HttpOnly`, appropriate `SameSite`, fixed domain/path, and a versioned signing/encryption secret from Secrets Manager.
- Before DNS cutover, add the AWS hostname to Cognito callback/logout allowlists without removing Vercel. Test login, logout, MFA, recovery, expired/revoked session, cross-tenant denial and rollback URL.

### 4.4 Private objects, queues, keys, and secrets

- Separate S3 buckets for private customer documents, quarantine, logs/audit and Terraform state as their ownership requires. Enable account/bucket Block Public Access, versioning, SSE-KMS, least-privilege bucket/key policy, lifecycle and deletion protection.
- Document events go to same-region SQS with DLQ, visibility timeout longer than processing, bounded retries, idempotency key, poison-message handling and reconciliation. Do not assume event delivery is exactly once or ordered.
- Use distinct KMS keys/aliases and administrators for database, documents, logs, secrets and Terraform state when separation is justified. Key deletion is a delayed, explicit two-person action.
- Secrets Manager stores production runtime secrets. ECS task definitions contain secret ARNs/references, never values. Record version/fingerprint and rotation owner; rotation must be rehearsed before enabling automatic rotation.
- Never place real secrets in Terraform variables/state output, GitHub logs, image layers, `.env` files, chat, screenshots or release documents.
- Configure AWS Backup only after checking overlap with native RDS/S3 protection. Define vault, retention, immutability, restore owner, cross-account copy and region policy explicitly; do not silently copy sensitive backup outside Hong Kong.

S3 Block Public Access can be applied at organization, account and bucket levels. See [S3 Block Public Access](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html).

### 4.5 Observability, audit, budgets, and support

- CloudWatch log groups have explicit Hong Kong region, retention, KMS key, PII allowlist/redaction and deletion policy. Application logs must not contain cookies, authorization headers, tokens, student names, document names or free-form payloads.
- CloudTrail organization trail and required data events flow to a security/log archive with integrity validation and restricted deletion. Decide AWS Config/Security Hub/GuardDuty based on threat model and budget rather than implying they are free.
- Dashboards and alarms cover ALB 4xx/5xx/latency, unhealthy targets, ECS desired/running/task exits, RDS capacity/connections/failover/backup, SQS age/depth/DLQ, S3/KMS denials, Cognito auth anomalies and application audit failure.
- SNS/PagerDuty/email recipients, severity, acknowledgement, escalation and test cadence must be named. An alarm without an owner is not an operational control.
- Budgets: actual and forecast alerts at agreed percentages, anomaly detection, service/account tags (`environment`, `system`, `owner`, `cost-center`, `data-classification`), monthly Finance review, and a separately approved hard stop policy. AWS Budgets notifications are not automatic service shutdown unless actions are intentionally configured.
- Record chosen AWS Support plan, authorized case openers, severity criteria and the rule that support payloads contain no unapproved PII.

## 5. Cloudflare public school-data configuration

### 5.1 Resources

- Company Cloudflare account and approved billing profile.
- One zone for the owned domain, or an approved partial/CNAME setup.
- Separate `school-public-staging` and `school-public-production` R2 buckets; never reuse for private documents or crawler working output.
- Production custom domain such as `data.example.com`. Disable the `r2.dev` URL in production; Cloudflare documents it as a rate-limited development endpoint.
- Separate bucket-scoped account tokens for publisher write and reconciliation read. No browser write token and no shared human token.
- Workers/Pages are **not required** for direct public R2 delivery. Introduce a Worker only for a measured, reviewed need such as content negotiation; it creates code, secrets, logging, deployment and cost surfaces.

R2 buckets are private by default. Production access should use a custom domain for caching/WAF; `r2.dev` is for development. See [R2 public buckets](https://developers.cloudflare.com/r2/buckets/public-buckets/) and [R2 authentication/tokens](https://developers.cloudflare.com/r2/api/tokens/).

### 5.2 Object and release contract

```text
releases/<release_id>/records.json
releases/<release_id>/manifest.json
releases/<release_id>/checksums.sha256
latest.json
```

`release_id` is an immutable content/build identifier, not `latest`. The publisher sequence is:

1. Compile only approved public records from the crawler's accepted/published output.
2. Validate schema, count, unique school/business keys, source URLs/provenance, date semantics, no forbidden fields/PII, and maximum file/object size.
3. Compute SHA-256 locally; manifest records schema version, release ID, generation time, object byte count, record count, hashes, prior release and compatibility version. It contains no workstation path, username, secret, private URL, review queue or internal error.
4. Upload immutable release objects with `Content-Type`, `Content-Encoding` if used, and long-lived immutable `Cache-Control`.
5. Read back/head objects, recompute or compare trusted checksums and validate manifest/counts from the public custom domain.
6. Write the small `latest.json` pointer **last**, with a short TTL/no stale-on-error policy chosen for the UI. Never overwrite an immutable release path.
7. Append a release receipt in the authoritative AWS audit/control plane; Cloudflare objects are not the approval record.

ETag must not be treated as a universal SHA-256: multipart R2 ETags have multipart semantics. Store an explicit SHA-256 in the manifest and optionally object metadata. See [R2 upload ETags](https://developers.cloudflare.com/r2/objects/upload-objects/).

### 5.3 Cache, CORS, lifecycle, and security

- Cache only `GET`/`HEAD` to `/releases/*` and `latest.json`; bypass requests carrying `Authorization`, `Cookie`, unexpected query strings, non-GET methods or private path prefixes.
- Immutable objects: edge/browser TTL can be long because URL changes per release. `latest.json`: short edge/browser TTL appropriate to publication SLA; purge only this pointer on publication if required.
- Enable Smart Tiered Cache only after checking plan and measured benefit. JSON is not cached by default in some configurations, so create a narrowly matched Cache Rule rather than a zone-wide Cache Everything rule.
- CORS: if the AWS app fetches data in the browser, allow only exact production/staging origins and `GET`/`HEAD`; no `*` with credentials; do not permit write methods/headers. Prefer server-side AWS fetch when browser CORS is unnecessary.
- Set explicit MIME, `nosniff`, content-disposition and CSP/frame behavior where applicable. Public means anyone can download and replicate the dataset; WAF is abuse control, not confidentiality.
- Lifecycle/retention: keep enough immutable releases for rollback, correction history and reproducibility; exact retention is a Product/Legal decision. Lifecycle may delete only after manifest/audit retention and rollback windows. Abort incomplete multipart uploads; test that locks and lifecycle do not conflict.
- Use rate limiting/bot controls only with a measured abuse threshold and avoid blocking legitimate public consumers. Log fields and retention need privacy review even for public objects.
- Reconciliation job compares R2 inventory/listing against authoritative release receipts and checksums; orphan, missing, hash mismatch or unexpected mutable overwrite ends publication in `needs_human`.

Cloudflare requires a custom domain for R2 cache and notes JSON is not cached automatically; Cache Rules control eligibility/TTL. CORS changes may require cache purge before new headers appear. See [R2 cache](https://developers.cloudflare.com/cache/interaction-cloudflare-products/r2/), [Cache Rules](https://developers.cloudflare.com/cache/how-to/cache-rules/), and [R2 CORS](https://developers.cloudflare.com/r2/buckets/cors/).

## 6. Current Vercel inventory without reading secrets

The repository currently contains only a minimal `vercel.json` (`framework`, build/dev/install commands), and `next.config.ts` has no self-hosting configuration. That does **not** prove the Vercel project has no dashboard-only environment variables, domains, integrations, Blob, Queues, Cron, log drains, protection, analytics or team settings.

The inventory owner should collect metadata under read-only authorization:

| Area | Record without secret values |
| --- | --- |
| Project/team | opaque project/team IDs, owners, plan, Git connection, production branch, root directory, Node/pnpm versions |
| Deployments | production/preview URLs, Git SHA, build command, function regions, redirects/rewrites/headers, rollback target |
| Environment variables | **name only**, target environment, branch scope, encrypted/sensitive flag, last-updated date, owner, build-time/runtime/public classification |
| Domains/DNS | hostname, registrar, nameserver/DNS provider, record type/name/TTL, certificate status, current project binding |
| Integrations | provider/product, resource IDs, data class, owner, target replacement; never token/value |
| Blob/Edge Config/KV/Postgres | store name/ID, access class, object/row counts and bytes, region, project bindings, export owner |
| Cron | path, schedule, timezone, enabled state, duration, concurrency/idempotency/lock semantics; never `CRON_SECRET` |
| Queues/Workflow | topic/group name, region, retention, schema/version, outstanding/in-flight counts, consumer and idempotency behavior |
| Observability | log drains, analytics, alerts, retention, destination, dashboards and incident links |

Use Vercel dashboard or a read-only owner-operated export. `vercel env ls`/dashboard can enumerate variable metadata; do **not** run `vercel env pull`, print values, or commit a generated `.env`. For migration, the secret owner re-enters/rotates each approved value directly into Secrets Manager or CI and signs a name/fingerprint receipt. `NEXT_PUBLIC_*` values are embedded at build time and must be reviewed as public, not migrated as secrets.

Vercel documents environment listing/export operations, project-transfer exclusions, Blob listing/download, Cron, and Queues. Review [Vercel environment CLI](https://vercel.com/docs/cli/env), [project transfer inventory](https://vercel.com/docs/projects/transferring-projects), [Vercel Blob CLI](https://vercel.com/docs/cli/blob), [Cron management](https://vercel.com/docs/cron-jobs/manage-cron-jobs), and [Vercel Queues](https://vercel.com/docs/queues). Project transfer is not the migration mechanism, but its exclusion list is a useful inventory: integrations, logs, log drains, Blob and some configuration require separate treatment.

## 7. Next.js self-hosting requirements

The first AWS image must be built from the same pinned source/dependencies as the Vercel comparison build. Prefer Next.js `output: 'standalone'` in a multi-stage Docker image, include required `public` and `.next/static` assets, run as non-root, and start the standalone server or `next start` according to the chosen artifact contract. Do not rebuild separately for each ECS task.

Required multi-instance decisions:

1. One immutable Git SHA, image digest and Next.js build ID across all tasks in a deployment.
2. Set a stable `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` at build for all instances of that build. It is base64 AES key material; manage it as a release secret and coordinate retention during rollback/mixed-version windows.
3. Set `deploymentId`/version-skew behavior so clients do not combine assets/actions from incompatible rolling deployments.
4. Audit whether the app uses ISR, `use cache`, `revalidateTag`, image optimization or filesystem cache. Local ephemeral caches diverge across Fargate tasks. Either implement shared cache/tag coordination or deliberately disable/limit cross-request caching until tested.
5. ALB/CDN must respect Next.js `Cache-Control`. Never cache authenticated dynamic HTML/API or responses using cookies/authorization.
6. Reverse-proxy limits, body/upload size, timeouts, streaming, `X-Forwarded-*`, trusted host/origin, client IP, secure-cookie detection and absolute callback URLs require integration tests.
7. Graceful drain: readiness becomes false, new traffic stops, in-flight requests finish within bounded timeout, process exits, and queued/background work is not lost or duplicated.

Next.js says a single `next start` supports framework features, but multi-server deployments need a common Server Actions encryption key, deployment identifier, shared cache and cache-tag coordination. See [Next.js self-hosting](https://nextjs.org/docs/app/guides/self-hosting) and [deploying to platforms](https://nextjs.org/docs/app/guides/deploying-to-platforms).

## 8. Environment-variable migration contract

Classify every variable before mapping it:

| Class | AWS destination | Migration rule |
| --- | --- | --- |
| Public build value (`NEXT_PUBLIC_*`) | Approved CI build arg/environment | Treat as public; changing it requires rebuild |
| Non-secret runtime configuration | ECS task definition/SSM parameter | Version and promote with release |
| Runtime secret | Secrets Manager -> ECS secret reference | Human/automation writes value directly; app role reads exact ARN only |
| Build secret | Ephemeral CI secret mount | Must not remain in layer, image history, logs or SBOM |
| Database/auth signing secret | Secrets Manager with dedicated owner/rotation | Prefer fresh production secret; dual-read only with explicit bounded plan |
| Vercel platform-provided value | Explicit AWS replacement | Do not blindly copy `VERCEL*`; remove platform assumptions |

For each name, record source owner, consumer, build/runtime phase, environment, data class, rotation/restart behavior, required/optional, fail-closed behavior, and target secret ARN/parameter name. Validate using names, configuration schema, and fingerprints only. Missing required production configuration must stop startup; it must not fall back to mock, local JSON, Neon, Vercel or a default tenant.

## 9. Migration sequence from Vercel to AWS

### Phase A: decisions and read-only discovery

1. Approve the data classification, hostname/DNS owner, AWS account topology, Cognito custom-domain exception, monthly budget, RPO/RTO and migration owners.
2. Inventory Vercel metadata from Section 6 and current DNS/registrar. Confirm whether Vercel Blob, Postgres/KV/Edge Config, Queues, Workflow, Cron, Functions, integrations or log drains exist.
3. Inventory current data stores and mutable writes. The current repository still describes Neon-backed crawler mutable state and table auto-creation; production migration must replace auto-DDL and plan authoritative data migration separately.
4. Define objective acceptance and rollback thresholds: auth success, error rate, P95, DB pool, queue depth, audit completeness, cross-tenant negative tests and data reconciliation.

### Phase B: company control plane

5. Register/secure AWS Organization accounts and Cloudflare company account; assign roles/MFA/contacts/billing/budgets/support.
6. Import or create Terraform remote-state control with encryption, locking/versioning, least privilege and break-glass ownership.
7. Add Cloudflare zone/partial zone and reproduce DNS records if nameserver change is approved. No traffic change yet.

### Phase C: non-production target

8. Build the pinned standalone Next.js container and push immutable digest to non-production ECR.
9. Provision non-production VPC, ECS/ALB/WAF, RDS, Cognito, private S3/KMS/SQS, Secrets and logs from reviewed IaC.
10. Replace Vercel-only assumptions and Neon/serverless adapter behavior with approved production adapters; no production DB write.
11. Test multi-task Server Actions, cache/revalidation, cookies, callback URLs, health/drain, rolling deployment, scanner/job idempotency, RLS/pool reuse and restore.
12. Build the public R2 staging publisher and test immutable release/checksum/latest contract with synthetic/public data only.

### Phase D: production shadow and data rehearsal

13. Produce exact production Terraform plan, image digest, environment-name map, migration checksum, DNS payload and cost estimate. Human approval is invalid if any payload changes.
14. Apply/provision only after exact approval. Keep production ALB on a non-public validation hostname or restricted access.
15. Rehearse Neon/PostgreSQL data migration or logical copy in isolated targets: schema/extensions/roles/RLS, counts, checksums, sequences, timestamps, query plans, delta/cutover and rollback. No live copy is implied by this document.
16. Publish one approved crawler release to R2, verify public schema/hash/count/cache/CORS/takedown and prove the AWS app can read a pinned release and fail safely when `latest.json` is unavailable or malformed.

### Phase E: canary and cutover

17. Add AWS callback/logout URLs and production secrets; deploy one approved image digest; run synthetics, empty-tenant and named pilot tests.
18. Freeze or bound mutable migration writes according to the approved data plan; perform final copy/delta and verify. Queue/cron producers must have one active owner to avoid duplicates.
19. Lower TTL beforehand, then change only `app.example.com` to the ALB. Keep Vercel deployment, domain binding, previous DNS value and compatible data path available for the approved rollback window.
20. Observe objective gates continuously. Do not route a percentage of authenticated users to two versions unless sessions, writes and migrations are explicitly compatible.

### Phase F: retire Vercel

21. After the observation/rollback window, disable Vercel Cron/Queues/producers first, confirm drain/outstanding state, then remove domain binding and Git auto-deploy. Vercel notes that instant rollback does not update active Cron configuration, so Cron needs an independent shutdown check.
22. Export/retain permitted logs, invoices, build/deployment evidence and required Blob/data objects; rotate secrets/tokens that Vercel could access; revoke integrations and team members.
23. Delete/cancel Vercel resources only under a separate exact inventory and recovery-approved action. Registrar transfer is unnecessary unless independently decided.

## 10. Rollback model

Rollback is legal only while schema and writes remain backward-compatible or while a tested restore/forward-repair path exists.

| Failure | Immediate action | Rollback evidence |
| --- | --- | --- |
| ALB/ECS health, 5xx or latency breach | Stop deployment/cutover; ECS circuit breaker to prior digest; DNS back to recorded Vercel value if data-compatible | Previous image/DNS, health, error and data-write compatibility receipt |
| Login/cookie/callback failure | Stop new traffic; preserve exact failing request metadata without tokens; revert DNS/config | Both callback allowlists, cookie tests, session/version evidence |
| DB checksum/schema/RLS mismatch | Do not cut over; isolate target; no ad-hoc production SQL | Migration checksum, catalogue fingerprint, source/target count/hash |
| Cross-tenant/PII exposure | Stop pilot; revoke access/session; preserve redacted evidence; incident response | Negative tests, audit trail, affected-scope decision |
| R2 stale/hash/public-schema failure | Keep last good `latest.json`; do not delete prior immutable release | Release receipt, manifest SHA-256, read-back validation |
| Duplicate cron/queue work | Disable new producer; use idempotency/reconciliation; no blind retry | Producer ownership, message IDs, outstanding/in-flight counts |

DNS rollback alone is insufficient after incompatible writes. The cutover plan must declare the last reversible point and what changes terminal state from `rollback_available` to `forward_repair_or_restore`.

## 11. Required registrations and configuration checklist

### Register/purchase

- [ ] Company registrar/domain ownership and renewal controls confirmed.
- [ ] AWS paid management account, Organizations, production/non-production accounts and company payment/tax profile.
- [ ] AWS support plan decision and named authorized support contacts.
- [ ] Cloudflare company account, billing profile/plan, R2 subscription and at least two Super Administrators.
- [ ] GitHub organization/CI ownership and billing confirmed.
- [ ] Optional third-party alerting/email/SMS/OCR contracts separately reviewed for region, PII and cost.

### Configure before production apply

- [ ] Root/recovery MFA, Identity Center, contacts, permission sets, CI federation, SCPs and break-glass.
- [ ] Budgets/anomaly detection/tags, invoice recipients and monthly review.
- [ ] DNS inventory, zone owner, hostnames, ACM validation and TTL/rollback payload.
- [ ] VPC/subnets/endpoints/NAT decision, security groups and network logs.
- [ ] ECR/ECS/ALB/WAF, health/drain/circuit breaker/autoscaling and immutable release identity.
- [ ] RDS Multi-AZ, RLS role contract, backups/PITR/restore and monitoring.
- [ ] Cognito pool/client/MFA/callback/logout/session/revocation and custom-domain exception.
- [ ] Private S3/KMS/SQS/DLQ/scanner/Secrets and document recovery.
- [ ] CloudWatch/CloudTrail/log archive/alarms/runbooks/redaction/retention.
- [ ] R2 buckets/tokens/custom domain/cache/CORS/lifecycle/public manifest/checksum/takedown.
- [ ] Vercel inventory, secret-name map, Blob/Queue/Cron drain, domain and retirement plan.

## 12. Acceptance evidence

| Gate | Deterministic evidence |
| --- | --- |
| Ownership | Company aliases, account IDs, role/member list, MFA/contacts/billing status; no credential values |
| Infrastructure | Reviewed Terraform plan hash, policy checks, account/region/resource manifest and cost estimate |
| Container | Git SHA, image digest, build ID, SBOM, scan result, task-definition revision and config fingerprint |
| Next.js | Two-task Server Action/cache/tag/deployment-skew tests; drain and rollback test |
| Identity | Callback/logout, MFA/recovery, cookie, revoke/session-version and negative authorization tests |
| Database | Migration checksum, role/owner/RLS catalogue fingerprint, pool reuse, failover and restore drill |
| Private documents | Block-public/encryption/version/scan/DLQ/replay/recovery evidence |
| Public dataset | Public-schema/PII scan, record count, SHA-256 manifest, immutable paths, read-back, cache/CORS and takedown test |
| Vercel exit | Inventory completeness, producer drain, no requests after DNS window, secret rotation and retained rollback evidence |

No `pnpm lint` or `pnpm build` was run for this research document; repository rules prohibit those commands unless explicitly requested. No cloud account, Vercel project, secret, DNS record, database or deployment was accessed or changed.

## 13. Human approval points

| Decision/action | Required owner | Status |
| --- | --- | --- |
| Public dataset fields, provenance/licence, retention and takedown | Product + Privacy/Legal | `pending` |
| AWS account topology, seller/payment/tax and monthly cap | Founder + Finance | `pending` |
| Cloudflare plan, zone ownership and R2 public location acceptance | Founder + Privacy/Security | `pending` |
| `app` DNS-only and `data` proxied hostname model | Infrastructure + Security | `pending` |
| Cognito custom-domain global CloudFront / `us-east-1` certificate exception | Privacy/Security | `pending` |
| RPO/RTO, backup retention and single-region outage behavior | Operations + Data owner | `pending` |
| Exact Terraform plan/apply, migration, image deploy and DNS payload | Named release approvers | `pending` per payload |
| Vercel resource deletion/cancellation and secret revocation | Product + Operations + Finance | `pending` |

Approval of the architecture does not approve any account registration, purchase, resource creation, migration, upload, DNS change, deployment, or deletion. Each externally consequential action requires its exact payload, cost and rollback evidence.

## 14. Official source index and currency limits

Official pages checked on 2026-08-11:

- AWS: [root-user best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/root-user-best-practices.html), [IAM security best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html), [Organizations best practices](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_best-practices.html), [Billing setup](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/billing-getting-started.html), [alternate contacts](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-update-contact-alternate.html), [ACM DNS validation](https://docs.aws.amazon.com/acm/latest/userguide/dns-validation.html), [ECS deployment circuit breaker](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-circuit-breaker.html), [RDS Secrets Manager](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-secrets-manager.html), [S3 Block Public Access](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html), [Cognito custom domain](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-add-custom-domain.html).
- Cloudflare: [create account](https://developers.cloudflare.com/fundamentals/account/create-account/), [roles](https://developers.cloudflare.com/fundamentals/manage-members/roles/), [account tokens](https://developers.cloudflare.com/fundamentals/api/get-started/account-owned-tokens/), [R2 tokens](https://developers.cloudflare.com/r2/api/tokens/), [public buckets](https://developers.cloudflare.com/r2/buckets/public-buckets/), [R2 cache](https://developers.cloudflare.com/cache/interaction-cloudflare-products/r2/), [Cache Rules](https://developers.cloudflare.com/cache/how-to/cache-rules/), [CORS](https://developers.cloudflare.com/r2/buckets/cors/), [object upload/ETags](https://developers.cloudflare.com/r2/objects/upload-objects/).
- Vercel: [environment CLI](https://vercel.com/docs/cli/env), [project transfer inventory](https://vercel.com/docs/projects/transferring-projects), [domains](https://vercel.com/docs/domains), [Blob CLI](https://vercel.com/docs/cli/blob), [Cron management](https://vercel.com/docs/cron-jobs/manage-cron-jobs), [Queues](https://vercel.com/docs/queues).
- Next.js: [self-hosting](https://nextjs.org/docs/app/guides/self-hosting), [deploying to platforms](https://nextjs.org/docs/app/guides/deploying-to-platforms).

Vendor consoles, plans, regions, product limits and documentation can change. Before procurement or execution, re-open the dated official pages, obtain current quotes/DPAs, export the exact IaC plan and confirm `ap-east-1` availability for every selected feature. This document intentionally does not claim that creating an account or selecting a region automatically proves end-to-end Hong Kong residency.
