# TD-002：AWS 敏感核心、Cloudflare 公開學校情報與 Vercel 遷移部署設計

| 屬性 | 內容 |
| --- | --- |
| 版本 | `v1.1` |
| 日期 | 2026-08-12（Asia/Hong_Kong） |
| 狀態 | 架構方向與 `DEC-068` source/plan baseline 已批准；Terraform plan/apply、註冊、採購、provision、migration、DNS、deploy 和刪除仍需 exact-payload 批准 |
| Run ID | `TD-AWS-CF-20260811-v1` |
| 決策依據 | `DEC-018`–`DEC-024`、`DEC-050`、`DEC-053`–`DEC-057`、`DEC-060`、`DEC-063`、`DEC-068` |
| 系統 | `erp-frontend/` Next.js 16.2.7；`automated_tracker_for_school_listing/hk-school-platform/` crawler |
| 研究附件 | `research/2026-08-11_AWS_CLOUDFLARE_ACCOUNT_REGISTRATION_AND_VERCEL_MIGRATION.md`、`research/2026-08-11_AWS_CLOUDFLARE_HYBRID_COST_RESIDENCY_CHECK.md` |

## 1. Outcome 和範圍

本設計固定兩個資料和部署平面：

1. **Cloudflare Public School Intelligence Plane**：只保存並分發已審核、已批准公開的全港學校招生資料 release。
2. **AWS Hong Kong Authenticated Tenant Plane**：保存和處理 CustomerOrganization、User、Membership、Student、Guardian、Case、Document、私有知識、授權、audit、log 和 backup。

本次同時定義從目前 GitHub -> Vercel 的 Next.js deployment 遷移至 GitHub -> AWS ECR/ECS Fargate 的方法。

### 1.1 Out of scope

- 本文件不建立或登入 AWS、Cloudflare、Vercel、registrar 或 GitHub 帳戶。
- 不讀取或複製 `.env*`、secret value、credential、token、production data 或 DNS secret。
- 不批准 Terraform apply、database migration、R2 upload、DNS/NS change、domain transfer、deployment、customer invitation、Vercel cancellation 或 deletion。
- 不把 crawler review/fallback queue、source capture、LLM trace 或未批准資料定義為公開內容。
- 不解決 `DEC-060` 的第二租戶 subscription、retention、support grant、termination/export 或 region-outage 語義。

## 2. Target topology

`DEC-068` 固定 production source baseline：private ECS 跨兩個 AZ、每 AZ 一個 NAT Gateway，並使用 ECR、S3、Logs、SQS、KMS、Secrets Manager、STS endpoints；web task 為 `1 vCPU / 2 GiB`、desired/min `2`、max `4`。RDS 為 PostgreSQL 17、`db.t4g.small`、20 GiB gp3、Multi-AZ、7-day backup。WAF 使用 managed core/known-bad-input rules，rate limit 必須由外部 exact payload 提供。Account、backend、CIDR、ACM、image digest、notification recipients 不得有 Git production default。P3-10 只認 saved binary `.tfplan` SHA-256；其他 hashes 只作補充 evidence。本段不授權 Terraform plan/apply。

```text
Internet
  +-- data.<base-domain>                         PUBLIC ONLY
  |     -> Cloudflare TLS/WAF/rate limit/cache
  |     -> R2: tianxing-school-public-prod
  |          releases/<release_id>/records.json
  |          releases/<release_id>/manifest.json
  |          channels/production/latest.json
  |
  +-- app.<base-domain>                          AUTHENTICATED
        -> Cloudflare DNS-only record
        -> ACM + AWS WAF + ALB, ap-east-1
        -> ECS Fargate Next.js web/BFF, private, >=2 tasks / >=2 AZs
             -> Cognito, ap-east-1
             -> RDS PostgreSQL Multi-AZ, ap-east-1
             -> private S3/KMS documents, ap-east-1
             -> SQS/DLQ + scanner/workers, ap-east-1
             -> Secrets Manager / CloudWatch / CloudTrail / backup

Crawler publication
  candidate output
    -> audit + human publish gate
    -> PublicReleaseCompiler(field allowlist + schema + PII/licence checks)
    -> immutable R2 upload
    -> GET/read-back + SHA-256/count verification
    -> latest.json compare-and-set last
    -> AWS publication receipt
```

### 2.1 Hostnames

| Hostname | Target | Cloudflare mode | Contract |
| --- | --- | --- | --- |
| `app.<base-domain>` | AWS ALB | **DNS-only** | authenticated ERP；host-only Secure cookies；never cache |
| `data.<base-domain>` | R2 custom domain | Proxied | anonymous approved school releases only |
| Cognito provider domain | Cognito regional managed-login endpoint, `ap-east-1` | Not in Cloudflare zone | provider authentication only；no custom-domain CloudFront |
| `www.<base-domain>` | later public-site decision | May be proxied | no ERP cookie/private response |
| `status.<base-domain>` | approved status page | Separate | no customer payload |

Cloudflare 可作 authoritative DNS，但不得 proxy `app`，也不得接收 authenticated request、cookie、authorization header、private response、Server Action 或敏感 log。若轉 Cloudflare nameserver，先在不改 origin 的情況下完整複製和核對 MX/SPF/DKIM/DMARC/CAA/verification/Vercel records；registrar transfer 不與部署遷移綁定。

## 3. Data classification and authority

### 3.1 Cloudflare allowed set

Cloudflare 只可保存 `PublicSchoolAdmissionRelease`。初始建議 allowlist：

- `school_key`、中英文校名、地區、學校級別／類型／資助類型；
- 官方網站、招生頁、申請表和 evidence URL；
- 分離的 S1／transfer 類型；
- 申請、截止、考試、面試、結果通知日期；
- required materials、學費、宿舍、SEN 等已批准公共事實；
- 機構地址和電話，前提是 Product／Privacy／Legal 批准其公開和更正用途。

不得公開：

- `review_queue.json`、fallback queue、internal suggested action、reviewer/publisher identity；
- warning receipt 全文、internal path、run/host/trace ID、log、screenshot、PDF cache、LLM prompt/response；
- crawler tickets、config、review decisions、schedule/run control；
- Student、Guardian、Advisor、CustomerOrganization、Case、Document 或 tenant-derived data；
- 未批准 notes、可能含個人姓名的 contact/extract、受版權限制的長引文。

Public source 不自動等於可無限制再發布。首次 release 前需批准 field catalogue、provenance、quote/licence、correction/takedown SLA、retention 和 owner。

### 3.2 AWS retained set

- identity/session、Organization/Membership/RoleBinding/grant/support access；
- CRM、Case、Task、private overlay、knowledge/report；
- private document bytes、metadata、version、scan、export、retention/legal hold；
- crawler review/fallback queue、tickets、config、review decisions、run control；
- audit、telemetry、log、backup、restore evidence；
- R2 approval receipt、release hash、correction/takedown audit。

### 3.3 Authority states

| State | Authority |
| --- | --- |
| crawler candidate/evidence | crawler staged output；不能被 frontend/R2 當 truth |
| review decision | SchoolIntelligence review workflow/AWS control plane |
| publication approval | exact manifest hash + human receipt |
| active public release | R2 immutable release pointed to by validated `latest.json` |
| tenant correction/private note | AWS RDS tenant relation；不得直接改 global release |
| global correction | new reviewed release；不原地覆寫舊 release |

## 4. Module interfaces

### 4.1 `PublicSchoolCatalogue`

```text
getCurrentCatalogue(ifNoneMatch?)
  -> { releaseId, etag, publishedAt, records, staleState }
  -> UNAVAILABLE | INVALID_MANIFEST | HASH_MISMATCH | SCHEMA_UNSUPPORTED
```

Implementation 隱藏 R2 URL、ETag、timeout、manifest/hash/schema validation、last-valid in-memory cache 和 degraded behavior。正式 adapter 是 `R2CatalogueAdapter`；測試／遷移 adapter 是 `LocalSnapshotAdapter`。Caller 不知道 bucket/token/key layout。

初期保留 `GET /api/crawler/schools`，由 AWS BFF server-side module 讀 R2。它不得把 browser cookie、authorization header、query 或 PII 轉發 R2。R2 outage 只令 school selector/summary 進入 `public_catalogue_unavailable`，不得影響 identity、CRM、case 或 document mutation。

### 4.2 `PublicReleasePublisher`

```text
publishApprovedRelease(candidateRef, approvalReceiptRef)
  -> { releaseId, manifestSha256, objectReceipts, activatedAt }
  -> NOT_APPROVED | PUBLIC_SCHEMA_REJECTED | PII_REJECTED |
     UPLOAD_FAILED | READBACK_MISMATCH | ACTIVATION_CONFLICT
```

順序固定為 compile -> validate -> upload immutable objects -> read-back -> reconcile -> activate pointer -> audit。`latest.json` 永遠最後寫；之前任何失敗都保留上一 active release。Retry 使用 `release_id` idempotency key；相同 key 不接受不同 hash。

## 5. What to register

### 5.1 AWS

```text
AWS Organization management/payer        # no workloads
  Security OU
    log-archive/security                  # before customer PII
  Workloads OU
    tianxing-nonprod
    tianxing-prod
```

需準備：company legal name/address、tax/invoice data、company payment method/currency、company root distribution alias、independent recovery phone、Billing/Operations/Security contacts、domain ownership、support approvers 和 budget owner。

Required controls：

- management root 使用至少兩個可恢復的 phishing-resistant MFA，無 root access key；password/email/phone採兩人 break-glass custody；
- AWS Organizations all features、consolidated billing、centralized member-root management；
- IAM Identity Center named workforce identities和temporary credentials；
- permission sets至少有 `ReadOnly`、`DeveloperNonProd`、`DeployNonProd`、`DeployProd`、`SecurityAudit`、`Billing`；
- DeployProd、DB migration、DNS、KMS admin、audit、break-glass分離；
- GitHub Actions OIDC assume narrow role，禁止長期 CI access key；
- Cost Explorer、Budgets、Anomaly Detection、cost tags、invoice owner和current support-plan decision。

### 5.2 Cloudflare

- 一個公司控制 account、company alias/legal/billing/payment，至少兩名具 passkey/MFA 的 Super Administrator；
- Finance、DNS/release、auditor角色分離；
- 啟用 R2 subscription；記錄 Account ID、zone ID、bucket name、token fingerprint/owner，不記 token value；
- 使用 account-owned token，禁止 Global API Key；publisher token只允許一個 bucket所需 object operations，DNS/IaC token另建；
- registrar維持公司所有、MFA、lock、auto-renew和expiry alert；不因改DNS而強制轉registrar。

### 5.3 GitHub and optional providers

- 確認 GitHub organization/repository/billing ownership，建立 `nonprod`/`production` environments和production approver；
- ECR push/ECS deploy透過GitHub OIDC；
- SES/email、SMS、OCR、alerting等provider逐項取得region、DPA、cost批准；不得因AWS帳戶建立而假定已可用。

## 6. Cloudflare settings

| Resource | Setting |
| --- | --- |
| Bucket | `tianxing-school-public-prod`；prod/nonprod分離 |
| Access | custom domain only；disable production `r2.dev` |
| Keys | `releases/<release_id>/{records.json,manifest.json}` + `channels/production/latest.json` |
| Release cache | `public, max-age=31536000, immutable` |
| Pointer cache | `public, max-age=60, must-revalidate` + ETag |
| CORS | approved `GET`/`HEAD` origins/methods/headers only |
| Write | publisher token only；browser/public write denied |
| Lifecycle | initially no release deletion；retention approval precedes deletion rule |
| Multipart | abort incomplete multipart after approved short window |
| Controls | WAF/rate limit、request/error/cache metrics、spend alert、reconciliation |

`latest.json` 至少含 `schema_version`、`release_id`、`manifest_url`、`manifest_sha256`、`published_at`。Public manifest至少含 count、SHA-256/bytes/schema、source snapshot hash、previous release、approval receipt hash（無actor identity）、compiler/policy version和takedown version。

R2沒有等同S3 Versioning/Object Lock/KMS/event/inventory的完整contract，因此以immutable keys、external approval ledger、prior pointer和read-back reconciliation補足；不得用於敏感document domain。

## 7. AWS production settings

所有可選 region 的 AWS workload、data、identity、log、key、backup 和 support resources 必須位於 production account、`ap-east-1`。沒有 `ap-east-1` endpoint 的 AWS workload service不採用，也不得以「官方 constraint」靜默改用其他region。AWS Organizations、IAM、Billing等global control-plane products沒有region選項，只能處理account/permission/billing metadata，不得承載敏感business payload。

### 7.1 Network/runtime

- VPC >=2 AZ；ALB在public subnets，ECS/RDS在private subnets，task/RDS無public IP；
- Internet -> ALB 443 only；ALB SG -> task port only；task SG -> RDS 5432和approved endpoints only；
- 每 AZ 一個 NAT Gateway，並同時建立 ECR API/DKR、S3、Secrets Manager、CloudWatch Logs、KMS、SQS、STS VPC endpoints；逐項驗證可用性、route、policy 和計價；
- ACM `ap-east-1`、HTTPS redirect、TLS policy、ALB logs/deletion protection、WAF/rate limit；
- ECR immutable digest/tag、scan、SBOM/provenance、lifecycle；CI/execution/task roles分離；
- ECS Fargate >=2 web tasks、>=2 AZ、non-root、read-only filesystem where compatible、bounded resources；
- readiness、deregistration、10–30秒SIGTERM drain、circuit breaker、prior digest rollback、autoscaling和alarms。

目前 Terraform 的 `desired_count=1`、0.25 vCPU/0.5 GiB health task和health-only listener只可作staging foundation，不能原樣升格production。

### 7.2 Next.js self-hosting

- `next.config.ts`採經驗證的 `output: "standalone"`；multi-stage image包含standalone server、`public/`、`.next/static/`；
- build once；同deployment tasks使用同Git SHA、image digest、build ID和`deploymentId`；
- `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY`在build/release boundary一致，rollback/mixed-version window保留相容key；
- 盤點ISR、`use cache`、`revalidateTag`、image optimization、filesystem cache；沒有shared cache/tag coordination前限制會產生跨task stale的path；
- 測試streaming、body/upload limit、timeout、`X-Forwarded-*`、trusted host/origin、secure cookie、callback和drain；
- authenticated dynamic HTML/API必須private/no-store；Cloudflare不在`app` request path。

### 7.3 RDS/Cognito/documents

- RDS PostgreSQL 17 Multi-AZ、private、KMS、deletion protection、storage ceiling、backup/PITR、maintenance和isolated restore drill；
- app role non-owner + `NOBYPASSRLS`；tenant tables `ENABLE` + `FORCE RLS`；transaction-local context和real pool-reuse tests；
- master/migration/app roles分離；Secrets Manager管理credential；RDS Proxy只在量測和pinning test後加入；
- Cognito production pool/client在`ap-east-1`，exact callback/logout、TOTP/recovery、token/revoke、message delivery、deletion/export；
- Cognito只證明identity；RDS保留membership/role/case/capability/expiry/session truth；
- Cognito使用`ap-east-1` User Pool和經驗證的regional/provider domain。禁止建立需要`us-east-1` ACM certificate及global CloudFront的custom login domain；若regional managed-login data-flow不能通過香港邊界probe，authentication launch必須blocked，不得fallback到區外AWS service；
- private S3 Block Public Access、versioning、SSE-KMS、opaque keys、quarantine-first；SQS/DLQ bounded retry/idempotency/reconciliation；
- scanner/OCR使用separate role；Textract無`ap-east-1` endpoint時採HK self-hosted或另批cross-region/provider；
- Secrets只以ECS ARN reference；CloudWatch/CloudTrail/log archive、PII redaction、alarm/runbook；audit failure令mutation fail closed。

## 8. Vercel inventory without secrets

Repository只證明：minimal `vercel.json`、無self-hosting `next.config`、GitHub->Vercel、Git snapshot crawler、Neon crawler mutable state、無Dockerfile、AWS health-only staging。它不證明dashboard沒有env、domain、integration、Blob、KV/Postgres、Edge Config、Queue、Cron、Workflow或log drain。

Vercel owner以read-only metadata記錄：

| Area | Record only；never value |
| --- | --- |
| Project/team | opaque IDs、owners、plan、Git/branch/root、Node/pnpm |
| Environment | variable **name**、scope、build/runtime/public/secret、owner/date |
| Domain/DNS | registrar、nameserver、hostname/type/TTL/purpose、certificate |
| Store/integration | product、opaque ID、region、counts/bytes、owner/export plan |
| Cron/queue/workflow | schedule/topic、enabled、outstanding/in-flight、owner/idempotency |
| Observability | log drain、analytics、alerts、retention、destination |

不得執行會輸出secret的`vercel env pull`或提交`.env`。Secret owner直接將批准值寫入Secrets Manager/CI；evidence只保存name、target ARN和fingerprint。`NEXT_PUBLIC_*`是public build value，改動需rebuild。

## 9. Data/application migrations

### 9.1 Git snapshot -> R2

1. 建立versioned public schema/compiler；existing four-file manifest保留作crawler internal gate。
2. 以相同module interface提供local和R2 adapters。
3. 對`records.json`做field allowlist、PII/licence/provenance、schema/count/hash validation。
4. Upload immutable release、read-back，再更新`latest.json`；保留prior pointer。
5. `GET /api/crawler/schools`在feature flag後切R2；review/admin routes仍讀AWS private state。
6. 一個完整release cycle和failure rehearsal後才停止Git snapshot更新；local adapter作bounded rollback，不再是active truth。

### 9.2 Neon/Vercel mutable state -> RDS

- 盤點Neon schema/count/owner，不假設只有四個crawler tables；
- `crawler_tickets`、`crawler_review_decisions`、`crawler_config`、`crawler_runs`由request-time auto-DDL改成immutable migration和owning repository；
- global school revisions與tenant overlay/ticket/case link分離，禁止null/zero tenant sentinel；
- isolated rehearsal驗證extensions、roles、owners、RLS、constraints、counts/hash、sequence/timezone、query plan和rollback/delta；
- final copy在empty production gate；已有live writes時用freeze或monotonic delta，不做無限dual-write；
- production adapter缺失時fail closed，不fallback Neon/local/mock。

### 9.3 Vercel runtime -> ECS

- 建立multi-stage Dockerfile、standalone config、health/readiness和production entrypoint；
- GitHub OIDC pipeline：approved build -> ECR digest -> nonprod -> evidence -> manual prod approval -> ECS revision；
- image promotion不rebuild，runtime config用task definition/Secrets Manager；
- nonprod先通過two-task、auth、RDS、S3、queue、cache、streaming、drain、rollback、load和negative authz。
- Vercel只作migration source inventory。AWS cutover後不得再從GitHub觸發Vercel production/preview deployment，也不得把Vercel當runtime standby或rollback target。

## 10. Migration state machine and phases

```text
discovery -> accounts_ready -> nonprod_ready -> production_shadow
 -> data_reconciled -> cutover_approved -> aws_active -> observation
 -> vercel_retired

pre-cutover failure -> needs_human / rollback_to_previous_state
exposure/mismatch   -> incident_stop
incompatible writes -> forward_repair_or_restore
```

### Phase A：discovery

1. 批准public field/licence、DNS owner、account topology、budget、RPO/RTO、Cognito domain choice。
2. 完成Vercel/DNS/data-store metadata inventory，確認Blob/KV/Postgres/Queue/Cron/Workflow/integration/log drain。
3. 固定acceptance：auth、5xx、P95、DB pool、DLQ、audit、zero PII/cross-tenant和zero unexplained diff。

### Phase B：company control plane

4. 註冊/保護AWS Organization、Cloudflare account、billing/MFA/roles/support/budgets。
5. 建Terraform remote state、GitHub OIDC roles和Cloudflare publisher token。
6. 若轉Cloudflare DNS，先原樣複製zone，不改Vercel origin。

### Phase C：non-production

7. Build pinned standalone image並部署nonprod >=2 tasks。
8. Provision nonprod AWS services和synthetic R2 release。
9. 通過Next.js multi-instance、RLS/pool、scanner、restore、callback/cookie和rollback tests。

### Phase D：production shadow

10. P3-07 source review 後，另取 exact production payload/tooling approval 執行 P3-07A saved `terraform plan -out`；保存 binary `.tfplan` SHA-256，另產 redacted summary/review evidence、image digest、env-name map、migration checksum、DNS payload和cost。Payload 或 binary hash 改變使批准失效；不得 apply。
11. Exact approval後provision；ALB先用restricted validation hostname。
12. Rehearse Neon/RDS和snapshot/R2；驗證AWS consumer degraded behavior。

### Phase E：cutover

13. 24–48小時前只降低`app` TTL；保留registrar和prior DNS value作evidence，但Vercel不是cutover後rollback target。
14. Add AWS callback/logout，deploy approved digest，run empty-tenant/synthetic/pilot checks。
15. Freeze/bound writes、final delta/reconcile；每個cron/queue只有一個producer。
16. Approval後只改`app.<base-domain>`到ALB；未證明write/session兼容前不做雙平台percentage routing。
17. 按error/P95/auth/DB/audit/queue observation。

### Phase F：decommission Vercel

18. 在AWS cutover前停Vercel cron/queue/producer並驗證drain，防止cutover後仍有Vercel side effect。
19. AWS observation passed後，保留允許的log/invoice/build evidence和data export，再移除domain binding/Git auto-deploy；不產生任何新Vercel deployment。
20. Rotate Vercel曾可讀的secrets，revoke integration/member。
21. Vercel deletion/cancellation需另一次exact-inventory destructive approval。

## 11. Rollback contracts

| Failure | Immediate action | Rollback |
| --- | --- | --- |
| R2 upload/hash fail | 不更新`latest.json` | previous release remains active |
| R2 outage/malformed latest | catalogue degraded；CRM unaffected | last-valid memory where available |
| ECS/ALB 5xx/P95 fail | cutover前stop；cutover後AWS incident state | AWS prior ECS task-definition/image digest；不得DNS回Vercel |
| login/cookie/callback fail | stop traffic；redacted evidence | config/DNS revert only if compatible |
| migration/RLS/hash fail | isolate；no ad-hoc SQL | no cutover；corrected rehearsal |
| duplicate cron/queue | disable new producer | idempotency/reconciliation |
| cross-tenant/PII exposure | incident stop/revoke/evidence | Security/Privacy decision required |
| incompatible post-cutover writes | no blind DNS rollback | compatible ECS or forward repair/restore |

Cutover後的`rollback_available`只表示AWS內可回到prior compatible ECS digest；不表示可回Vercel。Schema/write不相容後轉為`forward_repair_or_restore`。

## 12. Cost/operations guardrails

- AWS planning envelope：Pilot `$300–650/month`，Growth `$650–1,900`，Scale `$1,600–6,400`；不是quote。
- R2公開school dataset主要價值是release更新與Next deploy解耦，不是改變敏感平台總成本。
- Quote同時包含Fargate、ALB/WAF、RDS HA、S3/KMS/SQS、每 AZ NAT Gateway、指定 VPC endpoints、logs/audit、backup、scanner/OCR、support、R2、Cloudflare plan和tax。
- 50/80/100% budget thresholds、anomaly alert、owner和stop action在production前批准。

## 13. Registration/configuration checklist

### Register/purchase

- [ ] Company registrar/domain、renewal、MFA和DNS owner。
- [ ] AWS payer、nonprod、prod、security/log accounts及billing/tax/payment。
- [ ] AWS support plan和authorized contacts。
- [ ] Cloudflare company account、billing/plan、R2、two Super Administrators。
- [ ] GitHub organization/CI ownership和production approver。
- [ ] Email/SMS/OCR/alerting provider另行data/region/cost批准。

### Configure before production

- [ ] Root/recovery MFA、Organizations、Identity Center、contacts、OIDC CI、break-glass。
- [ ] Budgets/anomaly/tags/invoice和support。
- [ ] Cloudflare zone、`app` DNS-only、`data` R2、ACM validation、TTL/rollback。
- [ ] R2 bucket/token/cache/CORS/lifecycle/manifest/reconciliation/takedown。
- [ ] VPC、每 AZ NAT Gateway、ECR/S3/Logs/SQS/KMS/Secrets Manager/STS endpoints、SG、ECR/ECS/ALB/WAF/health/drain/autoscale。
- [ ] RDS HA/RLS/backups/restore、Cognito `ap-east-1` callback/MFA/session/revoke/provider-domain probe；no custom domain。
- [ ] private S3/KMS/SQS/DLQ/scanner/Secrets、CloudWatch/CloudTrail/alarms/runbooks。
- [ ] Vercel inventory、secret-name map、Neon migration、Blob/Queue/Cron pre-cutover drain、Git deployment disconnect和decommission。

## 14. Evidence and tickets

| Gate | Evidence | Owner |
| --- | --- | --- |
| Public schema | allowlist、PII/licence/provenance、takedown/retention | Product + Privacy/Legal |
| Ownership | aliases、opaque IDs、roles/MFA/contacts/billing；no credentials | Founder + Security + Finance |
| Infra | saved binary `.tfplan` SHA-256、separate redacted summary、account/region/resource/cost manifest | Infrastructure + Security + Budget |
| Container | SHA、digest、build/deployment ID、SBOM、scan、task revision | Release |
| Next.js | 2-task Actions/cache/tag/skew/stream/drain/rollback | App + Operations |
| Identity/DB | callback/MFA/revoke/negative authz；migration/RLS/pool/restore | Identity + Data + Security |
| Private docs | private/encrypted/version/scan/DLQ/replay/restore | Document + Privacy |
| Public release | schema/count/SHA/readback/immutable/latest/cache/CORS/takedown | SchoolIntelligence + Product |
| Cutover/exit | DNS prior/current、thresholds、rollback point、producer drain、rotation | Release + Operations |

Suggested implementation tickets：

1. `DPL-01` public field/manifest contract。
2. `DPL-02` `PublicSchoolCatalogue` local/R2 adapters。
3. `DPL-03` publisher compiler + fake/contract tests。
4. `DPL-04` standalone Docker/Next.js multi-instance contract。
5. `DPL-05` production Terraform source/static checks；no apply。
6. `DPL-06` Vercel metadata/env-name/data inventory。
7. `DPL-07` Neon auto-DDL -> versioned RDS migration/repositories；no production migration。
8. `DPL-08` nonprod exact plans and separate provision approvals。
9. `DPL-09` production shadow/data rehearsal/evidence。
10. `DPL-10` first R2 release、AWS deploy、DNS cutover、Vercel retirement as separate actions。

Tickets需映射Phase 3 `P3-07`、`P3-07A` 及 `P3-08`–`P3-18`；不得跳過既有release authority。

## 15. Official references

- [AWS root-user best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/root-user-best-practices.html)
- [AWS Organizations best practices](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_best-practices.html)
- [AWS IAM best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [ACM DNS validation](https://docs.aws.amazon.com/acm/latest/userguide/dns-validation.html)
- [ECS deployment circuit breaker](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-circuit-breaker.html)
- [Cognito custom domains](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-add-custom-domain.html)
- [Cloudflare accounts/zones](https://developers.cloudflare.com/fundamentals/concepts/accounts-and-zones/)
- [Cloudflare account-owned tokens](https://developers.cloudflare.com/fundamentals/api/get-started/account-owned-tokens/)
- [R2 API tokens](https://developers.cloudflare.com/r2/api/tokens/)
- [R2 custom domains](https://developers.cloudflare.com/r2/buckets/public-buckets/)
- [R2 cache](https://developers.cloudflare.com/cache/interaction-cloudflare-products/r2/)
- [R2 CORS](https://developers.cloudflare.com/r2/buckets/cors/)
- [Next.js self-hosting](https://nextjs.org/docs/app/guides/self-hosting)
- [Vercel environment CLI](https://vercel.com/docs/cli/env)
- [Vercel project inventory](https://vercel.com/docs/projects/transferring-projects)

Vendor feature、region、plan、price和contract會變動。External action前需重查官方頁面、取得current quote/DPA、核對exact resource region和輸出versioned evidence。
