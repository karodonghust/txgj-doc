# Release 1 AWS 香港遷移、家長／申請者門戶與平台計費實施方案

| 屬性 | 內容 |
| --- | --- |
| 文件類型 | Implementation Plan |
| 版本 | `v0.2` |
| 日期 | 2026-08-11（Asia/Hong_Kong） |
| Run ID | `R1-AWS-PORTAL-BILLING-20260811-v1` |
| 狀態 | `scope_approved`；`R1X-00` authority synchronization 已完成，`DEC-064`–`DEC-066` 已批准；`DP-01`–`DP-12` 仍按 owning ticket 阻擋未決業務語義 |
| 適用 repository | `erp-frontend/`；公開學校 release 邊界另涉及 crawler repository |
| 上游權威 | `PRD_IMPLEMENTATION_DECISIONS.md`、`TECHNICAL_DECISION_PRODUCTION_PLATFORM.md`、`TECHNICAL_DESIGN_AWS_CLOUDFLARE_PRODUCTION_DEPLOYMENT.md`、`PRD_PHASE_IMPLEMENTATION_PLAN.md` |
| 當前實施 checkpoint | 本地 evidence 已完成至 `P3-01`；`R1X-00` 已同步；下一張既有票據為 `P3-02`，`R1X-01/05` 等待各自 DP 批准 |
| 授權邊界 | 本文件只規劃本地 source、test、migration 與 IaC；不授權採購、provision、migration execution、production data write、DNS、deploy、邀請、通知發送、commit、push 或 Vercel 刪除 |

## 1. Outcome、stakeholders 與範圍

### 1.1 要解決的問題

Release 1 需要在目前最新決策下形成一條可執行的交付路徑：

1. 將完整 authenticated Next.js UI/BFF 從 Vercel 遷移至 AWS 香港 `ap-east-1`，並保持敏感資料、身份、文件、audit、log 和 backup 的香港邊界。
2. 完成現有「已有 interface／測試，但 production runtime 尚未接通」的 CRM 業務邏輯。
3. 新增家長／申請者只讀門戶；Advisor／Founder 可為指定案件生成訪問密鑰，並自行選擇到期時間。
4. 新增平台管理員界面，在不取得租戶 PII 的前提下觀察租戶數、每租戶推進中的案件數及合同單額，生成每月收費通知草稿。

### 1.2 Stakeholder outcome

| Stakeholder | 需要的結果 |
| --- | --- |
| Founder | 能掌握案件推進、控制外部查看權、覆核月度收費通知與 release gate |
| Advisor | 能執行 CRM 工作流，並為其有權管理的案件生成、撤銷及查閱門戶 grant |
| Guardian／Applicant | 無需成為內部員工帳號，只能查看明確批准的單一案件資料 |
| Platform Administrator／Finance | 只看計費所需的跨租戶聚合與合同資料，不因此取得學生、家長、文件或內部 notes |
| Security／Privacy | 所有 sensitive request 在 AWS 香港 request-time 授權；secret、cookie、log、telemetry 不洩漏 PII 或 capability |
| Operations | 能以 immutable build、可回滾 deployment、告警、restore evidence 和月度計費 receipt 營運 |

### 1.3 In scope

- AWS 香港 ECS/ALB production runtime、RDS/Cognito/S3/SQS/audit adapter 與 Vercel 退場。
- 現有 Release 1 CRM module 的 production repositories、composition root、正式 UI read/write paths。
- `ExternalPortalAccess` 與 read-only guardian/applicant workspace。
- `PlatformBilling` 的跨租戶 aggregate projection、合同單額與 monthly charge notice draft。
- 版本化 migration、error contract、負向授權、concurrency、replay、partial failure、browser/a11y、rollback 與 restore evidence。

### 1.4 Explicitly out of scope

- 家長公開註冊、家長 Cognito workforce role、家長寫入 assessment、留言、上傳文件、付款或電子簽署。
- Email／SMS／WhatsApp 自動傳送門戶密鑰或收費通知；Release 1 只生成密鑰和通知草稿，由已批准的人工渠道傳遞。
- 正式稅務發票、支付收款、會計總帳、收入確認、退款與逾期催收。
- 第二個 CustomerOrganization 的 production activation，直至 `DEC-060` 及第 12 節多租戶決策完成。
- AI 對客報告、Excel/CSV batch import、自動 crawler snapshot sync、Cloudflare 上的 authenticated UI。

## 2. 決策對齊與修訂要求

### 2.1 繼續生效的最新決策

| 決策 | 本計畫採用方式 |
| --- | --- |
| `DEC-018`–`DEC-024` | 所有 tenant、personal、identity、document、authorization、audit、log、backup 留在 AWS 香港 |
| `DEC-032`、`DEC-044` | mutation 使用 versioned envelope、transaction、idempotency、append-only audit 和 `409` optimistic concurrency |
| `DEC-051`–`DEC-056` | 區分 CustomerOrganization、OrganizationUser、EndCustomer 與 PlatformOperator；採 modular monolith owning module interface |
| `DEC-053` | shared PostgreSQL schema 使用 mandatory `organization_id`、FK locality、RLS、`NOBYPASSRLS` application role |
| `DEC-055` | 頁面是 server-side authorization 的結果，不是角色真相 |
| `DEC-060` | 第二租戶前，subscription、support、retention、termination/export 等不可逆語義保持 open |
| `DEC-061`、`DEC-062` | 受控歷史案件重建與 privacy-safe telemetry 保持 Phase 3/4 gate |
| `DEC-063` | authenticated app 全部進 AWS 香港；Cloudflare 只承載批准的公開學校 release；Vercel cutover 後退場 |
| `DEC-064` | Release 1 正式加入 bounded single-case Portal 與 aggregate-only PlatformBilling；open DP 行為 fail closed |
| `DEC-065` | Portal redeem 使用 `portal_auth` function-only capability，lookup 後另做 tenant-scoped request-time authorization |
| `DEC-066` | Platform Control audit 使用獨立 append-only aggregate，不放寬 tenant audit actor/organization invariants |

### 2.2 必須寫回決策台帳的 Release 1 scope amendment

使用者已於 2026-08-11 批准新增範圍；`R1X-00` 以 `DEC-064`–`DEC-066` 完成以下 authority 修訂：

- `DEC-001`：Release 1 加入受限、只讀 guardian/applicant portal 與 aggregate-only PlatformBilling。
- `DEC-033`：仍不發外部通知；人工交付 portal key 和 charge notice 不等於平台通知功能。
- `DEC-054`：補充 External Customer Portal 與 Platform Billing Control 兩個 surface，但不改變三個產品平面的資料所有權。
- `DEC-060`：平台管理頁的單租戶開發不等於第二租戶 activation；跨租戶 production gate 仍有效。
- `PRD_PHASE_IMPLEMENTATION_PLAN.md`：在 `P3-01` 之後加入本文第 11 節票據，重新計算 dependency 和 Phase 3 exit criteria。

Authority sync 只批准 scope 和兩個 architecture invariants，不批准 `DP-01`–`DP-12` 或 production readiness。未決行為仍不得寫入 schema default、policy、UI 或 release evidence。

## 3. 現有代碼完成度與未完成業務邏輯

### 3.1 已存在的本地實施基線

- `db/migrations/001`–`010` 已建立 identity/access、CRM、case、school overlay、task、document、audit/outbox 與 application DB role schema。
- `modules/{identity,access,crm,cases,schools,tasks,documents,notifications,operations}` 已有 owning interface、policy/service 和 focused integration/failure tests。
- `/api/v1/**` 已有 identity、case、assessment、collaborator、school target、document、task、guardian、school governance 和 dashboard Route Handler skeleton。
- `infra/terraform/environments/staging/**` 已有香港 staging foundation；文件、network、RDS、web runtime module 已存在。
- `P0-01`–`P2-12` 有 implementation records；這些證明 local contract/test coverage，不是 production adapter 或部署證明。

### 3.2 仍未寫完或未接通的代碼

| 缺口 | 現況證據 | 必須完成的 implementation | Enforcement owner |
| --- | --- | --- | --- |
| Production composition root | 18 個 `get*Runtime()` 目前一律拋出 `*RuntimeUnavailable` | 建立 HK-only composition root，注入 RDS repositories、Cognito verifier、S3 signer、SQS/worker、audit/outbox；缺 config 繼續 typed `503` | Shared runtime + owning modules |
| Production PostgreSQL repositories | 現有多為 interface 和 test fake；`P3-08/09` 尚未完成 | Identity、Access、CRM、Cases、Schools、Tasks、Documents、Audit、Notifications repositories；同 transaction fresh authz/RLS/audit | 各 owning module |
| 第二租戶 schema gate | `001_expand_identity_access.sql` 的 `access_organizations_one_active_idx` 目前強制最多一個 active organization | 保留此 guard，直至 `DEC-060`、`DP-06`–`DP-11` 和跨租戶 evidence 全部批准；之後以獨立 migration 移除／取代 | Access + Data + Security |
| Historical reconstruction | `P3-03/04` 尚未開始 | contract/policy/service/repository、migration、Route Handlers、Advisor draft／Founder review UI、atomic activation | Cases |
| Privacy-safe telemetry | `P3-05/06` 尚未開始 | allowlist schema、producer rejection、HK sink adapter、degraded state、30-day retention、alert receipt | AuditOperations |
| 正式 CRM UI data path | `/students` 仍使用 `modules/crm/infrastructure/mock-students.ts`；部分 `/cases` 頁與文案仍使用 legacy Neon `/api/cases` | Student 360/list/create、case list/new/detail/workspace 全部切 `/api/v1` owning interfaces；移除 production mock/Neon fallback 和錯誤文案 | CRM + Cases + UI |
| 完整 Route Handler surface | 現有 Route Handlers 主要覆蓋 command；部分 list/detail/read model 尚未形成正式 interface | 補齊 organization-scoped queries、pagination/filter、stable envelopes、denied/empty/error state | 各 owning module |
| Auth runtime | Cognito adapter/interface 已有，但正式 runtime 未配置 | Cognito `ap-east-1` verifier、opaque session repository、TOTP/revoke/reconciliation、cookie/callback origin | Identity |
| Document effects | upload/scan/version/policy interface 已有，但 production S3/SQS/scanner 未接通 | quarantine-first S3 intent、scan worker、DLQ、idempotency、reconciliation、restore | Documents |
| Public school catalogue | 仍以 Git snapshot/local API 為 active path | `PublicSchoolCatalogue` local/R2 adapters、public compiler/publisher、read-back/hash/pointer activation | SchoolIntelligence |
| Production platform | 只有 staging Terraform foundation | production IaC、standalone Docker、ECR/GitHub OIDC、ALB/WAF、>=2 tasks、multi-instance cache/build-key contract | Infrastructure + Release |
| External portal | 無 entity、migration、module、route 或 page | 本文第 5 節完整 vertical slice | ExternalPortalAccess |
| Platform billing admin | 無 contract amount、billing snapshot、notice 或跨租戶 aggregate module | 本文第 6 節完整 vertical slice | PlatformBilling |

### 3.3 不可誤判為「已完成」的項目

- Route Handler 能把 `RuntimeUnavailable` 映射為 `503`，只證明 fail-closed error contract，並不表示功能可用。
- In-memory repositories 只可作 test adapter，不是 production persistence。
- staging Terraform 的單 task／health-only listener 不符合 production `>=2 tasks / >=2 AZs`。
- 目前 UI 中的 Neon 或 mock 文案與路徑是 migration debt；AWS cutover 後不得存在 active fallback。

## 4. Target architecture 與 module seam

```text
Browser
  +-- Internal user
  |     -> app.<domain> / login via Cognito ap-east-1
  |     -> opaque internal session cookie
  |
  +-- Guardian / Applicant
        -> app.<domain>/portal/access
        -> redeem high-entropy portal key
        -> separate opaque portal session cookie

app.<domain> (Cloudflare DNS-only)
  -> AWS WAF + ALB, ap-east-1
  -> ECS Fargate Next.js UI/BFF, private, >=2 tasks / >=2 AZs
       -> Identity module -> Cognito + RDS
       -> CRM / Cases / Documents / Access -> RDS + private S3/SQS
       -> ExternalPortalAccess -> RDS grant/session + read model
       -> PlatformBilling -> aggregate projection + charge notice draft
       -> AuditOperations -> mandatory audit + privacy-safe telemetry

data.<domain> (Cloudflare proxied)
  -> R2 approved public school releases only
```

新增兩個 deep modules：

1. `ExternalPortalAccess`：interface 隱藏 key hashing、expiry、redeem、portal session、request-time case scope、visible-field policy、revoke 與 audit。
2. `PlatformBilling`：interface 隱藏 tenant aggregation、contract version、month close snapshot、pricing policy、rounding、draft generation 與 correction revision。

Route Handlers 不可直接 join 多個 module tables。跨 module transaction 由 owning repository 或明確 application command orchestration 執行；跨租戶管理 read 只讀 PlatformBilling 的最小化 projection。

## 5. 家長／申請者門戶

### 5.1 Identity model

`PortalViewer` 不是 internal `User`、`OrganizationMembership` 或 Cognito workforce identity。最小資料模型：

```text
PortalViewer
  id, organization_id, subject_type(guardian|applicant)
  guardian_id?, student_id?, status, record_version

PortalAccessGrant
  id, organization_id, service_case_id, portal_viewer_id
  secret_hash, secret_fingerprint, capability_set_version
  status(active|revoked|expired), issued_by, issued_at, expires_at
  revoked_by?, revoked_at?, revoke_reason?, record_version

PortalSession
  id, organization_id, grant_id, session_secret_hash
  created_at, idle_expires_at, absolute_expires_at, revoked_at?
```

Identity rules：

- 一個 raw portal key 只屬一個 organization、一個 viewer 和一個 ServiceCase；不可跨兄弟姊妹或跨 case 隱式擴張。
- Guardian 必須經現有 `StudentGuardianRelationship` 關聯；adult applicant 的 `student_id` 與年齡／法定身份規則仍是決策點。
- raw key 使用 CSPRNG 產生，至少 256 bits entropy；只顯示一次，RDS 只存 keyed hash／fingerprint，不存 plaintext。
- key 不作 URL query。使用者在 `/portal/access` 貼上／輸入後兌換成獨立、HttpOnly、Secure、SameSite portal session cookie。
- internal session 與 portal session 使用不同 cookie name、path、audience、signing key 和 authorization pipeline。

### 5.2 Actor permissions

| Action | Recommended Release 1 rule |
| --- | --- |
| Generate key | Founder 或該 case 的 active Primary Advisor；「任何 Advisor 是否可生成」留待 `DP-01` |
| Select expiry | 生成者必須明確選擇 `expires_at`；不可為 null 或靜默永久有效 |
| View key metadata | Founder、Primary Advisor；只顯示 fingerprint、viewer、issued/expires/status，不可重新顯示 raw key |
| Revoke | Founder 或 active Primary Advisor；立即失效所有由該 grant 建立的 portal sessions |
| Rotate | 建立新 grant 並原子 revoke 舊 grant；不可改寫舊 secret hash |
| View portal | 只有持有有效 portal session 且 grant、case、viewer relationship 在 request time 均有效的 external viewer |

### 5.3 Portal visible scope

建議 Release 1 最小 read-only allowlist：

- case number、customer-facing case stage、last customer-visible update time；
- 已批准對客顯示的 SchoolTarget 名稱與狀態；
- 家長／申請者需要完成的 action item、deadline 和完成狀態；
- 已由 Founder／Primary Advisor 標記為 `customer_visible` 的公告／進度訊息。

預設拒絕：internal notes、identity contact、其他 Guardian 資料、Advisor 私人資料、協作者資料、原始 assessment answers、audit、history gaps、pricing/contract、文件 bytes/download、export、comment/edit/delete。任何新增資料類別需提升 `capability_set_version`，舊 grant 不自動看到。

### 5.4 State transitions 與 invariants

```text
generated(active) -> expired
generated(active) -> revoked
generated(active) -> rotated (old revoked + new active)
```

- `issued_at < expires_at`，expiry 由 Founder/Advisor 選擇，但 hard maximum 尚待 `DP-02` 批准。
- case closed、cancelled、pending delete、viewer relationship 無效、organization suspended/terminated、issuer 被停用等是否立即 revoke，按 `DP-03/DEC-060` 決定；未決前 production activation blocked。
- 每次 portal read 在同一 repository transaction 重新檢查 grant status/expiry、portal session、case/viewer/organization scope；不得只信 cookie claim。
- revoke/expiry 在 request time 立即拒絕；cache key 必須含 grant/capability version，authenticated portal response `private, no-store`。
- generate/redeem/read/revoke/rotate、denied 和 rate-limit outcome 寫入不含 raw secret/PII 的 audit；raw key、cookie、URL、form value不得進 log/telemetry。
- redeem endpoint 需 IP/device-independent bounded rate limit、constant-shape error、secret rotation pepper、replay/credential-stuffing alert；不得回覆「grant 存在但已過期」等可枚舉差異。

### 5.5 Module interface 與 error contract

```text
issuePortalGrant(actor, caseId, viewerRef, expiresAt, idempotencyKey)
  -> { grantId, rawSecretOnce, fingerprint, expiresAt, recordVersion }

redeemPortalSecret(rawSecret, requestContext)
  -> { portalSessionSecret, absoluteExpiresAt }

getPortalWorkspace(portalSessionSecret)
  -> { caseSummary, schoolTargets, actionItems, messages, capabilityVersion }

revokePortalGrant(actor, grantId, expectedVersion, reason, idempotencyKey)
  -> { grantId, status: revoked, recordVersion }
```

Stable errors：`PORTAL_SECRET_INVALID`、`PORTAL_GRANT_EXPIRED`（只可在已建立 session 後使用）、`PORTAL_GRANT_REVOKED`、`PORTAL_SCOPE_DENIED`、`PORTAL_VIEWER_RELATIONSHIP_INACTIVE`、`PORTAL_RATE_LIMITED`、`PORTAL_VERSION_CONFLICT`、`PORTAL_RUNTIME_UNAVAILABLE`。Public redeem 對 invalid/expired/revoked 統一回 generic `401`，避免枚舉。

### 5.6 Pages and routes

- Internal：`/cases/[caseId]/access`，生成、顯示一次、copy、到期日、fingerprint、revoke/rotate、audit link。
- External：`/portal/access`、`/portal/cases/[caseId]`、`/portal/logout`。
- Route Handlers：`/api/v1/cases/[caseId]/portal-grants/**`、`/api/v1/portal/sessions/**`、`/api/v1/portal/workspace`。
- Portal layout 不顯示 internal navigation；所有 loading/empty/error/expired/denied/mobile/keyboard/focus states 必須獨立驗證。

## 6. 平台管理員與月度收費通知

### 6.1 PlatformOperator authorization

`PlatformOperator` 不屬於任何 CustomerOrganization membership。平台管理頁只查 `PlatformBilling` 投影，不允許直接切換 tenant 或讀 Student、Guardian、Case detail、Document、notes、knowledge。

建議角色：

- `platform_admin`：查看租戶運行狀態與聚合數字；不能修改合同或生成正式通知。
- `platform_finance`：管理已批准合同版本、生成 charge notice draft。
- `platform_billing_approver`：以不同 actor 批准/作廢 draft；是否允許與 Founder 同 actor 由 `DP-08` 決定。

Platform support access 仍由 `DEC-060` 控制，不可利用 billing page 繞過 support grant。

### 6.2 Data model

```text
CustomerContract
  id, organization_id, contract_number
  currency, contract_value_minor, pricing_policy_version
  effective_from, effective_to?, status(draft|active|superseded|terminated)
  approved_by, approved_at, record_version

MonthlyTenantMetric
  organization_id, billing_month
  advancing_case_count, count_policy_version
  source_cutoff_at, source_projection_version, generated_at

MonthlyChargeNotice
  id, organization_id, billing_month, contract_version_id
  metric_snapshot, pricing_policy_version
  subtotal_minor, adjustments_minor, tax_minor?, total_minor
  status(draft|approved|voided|superseded)
  generated_by, approved_by?, approved_at?, record_version
```

Money 使用 integer minor unit + ISO 4217 currency；禁止 float。合同版本不可原地改寫；生效期不得重疊。每個 organization + billing month 最多一個 current approved notice，correction 建立新 revision 並 supersede 舊版。

### 6.3 指標定義

管理頁最少顯示：

- tenant 總數，按 subscription status 分組；
- 每租戶 `advancing_case_count`；
- active contract 單額與 currency；
- 本月 projected charge、notice status、最後生成／批准時間；
- missing contract、count anomaly、pricing policy mismatch、past-due/suspended 等 exception。

`advancing_case_count` 不可用 UI 臨時計數。由 Cases module 發出不含 PII 的 case lifecycle projection event，PlatformBilling 按批准的 count policy 建立可重建 projection。哪些 case stage 算「正在推進」、月內新增／結案／暫停如何計費是 `DP-06`，未批准前只顯示 raw stage-group counts，不生成金額。

### 6.4 Charge notice workflow

```text
open month
  -> snapshot metrics at approved cutoff
  -> generate draft using pinned contract + pricing policy
  -> finance review
  -> approve OR void
  -> optional manual delivery outside platform

correction -> new draft revision -> approve -> prior notice superseded
```

- 「收費通知」不是稅務發票；法律名稱、格式、編號、稅項及發送渠道由 `DP-07` 決定。
- generate 是 deterministic pure calculation + immutable snapshot；相同 idempotency key 和相同 inputs 只能產生同一結果。
- missing/overlapping contract、unknown currency、unclosed metric snapshot、policy version mismatch 必須 fail closed，不可估算或填零。
- draft/approve/void/supersede 都需 audit；approval failure 不可留下部分 approved state。
- notice artifact 若生成 PDF，bytes 存 private S3 香港並使用短效下載；本計畫首版可只提供 HTML/JSON preview，避免提前引入文件 retention 語義。

### 6.5 Module interface 與 pages

```text
getPlatformBillingOverview(actor, month, filters)
  -> { tenantCounts, tenants[], exceptions[] }

closeTenantMetricSnapshot(actor, organizationId, month, cutoff, idempotencyKey)
  -> { snapshotId, advancingCaseCount, policyVersion }

generateMonthlyChargeNotice(actor, organizationId, month, idempotencyKey)
  -> { noticeId, calculation, status: draft, recordVersion }

approveMonthlyChargeNotice(actor, noticeId, expectedVersion, idempotencyKey)
  -> { noticeId, status: approved, recordVersion }
```

Pages：`/platform/tenants`、`/platform/tenants/[organizationId]/billing`、`/platform/billing/[month]`。Routes：`/api/v1/platform/tenants/**`、`/api/v1/platform/billing/**`。所有 routes 只能使用 platform actor context，不能接受 browser-provided organization ID 作授權真相。

Stable errors：`PLATFORM_ROLE_REQUIRED`、`BILLING_CONTRACT_MISSING`、`BILLING_CONTRACT_OVERLAP`、`BILLING_PERIOD_NOT_CLOSED`、`BILLING_POLICY_MISMATCH`、`BILLING_ALREADY_APPROVED`、`BILLING_VERSION_CONFLICT`、`BILLING_RUNTIME_UNAVAILABLE`。

## 7. 前端由 Vercel 遷移至 AWS 香港

### 7.1 Application changes

1. 在 `next.config.ts` 啟用並驗證 `output: "standalone"`。
2. 建立 multi-stage Dockerfile，包含 standalone server、`public/`、`.next/static/`；non-root、bounded CPU/memory、read-only filesystem where compatible。
3. 建立 `/api/v1/health` readiness；只在必要 dependency/config 可用時 ready，不查或洩漏 business data。
4. 所有 dynamic authenticated HTML/API 設 `private, no-store`；portal 與 internal cookie 分離。
5. 同一 deployment 的 tasks 固定 Git SHA、image digest、build ID、`deploymentId`、Server Action encryption key；驗證 mixed-version drain/rollback。
6. 盤點 ISR、`use cache`、`revalidateTag`、image optimization 和 filesystem cache；未有 shared invalidation 前禁止會造成跨 task stale 的 path。
7. 將 legacy `/api/**`、Neon、mock pages 遷移至 `/api/v1/**` 和 production RDS composition root；production 缺 adapter 一律 typed `503`。

### 7.2 Infrastructure changes

- Cloudflare `app.<domain>` 維持 DNS-only，指向 AWS ALB；Cloudflare 不接收 ERP request/cookie/private response。
- ACM、WAF、ALB 在 `ap-east-1`；ECS web tasks 在至少兩個 private subnets／AZ，無 public IP。
- RDS PostgreSQL 17 Multi-AZ private；app role `NOBYPASSRLS`，tenant transaction 使用 transaction-local organization context。
- Cognito User Pool `ap-east-1` regional/provider domain；不建立依賴 `us-east-1` ACM/CloudFront 的 custom domain。
- private S3/KMS/SQS/DLQ/scanner 均在香港；無香港 endpoint 的 workload service 不採用。
- CloudWatch/CloudTrail/audit/log/backup 香港化；application log 30 天、audit 1 年，PII allowlist/redaction。
- ECR immutable digest、SBOM/scan/provenance；GitHub Actions 使用 OIDC narrow role，不保存長期 AWS access key。

### 7.3 Migration sequence

```text
inventory
  -> local contracts/tests
  -> production IaC plan (no apply)
  -> nonprod >=2-task deployment
  -> RDS/RLS/Cognito/S3/SQS integration evidence
  -> production shadow + exact payload approval
  -> empty production schema
  -> approved image deployment
  -> callback/cookie/browser/security/restore gates
  -> DNS cutover to ALB
  -> AWS observation
  -> Vercel Git auto-deploy/domain/integration retirement
```

Cutover 前盤點 Vercel project/team、env **names only**、domain、stores、cron/queue/workflow、integrations 和 log drains；不得執行會輸出 secret 的 env pull。Cutover 前停掉任何 Vercel producer/cron 並驗證 drain。Cutover 後 rollback 只回 AWS prior compatible ECS task definition/image digest，不能把 Vercel 當 standby 或 rollback target。

## 8. Configuration contract

以下只記配置名稱、owner 和用途；secret value 只進 AWS Secrets Manager／受批准 CI store，不進 Git、聊天、plan artifact 或 evidence。

| Configuration | 類型／位置 | 用途與約束 |
| --- | --- | --- |
| `AWS_REGION=ap-east-1` | ECS runtime non-secret | 啟動時拒絕其他 region |
| `APP_BASE_URL`、`ALLOWED_HOSTS` | ECS runtime | 固定 AWS app origin、callback、trusted host |
| `DATABASE_HOST/PORT/NAME/USER` | ECS + Secrets Manager ref | private RDS；不得使用 `DATABASE_URL` Neon fallback |
| `COGNITO_USER_POOL_ID/CLIENT_ID/ISSUER` | ECS runtime | 必須屬 `ap-east-1` approved pool/domain |
| `INTERNAL_SESSION_HASH_KEY_REF` | Secrets Manager ARN | internal opaque session hash/rotation |
| `PORTAL_SECRET_PEPPER_REF` | Secrets Manager ARN | portal key keyed hash；與 internal session key 分離 |
| `PORTAL_SESSION_HASH_KEY_REF` | Secrets Manager ARN | portal session；與 grant pepper 分離 |
| `PORTAL_MAX_GRANT_DURATION` | policy config | 待 `DP-02` 批准；缺失時禁止 issue grant |
| `PORTAL_REDEEM_RATE_LIMIT_POLICY` | versioned config | 不記 raw key/IP 到一般 telemetry |
| `DOCUMENT_BUCKET/KMS_KEY_ARN/QUEUE_URL/DLQ_URL` | ECS/worker refs | 全部 `ap-east-1` private resources |
| `AUDIT_SINK_REF`、`TELEMETRY_SINK_REF` | ECS runtime refs | audit fail-closed；telemetry fail-open degraded |
| `BILLING_TIMEZONE=Asia/Hong_Kong` | policy config | month cutoff 和 display；計算保存 UTC instant |
| `BILLING_POLICY_VERSION` | versioned config | 未匹配 approved contract 時 fail closed |
| `BILLING_CUTOFF_DAY` | policy config | 待 `DP-06`；不可由 UI 任意改寫 |
| `R2_CATALOGUE_BASE_URL` | ECS runtime | server-side public catalogue only；不得轉發 cookie/auth/query |
| `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` | build/release secret | 同 deployment tasks 一致；rollback window 保留相容 key |

啟動 preflight 應驗證 region、host、resource ARN、required adapter、policy version 和 schema version；任何 production mismatch 使 task not-ready，不 fallback 至 Vercel、Neon、local JSON 或 in-memory adapter。

## 9. Data migrations and ownership

建議新增 additive migrations：

1. `011_expand_external_portal_access.sql`：viewer、grant、portal session、indexes、expiry/revoke constraints、RLS、audit/outbox references。
2. `012_expand_platform_billing.sql`：platform actor、contract version、monthly metrics、notice revision、money/currency/period constraints。
3. `013_expand_case_billing_projection.sql`：Cases-owned outbox event／projection checkpoint，不讓 Billing 直接寫 Cases tables。
4. `014_enable_multiple_active_organizations.sql`：只在第二租戶 activation gate 通過後，移除／取代 `access_organizations_one_active_idx`；不可與 billing UI migration 綁成同一次批准。

Data integrity：

- 所有 tenant-related rows 必須有 non-null `organization_id`；composite FK 保證 case/viewer/contract locality。
- Portal grant 的 `secret_hash` 全局唯一；raw secret 永不持久化。
- 同一 grant 同時最多一個 active secret；rotate 以單 transaction revoke old + create new。
- contract effective dates 不重疊；money minor unit non-negative，adjustment 必須有 reason/actor/audit。
- monthly snapshot 綁定 immutable cutoff、case-count policy 和 source projection version。
- RLS application role 不可讀 platform-control tables；platform runtime 使用獨立、最小 privilege DB role，不能繞過 tenant detail RLS。

Production migration 只可對 exact checksum payload 另行批准；先在 isolated PostgreSQL 驗證 migration、drift、RLS pool reuse、rollback/forward repair 和 zero-row baseline。

## 10. Verification and acceptance evidence

| Boundary | Minimum deterministic evidence |
| --- | --- |
| Existing CRM completion | 每個 `/api/v1` interface 的 production PostgreSQL integration tests；不存在 adapter 時 typed `503`；無 Neon/mock fallback |
| Portal issue/redeem | entropy/hash/no-plaintext、generic invalid response、rate limit、idempotency、single-display、expiry boundary tests |
| Portal authorization | cross-tenant/cross-case/other-sibling/ID guessing、revoked/expired/session-version、closed-case、cache、direct route 負向測試 |
| Portal UI | desktop/mobile、keyboard/focus、screen reader label、long/empty/loading/error/expired/denied；network/console 無 secret/PII |
| Billing aggregate | platform-only role、tenant PII absence、projection rebuild、stage count policy、month cutoff/timezone、multi-currency rejection |
| Charge notice | contract overlap/missing、integer rounding、idempotency、concurrent approve `409`、correction/supersede、audit failure rollback |
| Multi-tenancy | unprivileged DB role、FORCE RLS、pool reuse、search/export/S3/job/cache cross-tenant denial |
| AWS runtime | pinned standalone build、two-task skew/cache/drain、WAF/ALB/cookie/callback、load P95、no public RDS/S3/task |
| Deployment | plan hash、image digest、SBOM、config checksum、migration ledger、canary、AWS-only rollback receipt |
| Recovery | RDS PITR isolated restore、document metadata/bytes hash、audit continuity；測量 RPO/RTO |
| Privacy | zero raw key/token/cookie/PII in app log、telemetry、R2、Vercel inventory、screenshots/evidence |

不得以只通過 unit tests 宣稱 portal 或 billing production-ready。`pnpm lint`／`pnpm build` 依 repository 規則仍需使用者明確授權；未運行時必須在 release evidence 列為 gap。

## 11. Implementation sequencing

以下票據已整合進既有 Phase 3 dependency graph；保留 `R1X-*` namespace，避免重排既有 `P3-*` evidence identity。

| Ticket | Single outcome | Dependencies | Evidence / gate |
| --- | --- | --- | --- |
| `R1X-00` | 同步 DEC/PRD/phase authority，正式批准新增 Release 1 scope | `P3-01`；使用者 scope/architecture approvals；`DEC-064-066` | 所有 authority 文件一致；documentation gate；已於 2026-08-11 完成 |
| `R1X-01` | Portal contract、policy、migration 和 fakes | `R1X-00`、`DP-01-05/10` | identity/expiry/scope/revoke/error matrix；schema/RLS/function-only lookup tests |
| `R1X-02` | Portal grant issue/revoke/rotate internal interface | `R1X-01`、production Access/Identity repositories | one-TX authz/audit/idempotency；raw key single-display |
| `R1X-03` | Portal redeem/session/workspace read interface | `R1X-01/02` | generic auth failure、rate limit、request-time grant/case/viewer checks |
| `R1X-04` | Internal access page + external portal pages | `R1X-02/03` | browser/a11y/mobile/secret leakage/denied state evidence |
| `R1X-05` | Platform actor、contract、metrics、billing migrations/contracts | `R1X-00`、`DP-06/07/08/10` | money/effective-date/RLS/role/separate-platform-audit/error tests |
| `R1X-06` | Cases -> Billing aggregate projection | `R1X-05`、production Cases/Audit repositories | rebuild/idempotency/no-PII/no direct cross-module write |
| `R1X-07` | Monthly snapshot + charge notice draft workflow | `R1X-05/06` | deterministic calculation/version/cutoff/concurrency/audit evidence |
| `R1X-08` | Platform tenant/billing admin UI | `R1X-07` | aggregate-only browser/authorization/empty/error/export-denied evidence |
| `R1X-09` | Production composition root and all core RDS/effect adapters | existing `P3-08/09` scope + `R1X-02/03/06/07` + `DP-09/10` | no adapter fallback、RLS、transaction、partial failure evidence |
| `R1X-10` | AWS nonprod two-task end-to-end suite | `P3-07` IaC、`R1X-04/08/09` | auth/portal/billing/docs/cache/load/drain/rollback/restore |
| `R1X-11` | Exact-approved production deploy and empty-tenant gate | all prior + `DP-09-11` + existing Phase 3 approvals | zero business rows、pinned receipts、manual go/no-go |
| `R1X-12` | Vercel retirement after AWS observation | successful cutover/observation + `DP-12` | producer drain、domain/Git disconnect、secret rotation；deletion separate approval |

推薦順序：`P3-01` 與 `R1X-00` 已完成；下一步先批准 `R1X-01`／`R1X-05` 各自 DP，再讓 Portal 與 Billing contracts 並列實作。Production repository/composition、nonprod integration、production gate 必須 join 後才可部署。不要為視覺上的「並行」引入多 agent workflow。

## 12. 必須批准的決策點

| ID | 決策問題 | 建議 baseline | 未決影響 |
| --- | --- | --- | --- |
| `DP-01` | 哪些 Advisor 可生成／撤銷 portal key？ | 只有 active Primary Advisor；Founder 全局可操作 | authorization owner 不明時不可實作 generate |
| `DP-02` | 生成者可選 expiry 的 hard maximum 是多少？ | 必須選明確日期；平台 maximum 建議 90 天，可生成者選更短；續期建新 grant | 無 maximum 會形成長期 bearer credential；production blocked |
| `DP-03` | case 結案/取消、關係失效、issuer 停用、tenant past_due 時如何處理 grant？ | revoke/relationship invalid 立即 deny；case closed 是否保留只讀由 Product/Legal 決定 | 改變資料可見性與 retention |
| `DP-04` | Portal 可見哪些 field/document？ | 採第 5.3 節最小 allowlist，文件/download 全拒絕 | 不可由 UI 自行暴露新增欄位 |
| `DP-05` | Key 可否多裝置、多次兌換；是否需第二因素？ | 可重複兌換但 session 數有界；高敏感 scope 另決第二因素 | 影響 credential-sharing、support 和 UX |
| `DP-06` | 「正在推進的 case」及月度計價公式 | 先固定 stage allowlist、cutoff、partial-month、暫停/結案規則，再 version pricing policy | 未批准前只能顯示 counts，不能生成金額 |
| `DP-07` | 合同單額含義與收費通知法律定位 | 明定是 contract total、monthly base、per-active-case rate 或組合；Release 1 只產 draft notice，不稱 invoice | 直接影響收入、稅務、currency、rounding |
| `DP-08` | 誰可修改合同、生成、批准和發送 notice？ | Finance 生成，另一 actor 批准；平台不自動發送 | segregation of duties 和 audit 未定 |
| `DP-09` | Subscription `past_due/suspended/terminated` 對 CRM、portal、export 的效果 | 依 `DEC-060` 另行批准；不得以 billing code 默認鎖客戶資料 | 第二 tenant activation blocker |
| `DP-10` | 合同、notice、portal grant/session 的 retention 和 data-subject handling | Legal/Privacy 按分類批准；secret/session metadata 最小化保留 | purge、audit、backup expiry 無法完成 |
| `DP-11` | 第二個 tenant 何時可 production activation？ | 完成 `DEC-060`、RLS/negative tests、support/termination/restore gates後另行 go/no-go | Admin UI 存在不等於 multi-tenant launch |
| `DP-12` | Monthly notice 的人工交付渠道與 receipt | 首版平台外人工傳送，只在系統記 opaque delivery receipt；不保存通訊內容 | 自動 Email/SMS 仍不在 Release 1 |

### 12.1 已批准的 architecture decisions

1. Portal redeem 使用獨立 `portal_auth` PostgreSQL role。該 role 沒有 Portal table direct access、table owner 或 `BYPASSRLS`，只可呼叫固定 `search_path`、無 dynamic SQL、最小返回值的 `SECURITY DEFINER` keyed-hash equality lookup；lookup 後另開 tenant-scoped transaction 做完整 request-time authorization。`[DEC-065]`
2. PlatformOperator audit 使用獨立 `platform_audit_events` aggregate，不把 tenant audit 的 `organization_id`／`actor_user_id` 改成 nullable，也不偽造 OrganizationMembership。PlatformBilling mutation 與 platform audit 同 transaction fail closed，兩個 DB roles 互相不可讀對方 detail tables。`[DEC-066]`

## 13. Risks、budgets、stopping conditions

| Risk | Control / stop condition |
| --- | --- |
| Portal bearer key 洩漏 | 不進 URL/log/telemetry；single display、hash-only、rate limit、revoke；發現 raw key exposure 立即 revoke 並 incident stop |
| Portal 權限隨新欄位擴張 | versioned field allowlist；新欄位 default deny；舊 grant 不自動升級 |
| Platform admin 變成跨租戶後門 | aggregate-only projection、separate platform role/DB role、no tenant detail drill-down、負向測試 |
| 收費數字不可重建 | immutable month snapshot、contract/pricing/source versions、integer money、correction revision |
| AWS migration 雙寫或回 Vercel | bounded freeze/delta；cutover 後 AWS-only rollback；Vercel producer cutover 前 drain |
| Multi-instance Next.js 不一致 | build once、same keys/build/deployment ID、two-task skew/cache/drain tests |
| Production adapter 部分失敗 | business transaction + audit fail closed；effect outbox/idempotency/reconciliation；typed unknown-commit state |
| 第二 tenant 過早啟用 | `DEC-060` + `DP-06`–`DP-11` human gate；schema capability 不等於 activation authority |

Loop budget：每張 local ticket最多一次基於 deterministic counterexample 的修正後重跑；同一 failure 再重複兩次且無新 evidence，terminal state 為 `needs_human`。Cloud apply、migration、DNS、deploy、invitation、notice delivery 和 Vercel destructive action不自動重試，每次要求 exact payload approval。

## 14. Definition of done

Release 1 只有同時符合以下條件才可稱為可交付：

1. Authority 文件正式批准新增 portal/billing scope，且所有第 12 節 production-blocking 決策有 owner/version。
2. Existing CRM core 不再依賴 production mock/Neon；所有 runtime 注入 HK RDS/effect adapters，缺失時 fail closed。
3. Portal 的 single-case、read-only、expiry、revoke、request-time authorization、secret hygiene 和 browser evidence 全部通過。
4. Platform admin 只顯示 aggregate/billing facts；monthly notice 可由 pinned contract/policy/snapshot deterministic 重建。
5. AWS `ap-east-1` two-task application、Cognito、RDS、S3/SQS/audit/telemetry 通過 security、residency、load、rollback、restore evidence。
6. Production 首先保持 business-data-empty；第一批案件、第一個 portal grant、第二 tenant 和第一份 charge notice 各自需要獨立 human go/no-go。
7. AWS observation 通過後停止 Vercel deployment/producer；任何 Vercel deletion/cancellation 仍是獨立 destructive approval。

本文不是 release approval。它把「要寫什麼、由誰 enforce、以什麼 evidence 判定、哪些決策不能由工程自行發明」固定為下一輪 authority review 和 implementation 的共同基線。
