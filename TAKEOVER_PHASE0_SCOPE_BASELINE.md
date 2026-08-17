# Takeover Phase 0 Scope Baseline

> Simplified Chinese reading copy: `TAKEOVER_PHASE0_SCOPE_BASELINE.zh-CN.md`.
> Both files must preserve the same meaning; stop and correct any discrepancy.

| Field | Value |
| --- | --- |
| Status | `draft_for_user_review` |
| Baseline date | 2026-08-17 |
| Code baseline | `3dbea7534170a98b5bae34c1750dbb65f568ed3e` on `origin/main` |
| Product-document baseline | local-only `txgj-doc` commit `83df7812149aed8a99be83d9ff2800c9d7140bfc` |
| Authorized delivery target | Operable local Release 1 using deterministic synthetic data |
| External effects | None by default; this document grants no standing cloud, production, real-data, deployment, commit, or push authority |

## 1. Purpose

This baseline fixes the execution boundary for the current takeover. It prevents
future-production plans, historical completion claims, and existing source
artifacts from being interpreted as authority or proof for the local delivery.

Takeover Phase 0 is an audit and execution-alignment phase. It is not the same
work as the historical PRD Phase 0 (`P0-01` through `P0-12`). Existing P0
contracts, migrations, tests, and records are inputs to be verified; they are
not automatically repeated or accepted as runtime-complete.

## 2. Authority And Evidence Order

When sources disagree, use this order:

1. The user's latest explicit instruction.
2. Accepted or amended decisions in `txgj-doc/PRD_IMPLEMENTATION_DECISIONS.md`.
3. Adopted phase amendments and technical decisions in `txgj-doc`.
4. Ticket dependencies and acceptance contracts in
   `txgj-doc/PRD_PHASE_IMPLEMENTATION_PLAN.md`.
5. Version-bound implementation records, contracts, migrations, tests, and
   signed evidence in this repository.
6. Research and archived design documents.

Product documents govern approved business meaning. Current source and actual
test/runtime evidence govern implementation status. A source artifact or an
`implemented_local` label is never by itself runtime, browser, database, cloud,
or release evidence.

## 3. Current Local Release 1 Scope

### In Scope

- A local Next.js ERP and `/api/v1/**` business API using deterministic
  synthetic identities and business records.
- Local server-side sessions and development-only role login that are disabled
  outside the explicit local runtime mode.
- Student, Guardian, ServiceCase, the 16-field assessment, SchoolTarget, Task,
  Document, in-app Notification, and Audit workflows.
- Validated school snapshot use, filtering, review decisions, tickets, and
  controlled reconstruction using committed non-PII source data.
- A bounded read-only External Portal with strict field allowlists and no
  document listing or download.
- Platform advancing-case counts and opaque contract reference values, with no
  payable amount or accounting workflow.
- PostgreSQL 17, LocalStack S3/SQS behavior, and ClamAV under an explicit local
  Colima/Docker Compose profile.
- Focused tests per approved ticket and separately approved milestone-wide
  checks.

### Out Of Scope

- AWS, Cloudflare, Vercel, DNS, Terraform execution, production deployment, or
  production account and budget work.
- Real Cognito, RDS, S3, SQS, KMS, production secrets, or real notification
  delivery.
- Real Student, Guardian, ServiceCase, document, or other private business data.
- Neon-to-RDS migration, production backfill, first-case release, pilot, second
  organization activation, or production support access.
- Automated crawler scheduling or crawler-to-frontend deployment commits.
- AI/knowledge workflows, external Email/SMS/WhatsApp, non-K12 workflow, public
  portal registration or writes, and multi-tenant activation.
- Charge formulas, proration, invoices, charge notices, payments, refunds, tax,
  or accounting.

Future AWS Hong Kong documents remain the production target and security
constraint. They are deferred, not replaced by the local runtime target.

## 4. Non-Negotiable Execution Constraints

- `local-synthetic` and `production-aws` composition must be explicit and
  mutually exclusive. Local adapters must fail closed outside local mode.
- Production region, privacy, authorization, audit, tenant isolation, document
  quarantine, idempotency, and append-only migration constraints must not be
  weakened for local convenience.
- `/api/v1/**` is the target business API. Existing non-v1 and Neon paths are
  migration debt, not the target design.
- Mock and preview implementations remain until a verified replacement exists;
  their presence must be visible in the traceability matrix.
- Future-scope source may remain in the repository but must stay hidden or
  disabled in Release 1.
- No commit, push, broad test, installation, service start, migration execution,
  external operation, or deployment is implied by this baseline.

## 5. Repository Facts At Baseline

These counts are discovery facts, not completion claims:

| Fact | Observed state |
| --- | --- |
| Framework | Next.js `16.2.7`, React `19.2.4`, TypeScript `^5` |
| Package runtime contract | pnpm `10.34.4`; Node `>=20.11.0`; local target remains Node 22 |
| Ordered SQL migrations | 15 files |
| API Route Handlers | 50 total; 37 under `/api/v1` |
| Unavailable default runtimes | 21 runtime files contain `RuntimeUnavailable` |
| Page-level mock/preview consumers | 5 current consumers |
| Architecture discipline | 11 local tests enforce real-source module imports, table-write ownership, and server-only markers |
| Crawler handoff | Four committed JSON files exist; manifest is the older 2026-06-10 warning format, not `crawler-handoff/v1` |
| Release gate | P3-19 remains unsigned `no-go` |
| Root README | Still the create-next-app template; rewrite is deferred until the local runtime is real |

The later runtime audit owns interpretation of these facts and may refine the
counts without changing this scope boundary.

## 6. Known Conflicts And Takeover Disposition

| Conflict | Current disposition |
| --- | --- |
| Historical PRD Phase 0 versus takeover Phase 0 | Preserve historical tickets; use takeover Phase 0 only to verify and re-sequence current work. |
| AWS Hong Kong production plan versus local-first delivery | Keep AWS as the future production target; authorize only local synthetic work now. |
| Implementation records versus runnable behavior | Require current code, focused test, database, browser, and runtime evidence appropriate to each claim. |
| `/api/v1/**` contracts versus legacy `/api/cases` and Neon services | Treat v1 as target and legacy paths as explicit migration debt with exit criteria. |
| Production-only configuration guards versus local PostgreSQL/LocalStack | Add a separate fail-closed local composition later; do not relax production guards. |
| Current crawler snapshot versus `crawler-handoff/v1` validator | Keep current snapshot read-only; define conversion/republication evidence before activating the new validator path. |
| Future AI/knowledge and multi-tenant source versus Release 1 boundary | Keep source dormant and navigation/runtime-disabled; do not infer feature authority from file presence. |
| Historical workspace paths such as `erp-frontend/` | Resolve them relative to the current `Tianxingguoji` repository; do not recreate the old workspace layout. |

## 7. Phase 0.1 Exit Check

- Current authority order and evidence meaning are explicit.
- Local Release 1 in-scope and out-of-scope behavior is explicit.
- Future production architecture remains protected without becoming current
  execution authority.
- Known scope and implementation conflicts have a fail-closed disposition.
- No business code, runtime, toolchain, external service, or release state was
  changed.

The next proposed step is Phase 0.2: build the requirement-to-page/API/module/
database/test traceability matrix. It requires separate user confirmation after
review of this baseline.
