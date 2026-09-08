# CRM Cloud Design Revision Review

> Review date: 2026-08-11 (Asia/Hong_Kong)  
> Status: advisory; no decision, deployment, migration, cloud change, or release authority  
> Reviewed authority: `PHASE3_EMPTY_TENANT_PILOT_REVISION_PLAN.md`, `PRD_IMPLEMENTATION_DECISIONS.md`, `PRD_PHASE_IMPLEMENTATION_PLAN.md`, and `TECHNICAL_DECISION_PRODUCTION_PLATFORM.md`  
> Compared input: `/Users/mingjiexing/Downloads/留學 CRM 部署與雲端考量.md`  
> Evidence rule: the downloaded Gemini conversation is an untrusted design input, not a primary source or implementation contract. Time-sensitive product claims were checked against first-party documentation and the existing dated repository research.

## 1. Executive conclusion

The current design should **not** switch Release 1 from AWS Hong Kong to Azure, replace Cognito with Entra External ID, place sensitive case documents in Cloudflare R2, move transactional CRM ownership from the Next.js modular monolith to FastAPI, or enable Azure OpenAI for incompletely de-identified case data.

The downloaded conversation contains several sound requirements and patterns, but most of them are already represented more rigorously in the current decisions:

- authenticated and sensitive processing must be separated from any public/static plane;
- document bytes use private object storage and short-lived direct-upload/download intents;
- multi-tenancy uses server authorization plus shared-schema PostgreSQL RLS as defence in depth;
- global school facts remain separate from tenant-owned strategy and case knowledge;
- crawler evidence requires human review before governed publication;
- vector indexes and other AI/search artifacts are derived, rebuildable data;
- crawler, scanner, OCR, index, and AI work should not share the request-path resource envelope.

The real corrections are narrower:

1. Resolve the governance contradiction between accepted AWS decisions and the TDR's still-pending status.
2. Add an explicit 500 MB document lifecycle and cost/capacity workload to Phase 3 evidence.
3. Bind the production build and first-case gate to one exact, reviewed crawler snapshot receipt.
4. Complete global-fact, tenant-extension, knowledge-publication, and retrieval contracts before a second tenant.
5. Keep pricing, quota, subscription, AI provider, tenant-to-global promotion, and mainland deployment semantics open rather than inventing them from the downloaded conversation.

## 2. Problem framing and scope

### In scope

- Determine whether the new input invalidates existing production-platform, multi-tenant, knowledge, crawler, or pilot decisions.
- Identify conflicting statements, missing invariants, enforcement owners, evidence gates, and the document that should own each correction.
- Recommend a minimum, dependency-ordered revision set.

### Out of scope

- Selecting or purchasing a new cloud service.
- Changing accepted decisions by implication.
- Legal conclusions about PDPO/PIPL or mainland data localization.
- Production IaC, migration, data movement, crawler publication, snapshot copy, deployment, or release action.
- Turning the downloaded pricing suggestions into product pricing or financial forecasts.

### Stakeholders

- Founder/Product: product boundary, customer promise, pilot decision, and commercial semantics.
- Security/Privacy/Data: residency, identity, tenant isolation, knowledge ownership, and data lifecycle.
- Operations/Infrastructure/Budget: runtime topology, capacity, recoverability, and approved cost envelope.
- Customer Organization owners and advisors: case confidentiality, private know-how, delegated access, and usable workflows.
- Platform Data Reviewers: global school-fact provenance and publication quality.

## 3. Authority and status correction

The decision ledger already marks the following as accepted:

- sensitive-data Hong Kong boundary: `DEC-018`;
- RDS PostgreSQL Multi-AZ in `ap-east-1`: `DEC-019`;
- Cognito in `ap-east-1`: `DEC-020`;
- sensitive application runtime in AWS Hong Kong: `DEC-021`;
- private S3 document domain in `ap-east-1`: `DEC-024`.

However, `TECHNICAL_DECISION_PRODUCTION_PLATFORM.md` still labels the complete TDR as `recommended` and leaves AWS/Cognito/S3 rows pending. This is the highest-priority correction because it makes an accepted baseline appear to be an open vendor comparison.

Recommended status model:

| Item | Correct status |
| --- | --- |
| Hong Kong sensitive boundary, AWS RDS, Cognito, S3 | Accepted by existing DEC entries |
| ECS/ALB production topology and exact IaC payload | Recommended until separately approved/applied |
| Optional public-plane provider | Open, Release 1 may remain AWS-only |
| Future AI worker and provider | Open and disabled |
| Exact monthly budget/capacity | Open until dated calculator, workload, and alert thresholds are approved |

The TDR should cite the accepted entries rather than ask a second approver to re-approve them informally. Any move to Azure/Entra/R2 must be a new superseding decision with migration, rollback, data-flow, service-specific residency, and cost evidence.

## 4. Cross-source decision matrix

| Downloaded proposal | Current verdict | Reason / invariant owner | Revision destination |
| --- | --- | --- | --- |
| Containerize sensitive Next.js/API in Hong Kong | Retain | Matches `DEC-018/021`; Application + Infrastructure own evidence | TDR implementation detail |
| Direct object-store upload/download | Retain and deepen | Matches S3 intent model; Document module owns authz, quarantine, checksum, scan state | TDR + Phase 3 capacity evidence |
| Shared PostgreSQL plus RLS | Retain and deepen | `TenantAccess` server policy is first control; `FORCE RLS` is second; DB role cannot bypass | Ledger `DEC-053` contract |
| Entra maps organization, advisor, and case authorization | Reject | IdP proves identity only; RDS membership/scope/capability/expiry/session version is business truth | TDR identity section |
| Cloudflare R2 stores sensitive case files | Reject | No Hong Kong jurisdiction guarantee; egress price does not prove residency/log/support boundary | TDR rejected alternatives |
| Azure becomes Release 1 core cloud | Reject for current release | Conflicts with accepted AWS decisions and existing adapters/IaC; requires a superseding ADR | No Phase-plan change |
| FastAPI becomes transactional CRM core | Reject | Splits an already deep modular monolith, transaction ownership, authorization, audit, and error contracts | Python remains crawler/future isolated worker |
| Crawler writes directly to production knowledge/vector tables | Reject | Bypasses immutable four-file release and human warning gate | Ledger `DEC-050/052/059` |
| Public base plus tenant-private extension | Retain and specify | Global facts and tenant strategy have different owners, versions, visibility, and approval paths | Ledger `DEC-053/059` |
| NULL or zero tenant denotes public data | Reject | Tenant-owned rows require non-null organization locality; global data needs explicit global ownership/relation | Ledger schema contract |
| De-identification automatically makes a case reusable RAG knowledge | Reject | Full case, candidate lesson, approved article, snapshot, and derived index are distinct states | Ledger `DEC-058/059` |
| Azure OpenAI exemption permits incompletely de-identified prompts | Reject | Logging/training policy is not Hong Kong inference residency or permission to disclose PII | Ledger `DEC-047/048`; TDR AI section |
| Residential proxy pool is mandatory crawler infrastructure | Reject pending separate review | Terms, legal, supplier, credential, disclosure, and provenance risks are unresolved | Separate crawler TDR only if justified |
| USD 130-150 is a 24/7 HA production budget | Reject | Workload and cost scope are incomplete; current operating model is not 24/7 on-call | TDR cost section |
| Per-case marginal cost is below USD 1 | Treat as hypothesis only | Omits scanning/OCR, embeddings, vector compute, requests/egress, logs, backup, retries, support, and review labor | Product finance research, not PRD authority |

## 5. Required design amendments

### 5.1 `PRD_IMPLEMENTATION_DECISIONS.md`

Do not rewrite `DEC-018` through `DEC-024` based on the downloaded conversation. Amend the following contracts instead.

#### Identity and membership

Add or close explicit semantics for:

- `CustomerOrganization` legal/stable identity and duplicate rules;
- membership invite, accept, revoke, active-organization switch, and session invalidation;
- owner/founder naming, last-owner protection, and ownership transfer;
- one provider identity belonging to multiple organizations;
- support access as a separate, time-bounded grant rather than tenant membership.

Enforcement owner: `Identity` for provider subject/session; `TenantAccess` for organization, membership, role, scope, entitlement, and support grant.

#### Global facts and tenant extensions

Specify:

- global rows have an explicit platform owner or separate global relation, not a null/sentinel tenant;
- tenant extensions have non-null `organization_id`, a global-record foreign key, version, visibility, and unique locality;
- global revision updates trigger compatibility/impact review without overwriting tenant strategy;
- delete, retire, merge, supersede, and orphan behavior;
- tenant-submitted fact corrections become global only through a separate platform review.

Enforcement owner: `SchoolIntelligence` for global facts; `Knowledge` for approved knowledge versions and indexes; `TenantAccess` for visibility.

#### Crawler and knowledge state model

Map the existing A/B/C data classes to explicit states:

```text
raw crawler evidence
  -> review candidate
  -> approved global school revision
  -> immutable published snapshot
  -> rebuildable search/vector index

tenant case
  -> lesson candidate
  -> independently reviewed tenant KnowledgeArticleVersion
  -> approved tenant snapshot
  -> rebuildable tenant index
```

Required invariants:

- raw/review candidate data is not visible to tenants or AI retrieval;
- author/submitter cannot be sole approver;
- tenant content never becomes global by default;
- a published version pins sources, reviewer, schema/policy, and snapshot identity;
- source withdrawal, expiry, correction, or purge propagates to cache/index through reconciliation;
- indexes are never authoritative business truth.

#### Retrieval interface

Define one `Knowledge` module interface that accepts trusted actor, active organization, case context, purpose, approved snapshot, and budget. It must intersect:

- actor membership and role;
- case grant/capability/expiry;
- global visibility;
- tenant-knowledge visibility;
- source-case restrictions;
- approved version/snapshot state.

It returns authorized chunks with citations and version metadata. Query text, cache keys, logs, and telemetry require an explicit PII policy. Model, chunk schema, embedding dimension, index version, staleness, rebuild, and reconciliation are behavior-affecting versioned inputs.

#### Keep `DEC-060` open

The following remain human decisions before a second tenant:

- subscription states, seat/storage/AI entitlements, quota, overage, suspension, and termination;
- export and tenant exit;
- tenant-to-global promotion, IP rights, withdrawal, and benefit terms;
- exact global-fact versus tenant-strategy field taxonomy;
- AI provider/DPA/inference location/retention and de-identification risk acceptance;
- region-outage policy and support-access semantics.

### 5.2 `TECHNICAL_DECISION_PRODUCTION_PLATFORM.md`

Revise the document in this order:

1. Correct the mixed accepted/pending status described in Section 3 of this review.
2. Add a crosswalk showing that the downloaded Azure/R2/Entra/FastAPI proposal is historical input, not approved architecture.
3. Preserve AWS `ap-east-1`, Cognito, RDS, S3, and the complete authenticated Next.js BFF for Release 1.
4. Specify direct upload as short-lived, single-action, single-object/version intent with multipart/checksum/abort/quarantine/scan/reconciliation behavior.
5. Keep Python as the existing crawler implementation and a possible future isolated AI/data worker; do not move CRM transaction ownership out of the owning TypeScript modules.
6. Require service-specific evidence for runtime, identity, logs, backup, support, queues, model inference, and every secondary copy. A region label is insufficient.
7. Replace point estimates with a versioned cost model containing workload, retention, request, egress, scan/OCR, index, logging, backup, support, and tax assumptions.

Official evidence supports the present caution:

- Cloudflare R2 location hints are best effort, while guaranteed jurisdictions currently list EU and FedRAMP, not Hong Kong: [R2 data location](https://developers.cloudflare.com/r2/reference/data-location/).
- Vercel states that data may be transferred to the United States and other processing locations, so choosing a function region alone is not end-to-end residency evidence: [Vercel security and compliance](https://vercel.com/docs/security/compliance).
- Entra External ID's Go-Local residency add-on is currently listed only for Australia and Japan, not Hong Kong: [External ID pricing and Go-Local](https://learn.microsoft.com/en-us/entra/external-id/external-identities-pricing).
- Existing first-party research already records the service-by-service hosting and identity comparison: [Production Hosting, Identity, and Future Agent Runtime Options](2026-08-11_PRODUCTION_HOSTING_AUTH_OPTIONS.md).

### 5.3 `PRD_PHASE_IMPLEMENTATION_PLAN.md`

Keep the current macro sequence:

```text
synthetic HK staging
  -> empty production tenant
  -> 1-3 reconstructed cases
  -> five-business-day observation
  -> optional expansion to 5-10 cases
  -> expand / hold / rollback
```

Make only the following ticket-level amendments:

| Ticket/gate | Amendment |
| --- | --- |
| `P3-02` | Add a versioned workload profile: synthetic 500 MB document lifecycle, multipart/direct upload, interrupted retry, checksum mismatch, quarantine, scan/DLQ, orphan cleanup, DB connections, CPU/memory, and API/page P95. |
| `P3-07` | Require accepted production architecture authority before production IaC source; bind exact monthly cap, alert thresholds, and stop owner to dated evidence. |
| `P3-12` | Bind deployed build to one exact four-file crawler snapshot manifest/hash, provenance/freshness result, and any human warning receipt. Verify no automated sync credential/job exists. |
| `P3-19` | Include architecture authority, workload/cost evidence, and crawler snapshot receipt in the signed checksum bundle. |
| `P4-04` | Add daily storage growth, object requests/egress, scan queue/runtime, DB connection saturation, and per-case cost receipts. |
| `P4-07` | Expansion stops when approved cost/capacity thresholds are exceeded, not merely when an alert exists. |
| `P4-10` | State explicitly that `expand` closes only the first-tenant, at-most-ten-case pilot; it does not authorize commercial multi-tenancy, billing, AI, or crawler automation. |

Crawler release evidence must reject partial file sets, unknown/stale provenance, unapproved warnings, schema/count/hash mismatch, or a candidate that has not passed the designated reviewer gate.

### 5.4 `PHASE3_EMPTY_TENANT_PILOT_REVISION_PLAN.md`

The rollout, reconstruction, telemetry, and stop rules remain sound. Only add a compact cross-reference to the new Phase 3 workload/cost/snapshot gates if the implementation plan is amended. Do not duplicate platform selection, multi-tenant knowledge semantics, or subscription rules here.

## 6. Second-tenant release gate

No second customer organization should be enabled until all of the following have deterministic evidence:

1. `DEC-060` subscription, support, retention, outage, promotion, termination, and export semantics are closed.
2. Organization identity, membership lifecycle, last-owner/transfer, active-organization switching, and session invalidation are specified.
3. Versioned migrations enforce non-null tenant locality, composite foreign keys/uniques, `FORCE RLS`, an unprivileged non-owner application role, and transaction-local tenant context reset.
4. Cross-tenant negative tests cover UI, direct requests, ID guessing, search/vector retrieval, export, S3 intents, jobs, cache, counts/timing, and support access.
5. Global-fact and tenant-extension ownership, version, correction, retirement, promotion, withdrawal, and index rebuild behavior are approved.
6. Knowledge publication, source-case authorization, snapshot pinning, citations, purge, cache/index reconciliation, and query/log PII policy pass.
7. A minimal Platform Control Plane or equivalent controlled runbook exists for onboarding, entitlements, support grants, feature flags, quotas, incidents, backup/restore, and termination.
8. A workload representing expected organizations, users, cases, 500 MB document lifecycles, vector/search distribution, queues, and noisy-tenant behavior meets approved performance, restore, and cost thresholds.
9. Legal/privacy owners confirm customer data-controller/processor roles and every AI/provider/data-flow boundary.

## 7. Risks the downloaded conversation understates

- A cloud region does not prove end-to-end residency across identity, logs, support, backups, model inference, DNS/CDN, or subprocessors.
- A simple RLS `WHERE tenant_id = ...` example does not cover trusted tenant-context injection, privileged roles, composite relationship locality, jobs, cache, search, export, or object storage.
- Automatic name/phone removal is not proof that a historical case is anonymous, legally reusable, or safe to embed.
- One administrator approving crawler data is not sufficient when the same actor authored or corrected it.
- A mixed-cloud object store adds an identity, logging, support, incident, and recovery boundary even when egress is cheaper.
- HNSW and metadata filtering require workload-specific query-plan, latency, recall, and noisy-tenant evidence.
- A residential proxy can create contractual, legal, credential, disclosure, and provenance risks; it is not a default reliability feature.
- Base cloud cost is not total cost of service and cannot establish pricing, margin, SLA, or 24/7 support viability.

## 8. Recommended revision sequence

1. **Documentation authority:** correct TDR status versus accepted DEC entries.
2. **Decision contracts:** close identity/membership and global/tenant knowledge gaps; leave commercial/AI semantics explicitly open.
3. **Platform evidence:** approve exact AWS topology and cost/capacity model; do not provision through this documentation change.
4. **Phase plan:** amend the seven ticket/gate points in Section 5.3.
5. **Deterministic implementation evidence:** add the 500 MB lifecycle, snapshot receipt, cross-tenant retrieval, and cost/capacity tests.
6. **Human release decisions:** separately approve IaC, migration, deployment, first-case entry, pilot expansion, and any second tenant.

## 9. Review limitations

- The downloaded document is a Gemini conversation without primary-source citations for most claims; its prices, service availability, compliance statements, and competitive claims cannot be treated as current facts.
- No legal opinion was obtained. Mainland and Hong Kong data rules require qualified legal/privacy review tied to actual data flows and contracts.
- No cloud account, secret, production resource, database, or deployment was accessed.
- The root instructions reference `RTK.md`, but no such file was found in the workspace. This review followed the complete repository instructions supplied in the task and records the missing include as a harness gap.

## 10. Terminal state

The design review is `passed` as an advisory cross-read. The current AWS Hong Kong direction and Phase 3/4 macro sequence remain recommended. Document authority synchronization and the identified contract/evidence amendments remain pending human review; no production or release state changed.
