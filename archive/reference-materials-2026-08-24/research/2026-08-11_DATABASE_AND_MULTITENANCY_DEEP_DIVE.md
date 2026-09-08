# Database and Multitenancy Deep Dive

> Date checked: 2026-08-11 (Asia/Hong_Kong)  
> Scope: database platform, tenant isolation, recovery, capacity and second-tenant prerequisites  
> Status: research input only; it does not amend an accepted DEC, authorize cloud provisioning, execute a migration, move data, or enable a second tenant

## 1. Executive decision

Keep `DEC-053`'s architectural direction: **one private PostgreSQL deployment in Hong Kong, shared database/shared schema, non-null tenant keys, composite locality constraints, server-side authorization first and PostgreSQL RLS second**. Keep the accepted Release 1 database in `DEC-019`: **RDS for PostgreSQL Multi-AZ (one standby) in `ap-east-1`**. The pilot and expected 10-tenant workload do not yet provide measured evidence that Aurora's reader/scaling model, schema-per-tenant, or database-per-tenant would repay their operational and cost overhead.

Do not describe the current implementation as second-tenant ready. The local migrations have useful foundations, but there are material gaps:

1. `tianxing_app` is explicitly `NOBYPASSRLS`, yet no migration contains `FORCE ROW LEVEL SECURITY`. A table owner normally bypasses RLS; PostgreSQL documents `FORCE ROW LEVEL SECURITY` as the control that subjects the owner to policies. The application role must also remain a non-owner. [PostgreSQL row security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
2. Migration `008` discovers and protects only the tenant tables that exist when that migration runs. Later migrations add tenant tables and repeat policy setup selectively. A future table can therefore omit RLS unless a deterministic catalogue gate rejects it.
3. The policies compare `organization_id` to a transaction-local custom setting, but the repository has no live PostgreSQL integration evidence proving cross-tenant denial, missing-context denial, context reset after commit/rollback, owner-role behavior, or every table's policy state.
4. The current school base/snapshot model has `organization_id NOT NULL` on `schools_schools`, `schools_snapshots`, and snapshot records. That duplicates a supposedly global platform fact per tenant and does not implement `DEC-053`'s global base plus tenant-private overlay boundary.
5. Many actor columns reference global `identity_users(id)` alone. Where the actor must be a member of the row's organization, the invariant currently depends on a trigger/service check rather than a uniform composite membership FK. Each such field needs an explicit classification: global identity provenance or tenant-authorized actor.
6. Managed-service backup/PITR restores the whole instance/server/cluster, not one tenant. Shared-schema single-tenant recovery therefore needs a separate, tested selective-recovery workflow.

Neon is **not a production candidate** under the present Hong Kong boundary. Its official status documentation lists Singapore and Sydney as its Asia-Pacific AWS regions, not Hong Kong. [Neon status regions](https://neon.com/docs/introduction/status) Azure Database for PostgreSQL Flexible Server is available in Azure East Asia (Hong Kong), but Microsoft currently marks new zone-redundant HA deployments there as temporarily blocked; same-zone HA remains listed. It is a portability/exit candidate, not a reason to supersede the accepted AWS deployment without a new ADR, a live availability quote and full migration evidence. [Azure PostgreSQL regional availability](https://learn.microsoft.com/en-us/azure/postgresql/overview)

## 2. Problem, stakeholders and boundaries

### Problem

The system must add organizations without allowing identity, rows, relationships, files, jobs, search/vector results, exports, support access, recovery operations, or timing/count side channels to cross tenant boundaries. It must preserve the accepted Hong Kong data boundary and provide a recoverable path when one tenant asks for correction, export, termination or restore.

### Stakeholder outcomes

| Stakeholder | Required outcome |
| --- | --- |
| Customer Organization | Its data cannot be read, linked, indexed, exported, restored into, or affected by another tenant. |
| Advisor/Collaborator/Contractor | Access follows current membership, case scope, capability, expiry and redaction, not a browser claim. |
| Founder/Organization Owner | Can govern membership, sensitive export, retention and tenant-specific recovery with auditable evidence. |
| Platform Operator | Can operate infrastructure without standing access to tenant content; support access is explicit and expiring. |
| Security/Privacy | Every sensitive copy, backup, log and derived index remains within the approved Hong Kong boundary. |
| Engineering/Operations | Migrations, pool behavior, restore, reconciliation, capacity and rollback are deterministic and reproducible. |

### In scope

- PostgreSQL tenancy topology and database constraints.
- Tenant identity, membership, RLS, connection context and privileged jobs.
- Database platform comparison, recovery, capacity and cost drivers.
- Tenant boundaries outside relational rows: S3, outbox, cache, search/vector, export and support grants.
- Migration/rollback and the release gate before a second tenant.

### Out of scope

- Provisioning or purchasing a service, running a migration, changing production data, or accepting a legal/privacy risk.
- Product price, margin or SLA promises.
- Inventing the open subscription, support, retention, export, outage, promotion or termination semantics in `DEC-060`.
- Treating document bytes as PostgreSQL storage; accepted architecture places them in S3.

## 3. Authority and local implementation evidence

### 3.1 Accepted authority

The relevant decisions form one coherent chain:

- `DEC-004..011`, `026..032`, `042..046`: immutable IDs, tenant-local relationships, controlled state, collaborator scopes, optimistic concurrency, transaction + audit + outbox, retention and export controls.
- `DEC-018/019/022/037/038`: sensitive copies in Hong Kong; RDS PostgreSQL Multi-AZ in `ap-east-1`; planning targets DB RPO 5 minutes/RTO 4 hours; seven-day backups; monthly/quarterly restore drills.
- `DEC-051..059`: organization/membership model, A/B/C data classes, shared-schema RLS, three product planes, modular ownership, isolation/recovery evidence, and tenant-private knowledge namespaces.
- `DEC-060`: intentionally open business/legal semantics before a second subscriber.

The implementation plan correctly says Release 1 remains one organization and that adding `organization_id` is not itself a second-tenant launch. Its Phase 3/4 sequence remains synthetic staging -> empty production tenant -> 1-3 reconstructed cases -> observation -> at most 5-10 active pilot cases. That sequence is a first-tenant evidence lane, not the second-tenant gate.

### 3.2 What exists locally

| Evidence | Present behavior | Assessment |
| --- | --- | --- |
| `202608021330_001_expand_identity_access.sql` | Global `identity_users`; `access_organizations`; membership unique by `(organization_id,user_id)`; role/session composite membership FKs; partial unique allows only one active organization. | Good first-tenant identity foundation. The one-active-organization index must be removed only in a reviewed expand migration before a second tenant. |
| Domain migrations | Tenant-owned rows normally use `organization_id uuid NOT NULL`; many parent-child links include `organization_id` in composite FK/unique constraints. | Strong locality pattern, but catalogue verification must cover every tenant table and relationship. |
| `202608030030_008_expand_application_database_role.sql` | Creates `tianxing_app` with `NOBYPASSRLS`; grants `rds_iam`; enables RLS and creates a policy for then-existing tables containing `organization_id`. | Necessary but incomplete: no `FORCE RLS`; dynamic discovery at one migration point is not a lasting invariant. |
| Later migrations | Some newly added tables receive explicit RLS policy setup. | Selective repetition is drift-prone. Require a final-state catalogue assertion. |
| `modules/shared/db.ts` | Requires canonical organization/actor UUIDs, starts a transaction, calls `set_config('app.organization_id', ..., true)` and `set_config('app.actor_user_id', ..., true)`, then commits/rolls back and releases. Host is restricted to `*.ap-east-1.rds.amazonaws.com`. | Correct direction and fail-closed placement. It needs real PostgreSQL tests and makes Azure/Aurora/Neon a deliberate adapter/config migration, not a connection-string swap. |
| Access/domain services | Repository methods own the transaction boundary so authorization facts and writes can be read/locked atomically. | Preserve. Privileged jobs must use equivalent trusted context or narrowly scoped service interfaces. |
| Migration planner | Ordered immutable SQL, checksums, drift checks, separate migration URL/role and expand/contract policy. | Preserve. Add tenant-catalogue and policy fingerprint checks. |

### 3.3 Immediate schema findings

1. **RLS coverage is not complete by construction.** A migration-time loop cannot protect tables created later. Add a final migration and test that enumerate all ordinary tables and classify each as `global`, `tenant_owned`, `derived_tenant`, or `privileged_control`; unclassified tables fail.
2. **`FORCE RLS` is absent.** Add it to every tenant-owned/derived-tenant table and verify `relrowsecurity=true`, `relforcerowsecurity=true`, expected policy expression, owner != app role, and app role `rolbypassrls=false` from PostgreSQL catalogues.
3. **Global school facts are tenant-shaped.** Split immutable global snapshot/revision records from tenant overlays, shortlist/case links, private notes and knowledge. Do not use null/sentinel `organization_id` to represent global data.
4. **Identity actor locality requires classification.** A global User may belong to multiple organizations, so `identity_users` should remain global. An audit provenance field can reference global user ID, but an approver/assignee/requester whose authority is tenant-local should also bind membership/organization or be transactionally checked against active membership.
5. **Missing context should deny, not error unpredictably.** The current `current_setting(..., true)` yields null if missing, which causes equality to be null/false. Preserve that fail-closed behavior, but map rejected operations to stable application error contracts without revealing row existence.

## 4. Recommended tenancy topology

### 4.1 Comparison

| Pattern | Isolation | Operational/migration cost | Single-tenant restore | Noisy-tenant control | Verdict |
| --- | --- | --- | --- | --- | --- |
| Shared DB, shared schema + RLS | Logical isolation; strongest only with server authz, composite locality, forced RLS and unprivileged role | Lowest schema drift and connection overhead; one migration fleet | Hard: whole-instance PITR then selective extraction/replay | Shared CPU/I/O/locks; needs quotas, admission control and observability | **Default through measured 50-tenant scale** |
| Shared DB, schema per tenant | Namespace separation, but shared instance/roles/resources remain | Migration fan-out, `search_path` hazards, extension/schema drift, pooled connection state | Easier logical selection than shared tables, still whole-instance physical restore | Still shared compute/I/O; per-schema stats possible | Reject as default; little benefit for this workload |
| Database per tenant on one server/cluster | Stronger namespace/connection boundary | Connection pools and migrations multiply; cross-tenant control queries become harder | `pg_dump`/`pg_restore` can target one database, but managed PITR still restores whole server/cluster | Same host resource contention; database-level connection budgets | Consider only for a small regulated/outlier tier |
| Instance/cluster per tenant | Strongest blast-radius, encryption and maintenance isolation | Highest fixed cost, fleet management and release coordination | Clearest tenant-level physical recovery | Strongest resource isolation | Exception for contractual isolation, very large/noisy tenant or incompatible maintenance/retention |

### 4.2 Recommended hybrid

Use **shared DB/shared schema as the standard tier**. Add a placement abstraction before it is needed, not a premature fleet:

```text
CustomerOrganization
  -> TenantPlacement(placement_id, class, region, database target, policy version)
  -> shared-hk-primary (default)
  -> dedicated-hk-database/cluster (future exception)
```

The placement record is control-plane metadata, never accepted from a browser. Module repositories resolve it after trusted organization selection. A future dedicated placement must use the same logical schema/module contracts and event versions; cross-placement joins are forbidden. This keeps an exit path without paying per-tenant database/cluster cost now.

Trigger a dedicated placement review only when measured evidence shows one of:

- a tenant's sustained CPU, I/O, locks, WAL, connections or queue consumption breaches the shared budget despite throttling;
- a contractual encryption key, maintenance window, backup retention or restore-isolation requirement cannot be met in shared storage;
- a tenant's data size makes selective restore miss the approved RTO;
- incompatible extensions/schema versions are genuinely required;
- repeated security review rejects logical isolation despite deterministic evidence.

## 5. Tenant identity, membership and authorization model

### 5.1 Entities and identity rules

| Entity | Identity/invariant |
| --- | --- |
| `User` | Global immutable UUID; unique `(provider,provider_subject)`; email is an attribute, not relationship key. One provider identity may join multiple organizations. |
| `CustomerOrganization` | Immutable UUID and stable legal/customer reference; display name is not identity. Duplicate/legal-merger rules require human governance. |
| `OrganizationMembership` | Unique `(organization_id,user_id)`; status is explicit; invitation, activation, disable and termination are audited revisions. |
| `RoleBinding` | Tenant-local and tied to membership; no role from browser/IdP token is business authority. |
| `Session` | Captures active organization, membership and `session_version`; organization switch creates/rotates trusted session state and re-authorizes membership. |
| Case/collaborator/task grant | Always tenant- and resource-local, current-time evaluated, capability-specific and revocable. |
| `SupportGrant` | Separate from membership; target tenant, scope, capability, reason, approver(s), incident/request, expiry and audit required. Exact semantics remain `DEC-060`. |

Before a second tenant, close: organization stable/duplicate rules, owner versus Founder terminology, last-owner protection and transfer, multi-organization switch, invite/revoke/session invalidation, and disabled/terminated membership effects on jobs and exports.

### 5.2 Request flow

```text
provider identity
  -> opaque HK session lookup
  -> active User/session_version
  -> active organization + membership + role
  -> case/scope/capability/expiry policy
  -> BEGIN
  -> SET LOCAL trusted organization + actor context
  -> authorized reads/locks + mutation + audit + outbox
  -> COMMIT/ROLLBACK
  -> context disappears with transaction
```

The browser may request an organization switch but cannot assert the resulting organization context. Every request resolves it from server-side session/membership state.

## 6. PostgreSQL isolation contract

### 6.1 Tenant keys and relationships

- Every tenant-owned authoritative or derived row has `organization_id uuid NOT NULL`.
- Every tenant parent exposes a unique key containing its ID and organization, normally `UNIQUE(id, organization_id)`.
- Every tenant-to-tenant relationship includes the child `organization_id` in its FK. No relationship relies only on an ancestor join.
- Uniqueness with tenant meaning begins with `organization_id`, including human case number, external idempotency key, object key, export token and private knowledge slug.
- Global relations are explicit global tables with a platform owner/revision contract. Tenant extensions use a non-null organization key plus a global record/revision FK.
- Partition keys, indexes and query predicates should begin with/selectively include `organization_id` when supported by the workload; index design is measured, not mechanically duplicated.

### 6.2 RLS

For every tenant table:

```sql
ALTER TABLE tenant_table ENABLE ROW LEVEL SECURITY;
ALTER TABLE tenant_table FORCE ROW LEVEL SECURITY;

CREATE POLICY tenant_boundary ON tenant_table
  FOR ALL TO tianxing_app
  USING (organization_id = nullif(current_setting('app.organization_id', true), '')::uuid)
  WITH CHECK (organization_id = nullif(current_setting('app.organization_id', true), '')::uuid);
```

The exact expression should be standardized and fingerprinted. `FORCE` is still required even when the application role is not owner because ownership can drift during restore/migration. Superusers and roles with `BYPASSRLS` always bypass policies; such roles must never serve ordinary requests. PostgreSQL also warns that referential integrity checks bypass RLS and can create covert channels, which is another reason to use composite locality and stable non-disclosing API errors. [PostgreSQL row security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)

### 6.3 Connection pool and transaction-local context

PostgreSQL documents that `set_config(name,value,true)` applies only to the current transaction. [PostgreSQL configuration functions](https://www.postgresql.org/docs/current/functions-admin.html) Preserve the current runner's `BEGIN -> set_config(...,true) -> work -> COMMIT/ROLLBACK -> release` order and add these tests against actual PostgreSQL:

1. no context: select/update/insert/delete expose or affect zero tenant rows;
2. tenant A context: A works, B ID guessing returns no disclosure;
3. commit then connection reuse for B: no A residue;
4. rollback/error then reuse for B: no A residue;
5. nested/re-entrant repository cannot change organization mid-transaction;
6. app role cannot `SET ROLE`, create policy, alter table or access ungranted global/control tables;
7. table owner and migration role behavior is tested separately, never used as app evidence.

RDS Proxy can reuse a backend after each transaction, but AWS documents that PostgreSQL `SET`/`set_config` can pin sessions and reduce multiplexing. If Proxy is introduced, monitor pinned/database/client connection metrics and load-test the exact transaction-local pattern. Do not enable a pinning filter as a correctness shortcut; AWS says it is safe only when every transaction independently establishes all needed state. [RDS Proxy transactions](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.howitworks.html) [RDS Proxy pinning](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/rds-proxy-pinning.html) [RDS Proxy configuration](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy-best-practices.configuration.html)

### 6.4 Privileged jobs and operations

- Migration, restore and break-fix identities are separate, non-application roles with time-bounded credentials, exact payload approval and audit.
- Ordinary outbox/index/OCR/scanner/retention workers process one organization per lease/transaction and set the same trusted local context.
- Cross-tenant platform jobs first enumerate opaque organization IDs from a control relation, then open independent bounded transactions. No one transaction carries a wildcard tenant.
- A privileged job must not disable RLS to simplify iteration. If a true all-tenant operation is necessary, use a dedicated function/interface with fixed query shape, least grants, row/result limits, audit and reconciliation.
- Backup tools need deliberate privileges: PostgreSQL notes that backup users must avoid silently filtering rows through RLS. Backup success is independently reconciled by tenant counts/hashes. [PostgreSQL row security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)

## 7. Boundaries beyond database rows

| Boundary | Required tenant invariant |
| --- | --- |
| S3 | Opaque key includes placement/org prefix or unguessable tenant-scoped mapping; DB metadata is authoritative; presigned intent binds organization, object/version, action, size/type/checksum and short expiry. Bucket policy is not case authz. |
| Outbox/jobs | Every event has organization ID, aggregate ID, event/policy/schema version and idempotency key; worker re-authorizes current state for sensitive effects. Queue message tenant is untrusted until DB resolution. |
| Cache | Key includes organization + actor/policy/resource version where relevant; no shared personalized cache; revoke/expiry correctness must not depend only on invalidation. |
| Search/vector | Global and tenant namespaces are separate. Tenant retrieval filters are server-injected and enforced before returning candidates; source permissions are rechecked. Index is rebuildable, not authority. |
| Export | Export request, approval, snapshot, file, object key, download intent and purge receipt all bind organization. Collaborators never gain export through scope/capability composition. |
| Support | Target organization and expiring grant are explicit on each use; no standing PlatformOperator tenant-content role. |
| Logs/metrics | Organization uses opaque ID only; no PII/free text/query; control-plane aggregate metrics must prevent small-count/timing disclosure. |
| Quota/rate/cost | Counters and budgets are organization-scoped plus global safety caps. Atomic admission occurs before expensive work; delayed usage reconciliation corrects drift. |

For vector/search, do not assume metadata filtering alone provides good latency or recall. Test skewed tenant sizes, deleted/revoked sources, stale indexes and a noisy large tenant. At 50 tenants, a shared physical index with mandatory organization filter can remain viable if plans/recall pass; separate physical indexes are a measured isolation optimization, not an authorization boundary.

## 8. Managed PostgreSQL comparison

### 8.1 Hard gates

Any production candidate must prove all of:

1. primary, standby/storage, backups, snapshots, logs, keys, support data and restore target remain in Hong Kong;
2. private networking from the approved sensitive runtime;
3. PostgreSQL features required by migrations, RLS, IAM/auth, extensions and observability;
4. tested RPO/RTO, failover, connection recovery and restore/reconciliation;
5. dated official price/calculator receipt for the exact region/SKU/topology;
6. migration and rollback from the accepted RDS deployment without long dual-write.

### 8.2 Service matrix

| Service | Hong Kong/HA fit | Recovery/scale | Cost drivers | Verdict |
| --- | --- | --- | --- | --- |
| RDS PostgreSQL Multi-AZ DB instance | Accepted `ap-east-1`; synchronous standby in another AZ; standby does not serve reads | Automatic failover; PITR/snapshot creates a new instance; vertical scaling/read replica options | Primary + standby instance, duplicated Multi-AZ storage/I/O characteristics, excess backup, Proxy, monitoring/support | **Release 1 baseline and default** |
| Aurora PostgreSQL | Available only if exact `ap-east-1` engine/SKU is reverified; distributed storage across 3 AZs; HA requires reader instance(s) for fast promotion | Up to 15 readers; continuous backup/PITR to new cluster; Standard charges I/O, I/O-Optimized shifts cost to compute/storage | Writer + reader/ACUs, storage, I/O or I/O-Optimized premium, backup, Proxy/support | Revisit only on measured failover/read/connection/I/O need |
| Azure PostgreSQL Flexible Server | East Asia is Hong Kong. Same-zone HA listed; new zone-redundant HA currently marked temporarily blocked (`$`) with special existing-server note (`**`) | PITR creates a new server; 7-35 day retention; restored server is not automatically HA; geo backup leaves region and conflicts with current boundary | Primary/standby compute, provisioned storage/IOPS, excess backup, network/monitoring/support; quote required | Portability candidate, not present replacement |
| Neon | Official region list does not include Hong Kong | Serverless compute, branching/PITR, pooled connections; product-level HA claims do not cure residency failure | CU-hours, DB/history storage, branches, transfer, plan/support | **Ineligible until independently verified HK region and full boundary** |

AWS confirms a one-standby RDS Multi-AZ deployment uses a standby in another AZ and does not serve read traffic. [RDS Multi-AZ](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.html) RDS pricing is pay-as-you-go and Multi-AZ storage/write behavior differs because updates are synchronously replicated. [RDS PostgreSQL pricing](https://aws.amazon.com/rds/postgresql/pricing/)

Aurora charges instances plus storage and, under Standard, per-request I/O; I/O-Optimized removes I/O charges at higher compute/storage rates. Its official pricing guidance says the break-even depends on I/O as a share of total database spend, so it must be modeled from measured I/O rather than tenant count. [Aurora pricing](https://aws.amazon.com/rds/aurora/pricing/) Aurora restore creates a new cluster; latest restorable time is typically within five minutes. [Aurora backup and restore](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Managing.Backups.html)

Azure automatically backs up Flexible Server, offers 7-35 day retention, estimates backup RPO up to five minutes, and creates a new same-region server for PITR. A restored HA source is restored initially as a single instance and HA must be configured afterward. Geo-redundant backup asynchronously copies to a paired region, so it is disallowed by the present Hong Kong-only decision. [Azure backup and restore](https://learn.microsoft.com/en-us/azure/postgresql/backup-restore/concepts-backup-restore) Azure's public price page explicitly says prices are estimates and directs customers to the calculator for offer/region-specific prices. [Azure PostgreSQL pricing](https://azure.microsoft.com/en-us/pricing/details/postgresql/flexible-server/)

### 8.3 Portability cost

The application is PostgreSQL-oriented, but a vendor move is not zero-cost:

- `modules/shared/db.ts` currently validates an RDS `ap-east-1` hostname, database/user/application names and SSL shape;
- IAM auth role/proxy/private DNS/KMS/logging/backup/IaC and incident runbooks are AWS-specific;
- available PostgreSQL versions/extensions/parameters and failover endpoints differ;
- Azure restore networking/config/HA is not automatically copied; Neon pooling/roles/branching differ;
- RLS/composite FKs remain portable SQL, which is a reason to avoid vendor-only authorization.

A vendor switch therefore needs a new adapter/config contract, clean migration rehearsal, counts/hashes, policy catalogue, performance/failover/restore evidence and an application rollback window. Do not weaken hostname fail-closed behavior into accepting arbitrary PostgreSQL hosts; replace it with an explicitly approved placement/provider allowlist.

## 9. Recovery and tenant restore

### 9.1 Distinguish recovery mechanisms

| Mechanism | Restores | Does not provide |
| --- | --- | --- |
| Multi-AZ failover | Availability after instance/AZ failure | Historical rollback, tenant correction or backup |
| PITR/snapshot | Whole instance/server/cluster at a past point into a new target | In-place rollback or native one-tenant restore |
| Schema/app rollback | Compatible behavior/version | Lost/corrupted data recovery |
| Audit/outbox reconciliation | Evidence and side-effect convergence | Authoritative data backup |
| Tenant selective recovery | Chosen tenant facts merged through controlled process | Automatic managed-service feature |

AWS states that an RDS snapshot covers the entire DB instance, not an individual database, and restore creates a new instance. [RDS snapshot restore](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_RestoreFromSnapshot.html) Microsoft likewise states that single database/table restore is not directly supported by Flexible Server PITR. [Azure backup and restore](https://learn.microsoft.com/en-us/azure/postgresql/backup-restore/concepts-backup-restore)

### 9.2 Shared-schema single-tenant restore runbook

1. Freeze tenant mutations and expensive jobs; preserve exact incident/cutoff and current hashes.
2. PITR the whole source to a **new private isolated HK recovery target** using a restore role, never the live endpoint.
3. Apply the same schema/policy build and verify ownership, RLS catalogue, counts and checksums.
4. Extract only the tenant's A/B authoritative rows in dependency order to encrypted, access-controlled HK staging artifacts; exclude other tenant data and redact operational logs.
5. Diff current versus recovered state using stable IDs, versions, counts, hashes, legal hold and external side-effect receipts.
6. Human owner approves an exact corrective plan. Apply through owning module/recovery interface with expected versions, idempotency, audit and outbox compensation. Do not bulk overwrite current rows behind services.
7. Rebuild the tenant's C-class cache/search/vector projections; reconcile S3 versions and downstream receipts.
8. Resume access only after unexplained differences are zero and negative cross-tenant checks pass; destroy recovery artifacts under an approved retention receipt.

This runbook must be exercised before the second tenant because shared-schema topology trades cheaper operations for harder selective recovery. At Scale, measured restore/export duration is a possible trigger for dedicated placement.

## 10. Replaceable workload and cost model

### 10.1 Shared planning assumptions

These are **replaceable planning inputs, not forecasts or promises**:

| Profile | Tenants | Users | Active / retained cases | Logical documents | Monthly new/version sensitivity | Internet download sensitivity |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Pilot | 1 | 10 | 10 / 100 | up to ~50 GB | 10% of stock | 20% of stock |
| Growth | 10 | 100 | 100 / 1,000 | ~500 GB | 10% | 20% |
| Scale | 50 | 500 | 500 / 5,000 | ~2.5 TB | 10% | 20% |

Document bytes belong in S3; these figures drive metadata/version/scan events and end-to-end platform cost, not RDS allocated storage directly.

Structured DB, vector, logs, OCR and request rate must be measured independently. Use these deliberately broad low/high envelopes only to size tests:

| Driver | Pilot low-high | Growth low-high | Scale low-high | Measure that replaces assumption |
| --- | ---: | ---: | ---: | --- |
| Structured PostgreSQL incl. indexes | 5-20 GiB | 25-100 GiB | 100-500 GiB | `pg_database_size`, relation/index/WAL growth |
| Vector/search derived data | 0.5-5 GiB | 5-50 GiB | 25-250 GiB | approved chunk count, dimension/type, index amplification |
| HK app/audit/log ingest | 10-50 GiB/month | 50-250 GiB/month | 250 GiB-1 TiB/month | allowlisted event bytes and retention |
| OCR/scan share of new document bytes | 5%-30% | 5%-30% | 5%-30% | MIME/class policy and observed queue receipts |
| General API steady/short peak | 5/50 rps | 20/200 rps | 100/1,000 rps | route histogram and concurrency |
| Client / pooled DB backend concurrency | 20-100 / 5-20 | 100-500 / 20-80 | 500-2,000 / 50-250 | pool wait, backend count, pinned sessions |

The upper request/concurrency figures are stress envelopes, not predicted usage.

### 10.2 Cost equation

Do not compare a single headline monthly number. Maintain a dated calculator artifact with:

```text
DB monthly = HA compute
           + provisioned/consumed DB storage and I/O
           + backup/PITR history excess
           + connection proxy/private network
           + logs/metrics/support/tax

Platform monthly = DB monthly
                 + S3 versions/requests/download
                 + scan/OCR queues and compute
                 + cache/search/vector compute/storage
                 + app runtime/outbox workers
                 + engineering review/support labor
```

The accepted RDS pilot estimate in `DEC-019` is approximately USD 87.03/month for `db.t4g.small` Multi-AZ + 20 GiB gp3 before Proxy, backup excess, monitoring, traffic, Support and tax. Preserve it as a dated baseline, not as the Scale budget.

### 10.3 Capacity decisions by profile

| Profile | Likely database action | Evidence/stop trigger |
| --- | --- | --- |
| Pilot | Keep accepted small Multi-AZ RDS; load-test connection pool and 500 MB document lifecycle without placing bytes in DB. | CPU/memory/free storage, connections/pool wait, locks, WAL/backup, P95 and restore drill. Any sustained saturation blocks expansion. |
| Growth | Scale vertically first; add Proxy only if connection churn/peak evidence justifies it; measure pinning. Consider read replica only for proven read-only projection workload. | Per-tenant resource attribution, top queries, queue lag, backup growth, selective tenant restore within RTO. |
| Scale | Re-evaluate instance class/storage/index/partitioning; compare Aurora only with measured I/O/read/failover data; isolate a noisy tenant only after quota/tuning evidence. | Sustained tenant-share breach, restore RTO, connection envelope, failover recovery, cost per driver, not tenant count alone. |

No tenant threshold by itself mandates Aurora or database-per-tenant. Fifty idle/light tenants can be cheaper than one abusive tenant; placement follows measurements.

## 11. Noisy tenant, quota and performance controls

- Define per-organization budgets for interactive requests, exports, uploads/bytes, scan/OCR work, search/vector queries, concurrent long jobs and daily/monthly cost units.
- Enforce global safety ceilings and tenant admission atomically before enqueue; return stable retryability/limit codes.
- Use weighted/fair queues and per-tenant concurrency, with a reserved operations lane for revoke/purge/reconciliation.
- Attach organization to query/job metrics without PII. Monitor CPU, I/O, WAL, temp bytes, locks, dead tuples, connection/pool waits, statement timeouts and queue age.
- Use statement, lock, transaction and idle-in-transaction timeouts. Bound export/search result size and cursor lifetime.
- Index tenant predicates and test skew. Partition only after measured table/index/vacuum benefits; tenant-per-partition at 50 tenants can create operational fan-out without solving CPU contention.
- Quota suspension must not block revocation, security audit, tenant export/exit or retention duties; those semantics require `DEC-060` approval.

## 12. Migration, rollback and vendor exit

### 12.1 Shared-schema second-tenant migration

1. **Expand:** add/repair tenant keys, global-school tables, tenant extension relations, composite uniques/FKs and policy catalogue without enabling tenant 2.
2. **Backfill/reclassify:** first-tenant rows receive the approved organization; global school snapshots move to explicit global identities/revisions; tenant overlays retain organization locality. Record counts/hashes/rejects.
3. **Enforce:** `NOT NULL`, composite FKs, `ENABLE` + `FORCE RLS`, unprivileged grants, catalogue fingerprint and missing-context denial.
4. **Canary:** synthetic two-tenant data only; direct SQL/app/API/search/S3/job/export/support negative tests.
5. **Switch:** remove the one-active-organization constraint only after membership/session/subscription/support semantics are approved and code is compatible.
6. **Contract:** remove obsolete tenant-duplicated global shapes only after all supported builds stop reading them and rollback window closes.

Rollback means compatible application rollback or forward corrective migration. Never edit an applied SQL file; never rely on a destructive `down` migration; never roll back by disabling RLS.

### 12.2 Vendor move

Use logical dump/restore or replication only after the target passes the Hong Kong/service gates. Freeze or delta-capture with monotonic cursor and final zero-delta reconciliation; verify extensions, roles, owners, policies, sequences, collations/time zones, checksums/counts and query plans. Endpoint switch is separate and human-approved. Maintain the old read path only for a bounded, documented rollback window; compensate external effects rather than dual-writing indefinitely.

## 13. Required deterministic evidence before tenant 2

1. Close every `DEC-060` item that changes subscription state, support access, retention, export/exit, outage behavior or tenant-to-global knowledge.
2. Approve organization identity/duplicate/owner transfer and membership invite/activate/disable/switch/session-invalidation contracts.
3. Migrate global school base/revisions out of the current tenant-duplicated shape; approve tenant extension/correction/retirement/orphan behavior.
4. Catalogue every table and relation; prove tenant-owned rows are `NOT NULL`, tenant FKs are composite, and global/control tables have explicit visibility.
5. Prove every tenant table has `ENABLE` + `FORCE RLS`, canonical policy, non-app owner, and app role `NOBYPASSRLS`; fail migration drift checks otherwise.
6. Run real PostgreSQL pool tests for no context, commit/rollback reuse, cross-tenant ID guessing and concurrent membership/grant revoke.
7. Pass cross-tenant API/UI/count/timing, S3 intent/object, cache, outbox/job, export, support and search/vector negative tests.
8. Demonstrate privileged migration/restore/job identities, exact runbooks, time-bounded credentials, audit and no ordinary-request path.
9. Complete failover/PITR and selective one-tenant restore drills in Hong Kong with counts/hashes, S3 linkage, audit continuity, policy catalogue and unexplained difference zero.
10. Run replaceable Pilot/Growth/Scale workloads, including skew/noisy tenant, 500 MB file lifecycle, connection pinning, queue/DLQ, vector recall/latency and cost-driver receipts.
11. Produce dated official calculator receipts and approved budgets/stops for the selected exact RDS topology; alternatives require a superseding ADR.
12. Provide a minimal Platform Control Plane or equivalent controlled runbook for onboarding, placement, entitlement/quota, support grant, release, incident, backup/restore and termination.

Any cross-tenant read/write or unexplained restore difference is a release blocker, not a warning.

## 14. Decision recommendations

### `DEC-053`: retain and amend for enforceability

Retain shared HK PostgreSQL/shared schema, mandatory tenant key/composite locality, server authz first and forced RLS second. Amend or add an implementation appendix to state:

- every tenant table is catalogue-classified and enforced with `ENABLE` **and** `FORCE ROW LEVEL SECURITY`;
- application role is non-owner and `NOBYPASSRLS`; migration/restore roles never serve application traffic;
- policy/context/owner/grant fingerprints are migration drift evidence;
- transaction-local context is proven across pool commit/rollback reuse;
- actor FKs are explicitly global provenance or tenant membership-bound authority;
- global school facts use separate global relations, not null/sentinel tenant and not per-tenant base duplication;
- a standard shared placement plus future exceptional dedicated placement is allowed only through measured/human-approved gates;
- single-tenant restore is a required tested recovery workflow.

### `DEC-060`: keep open, but add prerequisites rather than engineering defaults

Do not close its business/legal semantics in code. Add explicit decisions for:

- organization stable identity, duplicate/legal merge, last-owner and ownership transfer;
- multi-organization membership switch and session invalidation;
- quota dimensions, overage/admission behavior and security/exit operations exempt from suspension;
- support grant actor/approver/expiry/notification/post-review;
- tenant selective restore authority, acceptable current-state overwrite/merge rules and customer approval;
- dedicated placement eligibility, cost owner, migration/exit and shared-to-dedicated rollback;
- global school versus tenant extension field taxonomy and lifecycle;
- vector/query/log PII policy, namespace deletion/rebuild and tenant-to-global promotion/withdrawal.

### New decision recommended

Create a new decision after human review: **Tenant Placement and Selective Recovery**.

It should establish shared HK RDS as default, define measured triggers for dedicated HK placement, forbid cross-placement joins, require a provider/placement allowlist, and own the single-tenant restore/export/termination evidence contract. This concern is operationally distinct from `DEC-053`'s row isolation and `DEC-060`'s still-open commercial/legal semantics.

## 15. Risks and stopping conditions

| Risk | Consequence | Control / stop |
| --- | --- | --- |
| Owner bypasses non-forced RLS | Cross-tenant exposure despite policy | `FORCE RLS`, non-owner app role, catalogue test; any failure blocks release. |
| Later table lacks policy | New feature silently bypasses isolation | Unclassified-table migration gate and final schema fingerprint. |
| Pool context leaks | Request B inherits tenant A | Transaction-local context and real connection-reuse tests. |
| Global facts are tenant-duplicated | Divergent truth, N copies, unsafe promotion | Separate global revision and tenant extension schema before tenant 2. |
| Privileged job uses wildcard | Broad blast radius | One-tenant leases/transactions, narrow function, audit and row limits. |
| Shared PITR mistaken for tenant restore | Other tenants overwritten or disclosed | Isolated full restore + selective owning-module reconciliation drill. |
| RDS Proxy pinning | Connection saturation/noisy failure | Load test/metrics; omit Proxy or resize/tune if multiplexing benefit fails. |
| Vector filter leaks or degrades recall | Unauthorized or wrong retrieval | Namespace + policy recheck + skew/recall/latency tests; index remains derived. |
| Azure/Neon selected by generic feature list | Residency/HA contradiction | Hard service-specific Hong Kong evidence; new ADR required. |
| Point price goes stale | Underbudgeted platform | Dated calculator receipts and formula/usage inputs; no invented exact totals. |

Terminal state for second-tenant preparation is `passed` only when all Section 13 evidence is approved. Otherwise use `needs_human` for unresolved `DEC-060` semantics, `blocked` for residency/isolation/recovery failures, or `budget_exhausted` for a workload that exceeds approved limits. Do not weaken isolation, restore or negative-test gates to obtain a pass.

## 16. Primary sources

Official sources checked on 2026-08-11:

- PostgreSQL: [Row security policies](https://www.postgresql.org/docs/current/ddl-rowsecurity.html), [CREATE ROLE / BYPASSRLS](https://www.postgresql.org/docs/current/sql-createrole.html), [configuration functions](https://www.postgresql.org/docs/current/functions-admin.html).
- AWS: [RDS Multi-AZ](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.html), [RDS PostgreSQL pricing](https://aws.amazon.com/rds/postgresql/pricing/), [RDS PITR](https://docs.aws.amazon.com/AmazonRDS/latest/APIReference/API_RestoreDBInstanceToPointInTime.html), [snapshot restore](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_RestoreFromSnapshot.html), [RDS Proxy transactions](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.howitworks.html), [RDS Proxy pinning](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/rds-proxy-pinning.html), [Aurora pricing](https://aws.amazon.com/rds/aurora/pricing/), [Aurora failover](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-failover.html), [Aurora backup/restore](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Managing.Backups.html).
- Microsoft: [Azure Database for PostgreSQL regional availability](https://learn.microsoft.com/en-us/azure/postgresql/overview), [high availability](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-high-availability), [backup and restore](https://learn.microsoft.com/en-us/azure/postgresql/backup-restore/concepts-backup-restore), [official pricing](https://azure.microsoft.com/en-us/pricing/details/postgresql/flexible-server/).
- Neon: [status/region list](https://neon.com/docs/introduction/status), [project/region model](https://neon.com/docs/manage/projects), [pricing](https://neon.com/pricing).

Prices, SKU availability and temporary regional restrictions change. Recheck the exact official region/SKU/calculator immediately before any architecture approval or procurement.
